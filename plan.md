# Reader3 Implementation Plan

## Architecture Overview

### Architectural Style
**Two-Process Monolithic Architecture**

The system is intentionally split into two separate, loosely-coupled processes:

1. **Offline Batch Processor** (`reader3.py`)
   - CLI tool for one-time EPUB conversion
   - Input: EPUB file → Output: Structured data + images
   - No server component, pure data transformation

2. **Online Web Server** (`server.py`)
   - FastAPI application serving processed books
   - Input: HTTP requests → Output: HTML responses + images
   - No processing, pure data serving

**Rationale**:
- **Separation of Concerns**: Processing is slow (I/O bound), serving is fast (cache bound)
- **Resource Efficiency**: Don't keep heavy EPUB parsing libraries in memory for server
- **Failure Isolation**: Processing errors don't crash server, server downtime doesn't block processing
- **Simplicity**: Each process has exactly one job
- **Developer Experience**: Can iterate on UI (server) without re-processing books

**Alternative Considered**: Single-process monolith with `/upload` endpoint
- **Rejected because**: Would need background job queue, progress tracking, timeout handling - too complex for "90% vibe coded" philosophy

### System Context

```
┌─────────────────────────────────────────────────────────────────┐
│                         User's Machine                          │
│                                                                 │
│  ┌──────────┐         ┌──────────────┐      ┌──────────────┐  │
│  │   EPUB   │────────>│  reader3.py  │─────>│  book_data/  │  │
│  │  Files   │  CLI    │  (Processor) │ pkl  │  ├─book.pkl  │  │
│  └──────────┘         └──────────────┘      │  └─images/   │  │
│                                              └──────┬───────┘  │
│                                                     │           │
│  ┌──────────┐         ┌──────────────┐            │           │
│  │  Browser │<────────│  server.py   │<───────────┘           │
│  │          │  HTTP   │  (FastAPI)   │  load pkl              │
│  └──────────┘         └──────────────┘                        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**Data Flow**:
1. User downloads EPUB from Project Gutenberg (or elsewhere)
2. User runs `uv run reader3.py dracula.epub` (one-time processing)
3. System creates `dracula_data/` folder with processed data
4. User runs `uv run server.py` (web server)
5. User visits `http://127.0.0.1:8123` in browser
6. Server scans for `*_data` folders, loads books from pickle
7. User clicks "Read Book", server serves chapters from cached Book object

## Layer Structure

### Layer 1: Data Model Layer
**Responsibility**: Define domain objects and their relationships

**Components**:
- `ChapterContent` dataclass (reader3.py:19-30)
- `TOCEntry` dataclass (reader3.py:33-40)
- `BookMetadata` dataclass (reader3.py:43-53)
- `Book` dataclass (reader3.py:56-67)

**Dependencies**: None (pure data structures)

**Design Decisions**:
- **Dataclasses over dicts**: Type safety, IDE autocomplete, clear schema
- **Immutable by default**: No setters, data set at construction
- **Nested structures**: TOCEntry has recursive `children` field for tree representation
- **Separation of concerns**: `ChapterContent` (physical files) vs `TOCEntry` (logical navigation)

**Key Relationships**:
```
Book
├── metadata: BookMetadata
├── spine: List[ChapterContent]  (linear reading order)
├── toc: List[TOCEntry]          (hierarchical navigation)
└── images: Dict[str, str]       (path mapping)
```

### Layer 2: Processing Layer
**Responsibility**: Transform EPUB files into structured Book objects

**Components**:
- `process_epub()` (reader3.py:175-283) - Main orchestration
- `clean_html_content()` (reader3.py:72-86) - HTML sanitization
- `extract_plain_text()` (reader3.py:89-93) - Text extraction
- `parse_toc_recursive()` (reader3.py:96-132) - TOC parsing
- `get_fallback_toc()` (reader3.py:135-146) - Fallback navigation
- `extract_metadata_robust()` (reader3.py:149-170) - Metadata extraction

**Dependencies**:
- Upstream: `ebooklib`, `BeautifulSoup`
- Downstream: Data Model Layer

**Processing Pipeline**:
```
1. Load EPUB          → ebooklib.epub.read_epub()
2. Extract Metadata   → extract_metadata_robust()
3. Create Output Dir  → os.makedirs()
4. Extract Images     → for item in book.get_items() if ITEM_IMAGE
5. Parse TOC          → parse_toc_recursive(book.toc)
6. Process Spine      → for item in book.spine:
   ├── Decode HTML
   ├── Fix image paths
   ├── Clean HTML
   ├── Extract text
   └── Create ChapterContent
7. Assemble Book      → Book(metadata, spine, toc, images, ...)
```

**Key Algorithms**:

**A. Recursive TOC Parsing** (reader3.py:96-132)
```python
def parse_toc_recursive(toc_list, depth=0) -> List[TOCEntry]:
    # Handle three types from ebooklib:
    # 1. tuple (Section, [Children]) → recurse into children
    # 2. epub.Link → leaf node
    # 3. epub.Section → leaf node without children
    # Returns flat list with nested children in .children field
```
**Complexity**: O(n) where n = total TOC entries
**Edge Cases**: Empty TOC → fallback to spine filenames

**B. Image Path Rewriting** (reader3.py:237-249)
```python
# Problem: EPUB internal paths like "images/pic%201.jpg"
#          must map to "images/pic1.jpg" on disk
# Solution: Build dual-key map (full path + basename)
for img in soup.find_all('img'):
    src_decoded = unquote(img.get('src'))
    filename = os.path.basename(src_decoded)
    # Try full path first, then basename
    if src_decoded in image_map:
        img['src'] = image_map[src_decoded]
    elif filename in image_map:
        img['src'] = image_map[filename]
```
**Complexity**: O(m) where m = images in chapter
**Edge Cases**: Missing images → leave src unchanged (broken image in browser)

**C. HTML Cleaning** (reader3.py:72-86)
```python
# Security + Usability: Remove dangerous/useless tags
tags_to_remove = ['script', 'style', 'iframe', 'video', 'nav', 'form', 'button', 'input']
for tag in soup(tags_to_remove):
    tag.decompose()  # Remove from tree
# Also remove HTML comments
```
**Rationale**: Prevent XSS, remove navigation UI from original EPUB

### Layer 3: Persistence Layer
**Responsibility**: Serialize/deserialize Book objects

**Components**:
- `save_to_pickle()` (reader3.py:286-290) - Write to disk
- `load_book_cached()` (server.py:19-35) - Read from disk with cache

**Dependencies**:
- Upstream: `pickle` stdlib
- Downstream: Data Model Layer, Processing Layer

**Storage Format**:
```
book_data/
├── book.pkl          # Pickled Book object (~50KB-5MB)
└── images/
    ├── cover.jpg
    ├── diagram1.png
    └── ...
```

**Caching Strategy** (server.py:19):
```python
@lru_cache(maxsize=10)
def load_book_cached(folder_name: str) -> Optional[Book]:
    # LRU evicts least-recently-used book when cache full
    # Typical usage: 1 book at a time, so cache hit rate ~99%
```

**Cache Invalidation**: None - books are immutable after processing. If EPUB re-processed, server must restart to clear cache.

**Alternative Considered**: JSON serialization
- **Rejected because**: BeautifulSoup objects don't serialize to JSON cleanly
- **Tradeoff**: Pickle is Python-only, has security risks, but is trivial to implement

### Layer 4: Web Layer
**Responsibility**: HTTP request handling and routing

**Components**:
- FastAPI app (server.py:13)
- Route handlers (server.py:37-105)
  - `GET /` - Library view
  - `GET /read/{book_id}` - Redirect to first chapter
  - `GET /read/{book_id}/{chapter_index}` - Chapter reader
  - `GET /read/{book_id}/images/{image_name}` - Image serving
- Jinja2 template engine (server.py:14)

**Dependencies**:
- Upstream: `FastAPI`, `Jinja2`, `uvicorn`
- Downstream: Persistence Layer, Data Model Layer

**Request Flow**:
```
1. Browser → GET /read/dracula_data/5
2. FastAPI → async read_chapter(book_id="dracula_data", chapter_index=5)
3. Load Book → load_book_cached("dracula_data")  [cache hit]
4. Validate Index → if chapter_index >= len(book.spine): 404
5. Get Chapter → current_chapter = book.spine[chapter_index]
6. Calculate Nav → prev_idx, next_idx
7. Render Template → templates.TemplateResponse("reader.html", {...})
8. Browser ← HTML with embedded chapter content
```

**Security Measures**:
```python
# Path traversal prevention (server.py:97-98)
safe_book_id = os.path.basename(book_id)      # "../../etc/passwd" → "passwd"
safe_image_name = os.path.basename(image_name)

# File existence check (server.py:102-103)
if not os.path.exists(img_path):
    raise HTTPException(status_code=404)
```

**Error Handling**:
- Book not found → 404 with message "Book not found"
- Chapter out of range → 404 with message "Chapter not found"
- Image not found → 404 with message "Image not found"
- Corrupted pickle → Logged to console, excluded from library

### Layer 5: Presentation Layer
**Responsibility**: HTML templates and client-side UI logic

**Components**:
- `templates/library.html` (42 lines)
- `templates/reader.html` (155 lines)
- Embedded CSS (no external stylesheets)
- Minimal JavaScript (TOC navigation only)

**Dependencies**:
- Upstream: Jinja2 template engine
- Downstream: None (pure HTML/CSS/JS)

**Template Architecture**:

**A. Library Template** (templates/library.html)
```jinja2
{% for book in books %}
  <div class="book-card">
    <div class="book-title">{{ book.title }}</div>
    <div class="book-meta">{{ book.author }}<br>{{ book.chapters }} sections</div>
    <a href="/read/{{ book.id }}/0" class="btn">Read Book</a>
  </div>
{% endfor %}
```
**Layout**: CSS Grid (responsive, 250px min column width)

**B. Reader Template** (templates/reader.html)
```
┌─────────────────────────────────────────────────┐
│ Sidebar (300px)       │ Main Content (flex)    │
│ ┌─────────────────┐   │ ┌──────────────────┐   │
│ │ ← Back to Lib.  │   │ │  Chapter HTML    │   │
│ │                 │   │ │                  │   │
│ │ Book Title      │   │ │  [Content here]  │   │
│ │ ───────────────  │   │ │                  │   │
│ │ TOC:            │   │ │  Images render   │   │
│ │ • Chapter 1     │   │ │  inline          │   │
│ │   • Subsection  │   │ │                  │   │
│ │ • Chapter 2 ←─  │   │ │                  │   │
│ │ • Chapter 3     │   │ │                  │   │
│ └─────────────────┘   │ │                  │   │
│                       │ ├──────────────────┤   │
│                       │ │ [← Prev] [Next →]│   │
│                       │ └──────────────────┘   │
└─────────────────────────────────────────────────┘
```

**Recursive TOC Rendering** (templates/reader.html:49-92):
```jinja2
{% macro render_toc(items) %}
  <ul class="toc-list">
  {% for item in items %}
    <li class="toc-item">
      <a href="#" onclick="findAndGo('{{ item.file_href }}')"
         class="toc-link {% if current_chapter.href == item.file_href %}active{% endif %}">
        {{ item.title }}
      </a>
      {% if item.children %}
        {{ render_toc(item.children) }}  {# Recursive call #}
      {% endif %}
    </li>
  {% endfor %}
  </ul>
{% endmacro %}
```

**JavaScript TOC Navigation** (templates/reader.html:124-151):
```javascript
// Server passes spine map to client
const spineMap = {
  "text/chapter1.html": 0,
  "text/chapter2.html": 1,
  ...
};

function findAndGo(filename) {
  const cleanFile = filename.split('#')[0];  // Remove anchor
  const idx = spineMap[cleanFile];
  if (idx !== undefined) {
    window.location.href = "/read/{{ book_id }}/" + idx;
  }
}
```
**Rationale**: Server uses integer indices for chapters, but TOC uses filenames. JavaScript bridges the gap.

**Design System**:
- **Typography**:
  - Body: Georgia serif, 1.15em, 1.8 line-height (readability)
  - UI: -apple-system, BlinkMacSystemFont (system fonts)
- **Colors**:
  - Text: #212529 (near-black)
  - Background: #ffffff (pure white)
  - Accents: #3498db (blue for links)
  - Active: #d63384 (pink for current TOC item)
- **Spacing**: 60px vertical padding, 40px horizontal (generous whitespace)
- **Responsiveness**: Max-width 700px for content (optimal line length)

## Design Patterns Applied

### Pattern 1: Data Transfer Object (DTO)
**Location**: All dataclasses (reader3.py:19-67)

**Purpose**: Transfer data between processing and serving layers without behavior

**Implementation**:
```python
@dataclass
class ChapterContent:
    id: str
    href: str
    title: str
    content: str  # HTML
    text: str     # Plain text
    order: int    # Reading position
```

**Benefits**:
- Type-safe data contracts
- No hidden behavior (pure data)
- Serializable (pickle-friendly)

### Pattern 2: Facade
**Location**: `Book` class (reader3.py:56-67)

**Purpose**: Simplify complex EPUB structure into single object

**Implementation**:
```python
@dataclass
class Book:
    metadata: BookMetadata    # Who, what, when
    spine: List[ChapterContent]  # Linear content
    toc: List[TOCEntry]       # Hierarchical navigation
    images: Dict[str, str]    # Path mapping
    # Hides complexity of EPUB internals
```

**Benefits**:
- Server doesn't need to understand EPUB format
- Single object encapsulates entire book
- Easy to cache, serialize, pass around

### Pattern 3: Repository Pattern (Simplified)
**Location**: `load_book_cached()` (server.py:19-35)

**Purpose**: Abstract data retrieval logic

**Implementation**:
```python
@lru_cache(maxsize=10)
def load_book_cached(folder_name: str) -> Optional[Book]:
    file_path = os.path.join(BOOKS_DIR, folder_name, "book.pkl")
    if not os.path.exists(file_path):
        return None
    with open(file_path, "rb") as f:
        return pickle.load(f)
```

**Benefits**:
- Routes don't know about pickle details
- Easy to swap persistence (pickle → JSON → SQLite)
- Caching transparently added with decorator

### Pattern 4: Template Method
**Location**: `parse_toc_recursive()` (reader3.py:96-132)

**Purpose**: Handle multiple TOC item types with same algorithm

**Implementation**:
```python
def parse_toc_recursive(toc_list, depth=0):
    for item in toc_list:
        if isinstance(item, tuple):
            # Section with children → recurse
            section, children = item
            entry = create_entry(section)
            entry.children = parse_toc_recursive(children, depth+1)
        elif isinstance(item, epub.Link):
            # Leaf node
            entry = create_entry(item)
        elif isinstance(item, epub.Section):
            # Another leaf variant
            entry = create_entry(item)
```

**Benefits**:
- Single function handles tree traversal
- Type-specific logic isolated in branches
- Easy to add new TOC item types

### Pattern 5: Cache-Aside
**Location**: `@lru_cache` decorator (server.py:19)

**Purpose**: Load from cache if present, otherwise load from disk and cache

**Implementation**:
```python
@lru_cache(maxsize=10)
def load_book_cached(folder_name: str) -> Optional[Book]:
    # On cache miss: load from disk and cache
    # On cache hit: return cached value
    # On cache full: evict LRU item
```

**Benefits**:
- 50-200ms pickle load → <1ms cache hit
- Automatic eviction (no manual cache management)
- Thread-safe (important for async FastAPI)

### Pattern 6: Static Resource Serving
**Location**: Image route (server.py:89-105)

**Purpose**: Serve static files with security checks

**Implementation**:
```python
@app.get("/read/{book_id}/images/{image_name}")
async def serve_image(book_id: str, image_name: str):
    safe_book_id = os.path.basename(book_id)      # Security
    safe_image_name = os.path.basename(image_name)
    img_path = os.path.join(BOOKS_DIR, safe_book_id, "images", safe_image_name)
    if not os.path.exists(img_path):
        raise HTTPException(status_code=404)
    return FileResponse(img_path)  # Streaming response
```

**Benefits**:
- No need for static file middleware (direct serving)
- Path traversal prevention (basename)
- Proper MIME type detection (automatic)

## Technology Stack

### Core Languages
- **Python 3.10+**
  - Rationale: Modern type hints, dataclasses, match statements
  - Alternatives considered: Python 3.8 (rejected - missing some type hint features)

### Backend Framework
- **FastAPI 0.121.2**
  - Rationale: Fast, async, automatic OpenAPI docs, modern Python
  - Alternatives considered:
    - Flask (rejected - synchronous, manual validation)
    - Django (rejected - too heavy for simple app)

### EPUB Parsing
- **ebooklib 0.20**
  - Rationale: Pure Python, handles EPUB2/3, active maintenance
  - Alternatives considered:
    - epub (rejected - outdated, Python 2)
    - Calibre (rejected - too complex, GPL license)

### HTML Processing
- **BeautifulSoup4 4.14.2**
  - Rationale: Robust HTML parsing, tree manipulation, handles malformed HTML
  - Alternatives considered:
    - lxml (rejected - more complex, C dependency)
    - html.parser (rejected - less robust)

### Template Engine
- **Jinja2 3.1.6**
  - Rationale: Comes with FastAPI, powerful, well-documented
  - Alternatives considered:
    - Mako (rejected - less popular)
    - String templates (rejected - no escaping, no logic)

### ASGI Server
- **Uvicorn 0.38.0**
  - Rationale: Fast, async, recommended by FastAPI
  - Alternatives considered:
    - Hypercorn (rejected - less mature)
    - Gunicorn (rejected - WSGI, not ASGI)

### Package Management
- **uv**
  - Rationale: Fast, modern, lockfile support
  - Alternatives considered:
    - pip + venv (rejected - slower, no lockfile)
    - poetry (rejected - heavier)
    - pipenv (rejected - slower than uv)

### Why No Database?
**Decision**: File-based storage (pickle + filesystem)

**Rationale**:
- Book data is immutable (never updated after processing)
- Small dataset (10-100 books typical, <1GB)
- Single-user (no concurrent writes)
- Simplicity (no migration scripts, no ORM, no connection pools)

**Tradeoffs**:
- (+) Zero configuration
- (+) Easy backup (just copy folder)
- (+) No query language to learn
- (-) No search without loading all books
- (-) No relationships (joins)
- (-) Pickle security risk

**When to migrate to database**:
- Multi-user support needed → need sessions, user tables
- Search needed → need full-text index (SQLite FTS5)
- >1000 books → need pagination, queries
- Analytics needed → need aggregations

## Data Flow

### Processing Flow (Offline)
```
┌──────────────┐
│ User runs    │
│ reader3.py   │
│ dracula.epub │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 1. Load EPUB (ebooklib)              │
│    epub.read_epub("dracula.epub")    │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 2. Extract Metadata                  │
│    title, authors, language, etc.    │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 3. Create Output Directory           │
│    dracula_data/                     │
│    dracula_data/images/              │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 4. Extract Images                    │
│    For each ITEM_IMAGE:              │
│    - Sanitize filename               │
│    - Write to images/                │
│    - Map: internal_path → local_path │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 5. Parse Table of Contents           │
│    Recursive: epub.toc → TOCEntry[]  │
│    Fallback: spine → TOCEntry[]      │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 6. Process Spine (Linear Content)    │
│    For each spine item:              │
│    ┌────────────────────────────┐    │
│    │ a. Decode HTML (UTF-8)     │    │
│    │ b. Parse with BeautifulSoup│    │
│    │ c. Rewrite image paths     │    │
│    │ d. Clean HTML (remove tags)│    │
│    │ e. Extract body content    │    │
│    │ f. Extract plain text      │    │
│    │ g. Create ChapterContent   │    │
│    └────────────────────────────┘    │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 7. Assemble Book Object              │
│    Book(                             │
│      metadata=BookMetadata(...),     │
│      spine=ChapterContent[],         │
│      toc=TOCEntry[],                 │
│      images=Dict[str, str]           │
│    )                                 │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ 8. Pickle to Disk                    │
│    dracula_data/book.pkl             │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│ Processing   │
│ Complete     │
└──────────────┘
```

### Serving Flow (Online)
```
┌──────────────┐
│ User visits  │
│ localhost:   │
│ 8123/        │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│ GET / (Library View)                 │
│                                      │
│ 1. Scan BOOKS_DIR for *_data folders│
│ 2. For each folder:                  │
│    - load_book_cached(folder)        │
│    - Extract: title, author, count   │
│ 3. Render library.html               │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│ User clicks  │
│ "Read Book"  │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│ GET /read/{book_id}/0                │
│                                      │
│ 1. load_book_cached(book_id)         │
│    ├─ Cache hit? Return cached       │
│    └─ Cache miss? Load + cache       │
│ 2. Get chapter: book.spine[0]        │
│ 3. Calculate: prev_idx, next_idx     │
│ 4. Render reader.html:               │
│    - book (metadata + toc)           │
│    - current_chapter (content)       │
│    - navigation links                │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ Browser renders:                     │
│ - Sidebar with TOC                   │
│ - Chapter HTML content               │
│ - Images: <img src="images/pic.jpg"> │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────────────────────────────┐
│ Browser requests image:              │
│ GET /read/{book_id}/images/pic.jpg   │
│                                      │
│ 1. Sanitize: book_id, image_name     │
│ 2. Build path: BOOKS_DIR/book_id/    │
│    images/image_name                 │
│ 3. Check existence                   │
│ 4. Return FileResponse (streaming)   │
└──────┬───────────────────────────────┘
       │
       ▼
┌──────────────┐
│ Image        │
│ displayed    │
└──────────────┘

┌──────────────┐
│ User clicks  │
│ "Next"       │
└──────┬───────┘
       │
       ▼
┌──────────────────────────────────────┐
│ GET /read/{book_id}/1                │
│ (Same flow as chapter 0)             │
└──────────────────────────────────────┘
```

### TOC Navigation Flow
```
┌──────────────────┐
│ User clicks TOC  │
│ item: "Chapter 2"│
└──────┬───────────┘
       │
       ▼
┌─────────────────────────────────────────┐
│ JavaScript: findAndGo("text/ch2.html")  │
│                                         │
│ 1. Strip anchor: "text/ch2.html"        │
│ 2. Lookup in spineMap:                  │
│    spineMap["text/ch2.html"] → 5        │
│ 3. Navigate: window.location.href =    │
│    "/read/dracula_data/5"               │
└─────┬───────────────────────────────────┘
      │
      ▼
┌──────────────────────────────────────┐
│ GET /read/dracula_data/5             │
│ (Server flow as before)              │
└──────────────────────────────────────┘
```

## Regeneration Strategy

### Phase 1: Specification-First Rebuild
If building from scratch using this spec:

1. **Start with spec.md** (this document's companion)
   - Read functional requirements → know what to build
   - Read constraints → know what NOT to build
   - Read success criteria → know when done

2. **Apply extracted skills** (from intelligence-object.md)
   - HTML sanitization pattern
   - Path security pattern
   - Recursive tree parsing pattern
   - Cache-aside pattern

3. **Implement with modern best practices**
   - Fix security gaps (SQLite instead of pickle)
   - Add tests (pytest for processing, httpx for API)
   - Add configuration (environment variables)
   - Add logging (structured JSON logs)

### Phase 2: Test-First Development
Rebuild strategy using TDD:

1. **Write acceptance tests from spec success criteria**
   ```python
   def test_process_epub():
       # Given: sample EPUB file
       # When: process_epub("dracula.epub", "output/")
       # Then: book.pkl exists, images extracted, TOC parsed
   ```

2. **Implement features to pass tests**
   - Start with data models (simplest)
   - Then processing (complex logic)
   - Then serving (API layer)
   - Finally templates (presentation)

3. **Refactor with confidence**
   - Tests protect behavior during refactoring
   - Can safely swap pickle → SQLite
   - Can safely optimize HTML cleaning

### Phase 3: Incremental Migration (If Replacing Existing)
If replacing legacy reader system:

1. **Strangler Pattern**
   - New implementation runs alongside old
   - Feature flag: `USE_NEW_READER=true`
   - Gradually shift traffic: 10% → 50% → 100%

2. **Feature Flags**
   ```python
   if os.getenv("USE_NEW_READER") == "true":
       return new_read_chapter(book_id, chapter_index)
   else:
       return legacy_read_chapter(book_id, chapter_index)
   ```

3. **Parallel Run**
   - Process same EPUB with both systems
   - Compare outputs (diff HTML, check image paths)
   - Validate equivalence before cutover

4. **Cutover**
   - Remove feature flags
   - Delete old code
   - Update documentation

## Improvement Opportunities

### Technical Improvements

#### 1. Replace Pickle with SQLite
**Rationale**: Security, queryability, portability

**Migration**:
```sql
CREATE TABLE books (
    id TEXT PRIMARY KEY,
    title TEXT,
    authors TEXT,  -- JSON array
    language TEXT,
    processed_at TEXT
);

CREATE TABLE chapters (
    id INTEGER PRIMARY KEY,
    book_id TEXT,
    href TEXT,
    title TEXT,
    content TEXT,  -- HTML
    text TEXT,     -- Plain text
    order_index INTEGER,
    FOREIGN KEY (book_id) REFERENCES books(id)
);

CREATE VIRTUAL TABLE chapters_fts USING fts5(text, content='chapters', content_rowid='id');
```

**Benefits**:
- (+) No pickle security risk
- (+) Full-text search via FTS5
- (+) Query by author, language, etc.
- (+) Portable (SQLite is cross-platform)
- (-) More complex than pickle.dump()

#### 2. Add Search Functionality
**Current**: Text extracted but not indexed

**Implementation**:
```python
@app.get("/search")
async def search(q: str, book_id: Optional[str] = None):
    # Search across all chapters or within book
    results = db.execute("""
        SELECT chapters.*, books.title, snippet(chapters_fts, 0, '<mark>', '</mark>', '...', 32)
        FROM chapters_fts
        JOIN chapters ON chapters_fts.rowid = chapters.id
        JOIN books ON chapters.book_id = books.id
        WHERE chapters_fts MATCH ? AND (? IS NULL OR book_id = ?)
        LIMIT 50
    """, [q, book_id, book_id])
    return {"results": results}
```

**UI**: Search bar in library, search box in reader sidebar

#### 3. Refactor HTML Cleaning using Bleach
**Current**: Manual tag removal (reader3.py:75)

**Improved**:
```python
import bleach

ALLOWED_TAGS = ['p', 'br', 'strong', 'em', 'h1', 'h2', 'h3', 'h4', 'h5', 'h6',
                'ul', 'ol', 'li', 'a', 'img', 'blockquote', 'code', 'pre']
ALLOWED_ATTRS = {'a': ['href'], 'img': ['src', 'alt']}

def clean_html_content(soup: BeautifulSoup) -> str:
    html = str(soup)
    return bleach.clean(html, tags=ALLOWED_TAGS, attributes=ALLOWED_ATTRS, strip=True)
```

**Benefits**: Industry-standard sanitization, XSS prevention

### Architectural Improvements

#### 1. Introduce Configuration System
**Current**: Hardcoded values (server.py:17, 110)

**Improved**:
```python
# config.py
from pydantic import BaseSettings

class Settings(BaseSettings):
    books_dir: str = "."
    host: str = "127.0.0.1"
    port: int = 8123
    cache_size: int = 10

    class Config:
        env_file = ".env"

settings = Settings()
```

**Usage**:
```bash
# .env file
BOOKS_DIR=/home/user/library
HOST=0.0.0.0
PORT=8080
CACHE_SIZE=20
```

#### 2. Add Reading Progress Tracking
**Implementation**:
```python
# Store in browser localStorage
localStorage.setItem('reader3_progress', JSON.stringify({
  'dracula_data': { lastChapter: 5, scrollPosition: 234 },
  'frankenstein_data': { lastChapter: 2, scrollPosition: 0 }
}));

// On library page, show "Continue Reading" button
```

**Alternative**: Server-side with SQLite
```sql
CREATE TABLE reading_progress (
    book_id TEXT,
    user_id TEXT,  -- For future multi-user
    last_chapter INTEGER,
    last_updated TEXT,
    PRIMARY KEY (book_id, user_id)
);
```

#### 3. Add EPUB Validation
**Current**: Crashes on malformed EPUBs

**Improved**:
```python
def validate_epub(epub_path: str) -> Tuple[bool, Optional[str]]:
    try:
        book = epub.read_epub(epub_path)

        # Check required metadata
        if not book.get_metadata('DC', 'title'):
            return False, "Missing title metadata"

        # Check spine exists
        if not book.spine:
            return False, "Empty spine (no chapters)"

        return True, None
    except Exception as e:
        return False, f"EPUB parse error: {str(e)}"

# In main()
is_valid, error = validate_epub(epub_file)
if not is_valid:
    print(f"ERROR: {error}")
    sys.exit(1)
```

### Operational Improvements

#### 1. CI/CD Pipeline
```yaml
# .github/workflows/test.yml
name: Test
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - run: pip install uv
      - run: uv sync
      - run: uv run pytest tests/ --cov=. --cov-report=xml
      - uses: codecov/codecov-action@v3
```

#### 2. Docker Deployment
```dockerfile
FROM python:3.10-slim
WORKDIR /app
COPY pyproject.toml uv.lock ./
RUN pip install uv && uv sync --frozen
COPY . .
EXPOSE 8123
CMD ["uv", "run", "server.py"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  reader3:
    build: .
    ports:
      - "8123:8123"
    volumes:
      - ./library:/app/library
    environment:
      - BOOKS_DIR=/app/library
```

#### 3. Monitoring with Prometheus
```python
from prometheus_fastapi_instrumentator import Instrumentator

app = FastAPI()
Instrumentator().instrument(app).expose(app)

# Exposes /metrics endpoint
# Tracks: request count, latency, error rate
```

#### 4. Structured Logging
```python
import structlog

logger = structlog.get_logger()

@app.get("/read/{book_id}/{chapter_index}")
async def read_chapter(...):
    logger.info("chapter_viewed",
                book_id=book_id,
                chapter_index=chapter_index,
                user_agent=request.headers.get("user-agent"))
```

## Migration from v2.x to v3.0 (Inferred)

### What Changed in v3.0

1. **Spine-based processing** (vs TOC-based?)
   - v2.x likely processed based on TOC order
   - v3.0 uses EPUB spine (guarantees linear order)

2. **Recursive TOC parsing**
   - v2.x likely flat TOC
   - v3.0 preserves hierarchical structure

3. **LRU cache added**
   - v2.x loaded pickle on every request
   - v3.0 caches books in memory

4. **JavaScript TOC navigation**
   - v2.x likely server-side TOC links
   - v3.0 client-side mapping (more responsive)

### Migration Path

If users have v2.x processed books:

1. **Re-process all books** (safest)
   ```bash
   for epub in *.epub; do
       uv run reader3.py "$epub"
   done
   ```

2. **Version detection** (if needed)
   ```python
   book = pickle.load(f)
   if not hasattr(book, 'version') or book.version < "3.0":
       raise ValueError("Book processed with old version, please re-process")
   ```

## Conclusion

This implementation plan provides a complete blueprint for:
- **Regenerating** the system from scratch
- **Understanding** the architectural decisions and tradeoffs
- **Improving** the system with modern best practices
- **Migrating** to production-grade infrastructure when needed

The core philosophy remains: **Simplicity first, complexity only when necessary**. The current implementation (~425 lines) achieves 90% of use cases. The improvements above are for the remaining 10% when simple becomes insufficient.
