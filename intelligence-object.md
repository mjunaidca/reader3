# Reader3 Reusable Intelligence

## Purpose

This document extracts reusable intelligence from the Reader3 codebase - patterns, decisions, and mental models that can be applied to other projects. Think of this as "compressed knowledge" that transcends the specific implementation.

**What makes these patterns reusable:**
- They solve common problems in web apps, content processing, and UI design
- They represent thoughtful tradeoffs between complexity and simplicity
- They embody tacit knowledge that isn't always written down
- They can be encoded as "skills" for LLM-assisted development

---

## Extracted Skills

### Skill 1: HTML Content Sanitization for User-Generated Content

**Persona**: You are a security-conscious backend engineer processing untrusted HTML content.

**Questions to ask before implementing**:
1. What HTML tags are semantically necessary for content? (p, h1-h6, img, a, etc.)
2. What tags introduce security risks? (script, iframe, form, object)
3. What tags are presentation-only and should be removed? (style, font, center)
4. Do I need to preserve HTML structure or just text?
5. What attributes are safe? (href on links, src on images)
6. Should I whitelist (safer) or blacklist (easier but riskier)?

**Principles**:
1. **Whitelist over blacklist**: Explicitly allow safe tags, block everything else
2. **Defense in depth**: Remove dangerous tags AND sanitize attributes
3. **Extract text separately**: Always maintain a plain-text version
4. **Comments are code**: HTML comments can contain sensitive info, remove them
5. **Test with real-world data**: Use actual user-generated content, not synthetic

**Implementation Pattern** (from reader3.py:72-93):
```python
from bs4 import BeautifulSoup, Comment

def clean_html_content(soup: BeautifulSoup) -> BeautifulSoup:
    """
    Two-phase sanitization:
    1. Remove dangerous/useless tags (security + UX)
    2. Remove comments (security)
    """

    # Phase 1: Remove dangerous tags (can execute code)
    dangerous_tags = ['script', 'style', 'iframe', 'video', 'form', 'button', 'input']
    for tag in soup(dangerous_tags):
        tag.decompose()  # Remove from tree entirely

    # Phase 2: Remove HTML comments (can leak info)
    for comment in soup.find_all(string=lambda text: isinstance(text, Comment)):
        comment.extract()

    return soup

def extract_plain_text(soup: BeautifulSoup) -> str:
    """
    Always maintain a text-only version:
    - For search indexing
    - For LLM context
    - For accessibility
    """
    text = soup.get_text(separator=' ')
    return ' '.join(text.split())  # Collapse whitespace
```

**When to apply**:
- Processing EPUB, DOCX, HTML emails
- User-submitted markdown (after converting to HTML)
- Web scraping (untrusted sources)
- CMS content (even if trusted, defense in depth)

**Alternatives considered**:
- **bleach library** (Pro: Industry standard, Con: Another dependency)
- **html.parser** (Pro: Stdlib, Con: Less robust than BeautifulSoup)
- **Regex** (Pro: Fast, Con: HTML is not regular, dangerous)

**Gotchas**:
- Don't forget to remove comments (often overlooked)
- `decompose()` vs `extract()`: decompose fully removes, extract just detaches
- Some EPUBs use `<nav>` for navigation - decide if you want to keep it

---

### Skill 2: Recursive Tree Parsing with Type Polymorphism

**Persona**: You are a data engineer parsing hierarchical structures with inconsistent types.

**Questions to ask before implementing**:
1. What types can appear in the tree? (tuples, objects, different classes)
2. How do I identify leaf nodes vs parent nodes?
3. Is the tree homogeneous (same type throughout) or heterogeneous?
4. What's the maximum depth? (affects recursion limit)
5. Should I use recursion or iterative traversal? (recursion is cleaner, iteration is safer)
6. How do I handle malformed trees? (missing children, cycles)

**Principles**:
1. **Type dispatch**: Use `isinstance()` to handle different node types
2. **Preserve structure**: Don't flatten unless you have to
3. **Depth parameter**: Track depth for debugging and limits
4. **Fallback strategy**: Have a plan B if tree is empty/malformed
5. **Separate concerns**: Tree traversal separate from node processing

**Implementation Pattern** (from reader3.py:96-132):
```python
from ebooklib import epub
from typing import List, Union

def parse_toc_recursive(toc_list, depth=0) -> List[TOCEntry]:
    """
    Handles three types from ebooklib:
    1. tuple (Section, [children]) - Parent with children
    2. epub.Link - Leaf node (link)
    3. epub.Section - Leaf node (section without children)

    Why three types? Library evolved over time, maintains backward compat.
    """
    result = []

    for item in toc_list:
        # Type 1: Tuple (Section, Children) - Recursive case
        if isinstance(item, tuple):
            section, children = item
            entry = create_toc_entry(section)  # Extract common logic
            entry.children = parse_toc_recursive(children, depth + 1)  # Recurse
            result.append(entry)

        # Type 2: epub.Link - Leaf node
        elif isinstance(item, epub.Link):
            entry = create_toc_entry(item)
            result.append(entry)

        # Type 3: epub.Section - Leaf node (legacy)
        elif isinstance(item, epub.Section):
            entry = create_toc_entry(item)
            result.append(entry)

        # Unknown type: log and skip (defensive programming)
        else:
            print(f"Warning: Unknown TOC item type: {type(item)}")

    return result

def create_toc_entry(item) -> TOCEntry:
    """Extract common logic for creating entries."""
    # Split href into filename and anchor
    parts = item.href.split('#', 1)
    file_href = parts[0]
    anchor = parts[1] if len(parts) > 1 else ""

    return TOCEntry(
        title=item.title,
        href=item.href,
        file_href=file_href,
        anchor=anchor
    )
```

**Fallback strategy** (from reader3.py:135-146):
```python
def get_fallback_toc(book_obj) -> List[TOCEntry]:
    """
    When TOC is missing/empty, generate from spine (linear reading order).
    Always have a Plan B for malformed data.
    """
    toc = []
    for item in book_obj.get_items():
        if item.get_type() == ebooklib.ITEM_DOCUMENT:
            # Generate title from filename (best guess)
            title = (item.get_name()
                     .replace('.html', '')
                     .replace('.xhtml', '')
                     .replace('_', ' ')
                     .title())
            toc.append(TOCEntry(title=title, href=item.get_name(),
                               file_href=item.get_name(), anchor=""))
    return toc
```

**When to apply**:
- Parsing JSON/XML with nested structures (sitemap, org chart)
- AST traversal (code parsing)
- File system traversal (directory trees)
- Comment threads (Reddit-style nested comments)
- Menu hierarchies (navigation, dropdowns)

**Gotchas**:
- Python recursion limit (default 1000) - use `sys.setrecursionlimit()` if needed
- Cycles in tree (circular references) - track visited nodes
- Deep trees (>100 levels) - consider iterative approach with stack

---

### Skill 3: Path Security Pattern (Preventing Directory Traversal)

**Persona**: You are a web security engineer serving user-specified files.

**Questions to ask before implementing**:
1. Can users control any part of the file path? (URL param, query string, header)
2. What's the allowed directory scope? (e.g., only `/var/books/`)
3. Are symbolic links a risk? (can point outside allowed directory)
4. Should I validate file extensions? (only serve .jpg, .png, etc.)
5. What should happen for missing files? (404 vs 403 vs 500)

**Principles**:
1. **Never trust user input**: Always sanitize paths
2. **Use basename for user input**: Strips directory components (../../etc/passwd → passwd)
3. **Whitelist base directory**: All files must be under a known root
4. **Check file existence**: Fail gracefully for missing files
5. **No information leakage**: Don't reveal directory structure in errors

**Implementation Pattern** (from server.py:89-105):
```python
import os
from fastapi import HTTPException
from fastapi.responses import FileResponse

BOOKS_DIR = "."  # Allowed base directory

@app.get("/read/{book_id}/images/{image_name}")
async def serve_image(book_id: str, image_name: str):
    """
    Secure file serving:
    1. Sanitize all user inputs
    2. Build path from trusted base
    3. Verify existence
    4. Return file or 404 (no info leak)
    """

    # Step 1: Sanitize user inputs (CRITICAL)
    safe_book_id = os.path.basename(book_id)      # "../../etc" → "etc"
    safe_image_name = os.path.basename(image_name)  # "../passwd" → "passwd"

    # Step 2: Build path from TRUSTED base directory
    img_path = os.path.join(BOOKS_DIR, safe_book_id, "images", safe_image_name)

    # Step 3: Verify file exists (don't serve directories!)
    if not os.path.exists(img_path):
        raise HTTPException(status_code=404, detail="Image not found")

    # Optional: Verify it's actually a file (not directory)
    if not os.path.isfile(img_path):
        raise HTTPException(status_code=404, detail="Image not found")

    # Step 4: Serve file (FileResponse handles MIME type, streaming)
    return FileResponse(img_path)
```

**Additional security** (if needed):
```python
import pathlib

def is_safe_path(base_dir: str, user_path: str) -> bool:
    """
    More robust check: verify resolved path is under base_dir.
    Handles symlinks and relative paths.
    """
    base = pathlib.Path(base_dir).resolve()
    target = (pathlib.Path(base_dir) / user_path).resolve()
    return target.is_relative_to(base)  # Python 3.9+

# Usage
if not is_safe_path(BOOKS_DIR, f"{book_id}/images/{image_name}"):
    raise HTTPException(status_code=403, detail="Access denied")
```

**When to apply**:
- Serving user uploads (avatars, documents)
- Static file serving (images, CSS, JS)
- Download endpoints (reports, exports)
- Backup file access
- Log file viewers

**Attack examples prevented**:
```python
# Attack 1: Directory traversal
book_id = "../../etc"
image_name = "passwd"
# After basename: book_id="etc", image_name="passwd"
# Final path: ./etc/images/passwd (not /etc/passwd)

# Attack 2: Null byte injection (older systems)
image_name = "safe.jpg\x00.php"
# After basename: "safe.jpg\x00.php"
# os.path.exists() will fail (file doesn't exist)

# Attack 3: Symlink escape
# If books/evil/images/passwd -> /etc/passwd
# is_safe_path() will detect and block
```

**Gotchas**:
- Windows vs Linux path separators (`\` vs `/`)
- Case sensitivity (macOS is case-insensitive, Linux is case-sensitive)
- Unicode normalization (é vs é - different codepoints, same appearance)

---

### Skill 4: LRU Cache-Aside Pattern for Expensive Operations

**Persona**: You are a performance engineer optimizing slow data access.

**Questions to ask before implementing**:
1. What's the read-to-write ratio? (High read = good cache candidate)
2. How expensive is the operation? (>50ms = worth caching)
3. How often does data change? (Rarely = good for caching)
4. What's the working set size? (How many items accessed frequently?)
5. What's the cache invalidation strategy? (TTL, manual, never)

**Principles**:
1. **Cache-aside pattern**: Check cache first, load on miss, update cache
2. **LRU eviction**: Least Recently Used - simple and effective
3. **Measure first**: Profile before optimizing
4. **Size the cache**: Too small = thrashing, too large = memory waste
5. **Thread-safe by default**: Use built-in thread-safe implementations

**Implementation Pattern** (from server.py:19-35):
```python
from functools import lru_cache
import pickle
from typing import Optional

@lru_cache(maxsize=10)  # Keep 10 most-recently-used books in memory
def load_book_cached(folder_name: str) -> Optional[Book]:
    """
    Cache-aside pattern:
    1. Check cache (automatic with @lru_cache)
    2. On cache miss: load from disk, cache result
    3. On cache hit: return cached value (fast!)

    Why maxsize=10?
    - Typical user reads 1 book at a time
    - 10 books ≈ 50-500MB RAM (acceptable)
    - Handles "browse library, read book, browse again" workflow
    """
    file_path = os.path.join(BOOKS_DIR, folder_name, "book.pkl")

    # Early return for missing files (don't cache None!)
    if not os.path.exists(file_path):
        return None

    try:
        # Expensive operation: pickle deserialization (50-200ms)
        with open(file_path, "rb") as f:
            book = pickle.load(f)
        return book  # Cached automatically by decorator
    except Exception as e:
        # Don't cache errors (return None, not cached)
        print(f"Error loading book {folder_name}: {e}")
        return None
```

**Performance impact**:
```
First load (cache miss):  180ms  (pickle.load)
Subsequent loads (hit):   <1ms   (memory access)

Speedup: 180x faster
```

**Cache statistics** (monitoring):
```python
# lru_cache provides introspection
info = load_book_cached.cache_info()
print(f"Hits: {info.hits}, Misses: {info.misses}, Size: {info.currsize}/{info.maxsize}")
# Example output: Hits: 234, Misses: 12, Size: 8/10

# Hit rate
hit_rate = info.hits / (info.hits + info.misses)
print(f"Cache hit rate: {hit_rate:.1%}")  # 95.1%
```

**Cache invalidation** (when data changes):
```python
# Manual invalidation (if book re-processed)
load_book_cached.cache_clear()  # Clear all

# Or invalidate specific entry (Python 3.9+)
# Not directly supported by lru_cache, would need custom wrapper

# Alternative: TTL-based caching
from cachetools import TTLCache, cached

cache = TTLCache(maxsize=10, ttl=300)  # 5-minute TTL

@cached(cache)
def load_book_cached_ttl(folder_name: str) -> Optional[Book]:
    # Same implementation as above
    pass
```

**When to apply**:
- Database query results (user profiles, product data)
- API responses (external service calls)
- Computed values (analytics, aggregations)
- File parsing (CSV, JSON, XML)
- Image thumbnails (generated on-demand)

**Alternatives**:
| Pattern | Use Case | Pros | Cons |
|---------|----------|------|------|
| **LRU Cache** | Bounded working set | Simple, built-in, thread-safe | No TTL, manual invalidation |
| **TTL Cache** | Stale data acceptable | Auto-invalidation | More complex, needs cachetools |
| **Redis** | Multi-process/server | Shared cache, persistence | Infrastructure dependency |
| **CDN** | Static assets | Edge caching, global | Only for HTTP responses |

**Gotchas**:
- Don't cache None/errors (wastes cache space)
- Unhashable arguments (lists, dicts) won't work with lru_cache
- Cache size affects memory (monitor RSS)
- Thread-safety: lru_cache is thread-safe, but your function must be too

---

### Skill 5: Two-Process Architecture for Batch + Serve Workflows

**Persona**: You are a systems architect designing a data processing + serving system.

**Questions to ask before implementing**:
1. Is processing synchronous (blocking) or can it be deferred?
2. How long does processing take? (>10s = probably async)
3. Can processing failures crash the server?
4. Do users need real-time processing or batch is OK?
5. Should processing and serving share a runtime?

**Principles**:
1. **Separate concerns**: Processing is I/O-bound, serving is request-bound
2. **Fail independently**: Processing errors don't crash server
3. **Scale independently**: Add more workers OR more servers
4. **Optimize independently**: Different resource profiles (CPU vs memory)
5. **Deploy independently**: Update processor without restarting server

**Architecture Pattern** (from reader3.py + server.py):
```
┌─────────────────────────────────────────────┐
│         Process 1: Batch Processor          │
│                (reader3.py)                 │
│                                             │
│  CLI Tool:                                  │
│  - Runs on-demand (user triggers)          │
│  - Heavy I/O (EPUB parsing, image extract) │
│  - Writes to disk (pickle + images)        │
│  - No HTTP server (pure data transform)    │
│  - Can take 30-60 seconds (blocking OK)    │
│                                             │
│  Resource profile:                          │
│  - High disk I/O                            │
│  - Moderate CPU (HTML parsing)              │
│  - Moderate memory (one book at a time)    │
└──────────────┬──────────────────────────────┘
               │
               │ Writes to disk
               ▼
         ┌─────────────┐
         │  book.pkl   │
         │  images/    │
         └──────┬──────┘
                │
                │ Reads from disk
                ▼
┌─────────────────────────────────────────────┐
│          Process 2: Web Server              │
│                (server.py)                  │
│                                             │
│  FastAPI App:                               │
│  - Runs continuously (daemon)               │
│  - Light I/O (cache-bound after warmup)    │
│  - Reads from disk (pickle load)            │
│  - HTTP server (async, handles concurrent)  │
│  - Sub-100ms responses (interactive)        │
│                                             │
│  Resource profile:                          │
│  - Low disk I/O (cached)                    │
│  - Low CPU (just serving HTML)              │
│  - Memory grows with cache (10 books)      │
└─────────────────────────────────────────────┘
```

**Why separate processes?**

| Aspect | Monolith (single process) | Two-process (reader3) |
|--------|---------------------------|----------------------|
| **Complexity** | Lower (one codebase) | Higher (two processes) |
| **Dependencies** | ebooklib in server memory | ebooklib only when needed |
| **Failure isolation** | Processing crash = server down | Processing crash ≠ server down |
| **Scaling** | Scale everything together | Scale independently |
| **Development** | Restart server to test processor | Iterate on processor separately |
| **Deployment** | Single artifact | Two artifacts (but simpler) |

**When to use two-process architecture:**
- Batch processing + API serving (this case)
- Video encoding + streaming service
- Data pipeline + analytics dashboard
- Report generation + download service
- ML training + inference API

**When to use single process:**
- Processing is fast (<1s)
- Processing is rare (once per deploy)
- Need transactional consistency
- Simple CRUD (no heavy processing)

**Communication patterns:**

**Option 1: File-based (reader3 uses this)**
```python
# Processor writes
with open("data.pkl", "wb") as f:
    pickle.dump(data, f)

# Server reads
with open("data.pkl", "rb") as f:
    data = pickle.load(f)
```
- Pros: Simple, no network, persistent
- Cons: No real-time, manual coordination

**Option 2: Queue-based (for real-time)**
```python
# Processor publishes
redis_client.rpush("jobs", json.dumps(job))

# Server consumes
job = redis_client.blpop("jobs")
```
- Pros: Real-time, decoupled, durable
- Cons: Infrastructure (Redis), more complex

**Option 3: API-based (for hybrid)**
```python
# Processor calls server API
requests.post("http://server/api/books", json=book_data)

# Server receives
@app.post("/api/books")
def receive_book(book: BookData):
    save_to_db(book)
```
- Pros: Real-time, clean interface
- Cons: Server must be running, network dependency

**Deployment patterns:**

**Development:**
```bash
# Terminal 1: Process book (on-demand)
uv run reader3.py dracula.epub

# Terminal 2: Start server (continuous)
uv run server.py
```

**Production (systemd):**
```ini
# /etc/systemd/system/reader3-server.service
[Unit]
Description=Reader3 Web Server
After=network.target

[Service]
Type=simple
User=reader3
WorkingDirectory=/opt/reader3
ExecStart=/usr/bin/uv run server.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
# Processing is manual (user SSH, run command)
ssh user@server
uv run reader3.py new_book.epub
```

**Production (Docker):**
```yaml
# docker-compose.yml
services:
  # Server runs continuously
  server:
    image: reader3:latest
    command: uv run server.py
    volumes:
      - ./books:/app/books
    ports:
      - "8123:8123"
    restart: always

  # Processor runs on-demand (manual docker-compose run)
  processor:
    image: reader3:latest
    command: uv run reader3.py /books/input.epub
    volumes:
      - ./books:/app/books
    profiles: ["cli"]  # Not started automatically
```

```bash
# Run processor manually
docker-compose run processor uv run reader3.py dracula.epub
```

---

### Skill 6: Jinja2 Recursive Macros for Hierarchical Data

**Persona**: You are a frontend engineer rendering nested data structures.

**Questions to ask before implementing**:
1. What's the maximum nesting depth? (affects performance, readability)
2. Should all levels render the same? (homogeneous) or different? (heterogeneous)
3. Do I need to track depth/path for styling?
4. Should nodes be clickable/interactive?
5. Is the tree large enough to need virtual scrolling?

**Principles**:
1. **Macros for reusable templates**: Like functions for HTML
2. **Recursion for tree structures**: Macro calls itself
3. **Pass context explicitly**: Don't rely on implicit variable scope
4. **Base case**: Handle empty lists/leaf nodes
5. **Style by depth**: Use CSS descendant selectors or depth parameter

**Implementation Pattern** (from templates/reader.html:49-92):
```jinja2
{# Recursive macro for nested TOC #}
{% macro render_toc(items, depth=0) %}
    <ul class="toc-list depth-{{ depth }}">
    {% for item in items %}
        <li class="toc-item">
            {# Determine if this is the active chapter #}
            {% set is_active = current_chapter.href == item.file_href %}

            {# Render link with active state #}
            <a href="#" onclick="findAndGo('{{ item.file_href }}')"
               class="toc-link {% if is_active %}active{% endif %}">
                {{ item.title }}
            </a>

            {# Recursive call for children #}
            {% if item.children %}
                {{ render_toc(item.children, depth + 1) }}
            {% endif %}
        </li>
    {% endfor %}
    </ul>
{% endmacro %}

{# Invoke macro with root TOC items #}
{{ render_toc(book.toc, depth=0) }}
```

**Generated HTML** (for nested TOC):
```html
<ul class="toc-list depth-0">
  <li class="toc-item">
    <a href="#" class="toc-link">Part I</a>
    <ul class="toc-list depth-1">
      <li class="toc-item">
        <a href="#" class="toc-link active">Chapter 1</a>
      </li>
      <li class="toc-item">
        <a href="#" class="toc-link">Chapter 2</a>
        <ul class="toc-list depth-2">
          <li class="toc-item">
            <a href="#" class="toc-link">Section 2.1</a>
          </li>
        </ul>
      </li>
    </ul>
  </li>
</ul>
```

**CSS for depth-based styling**:
```css
/* Base styles */
ul.toc-list {
    list-style: none;
    padding-left: 0;
    margin: 0;
}

/* Indent children */
ul.toc-list ul {
    padding-left: 20px;
}

/* Alternative: Style by depth class */
.depth-0 { font-weight: bold; }
.depth-1 { font-size: 0.95em; }
.depth-2 { font-size: 0.9em; color: #666; }
```

**Advanced: Passing context through recursion**:
```jinja2
{% macro render_toc(items, depth=0, parent_path="") %}
    <ul class="toc-list">
    {% for item in items %}
        {% set current_path = parent_path ~ "/" ~ item.title %}
        <li class="toc-item" data-path="{{ current_path }}">
            <a href="#" title="Path: {{ current_path }}">
                {{ item.title }}
            </a>

            {% if item.children %}
                {# Pass current_path as parent_path to children #}
                {{ render_toc(item.children, depth + 1, current_path) }}
            {% endif %}
        </li>
    {% endfor %}
    </ul>
{% endmacro %}
```

**When to apply**:
- File trees (directory browsers)
- Comment threads (nested replies)
- Organization charts
- Category hierarchies (e-commerce)
- Menu navigation (dropdowns)
- JSON viewers

**Alternatives**:
| Approach | Use Case | Pros | Cons |
|----------|----------|------|------|
| **Recursive macro** | Deep trees (5+ levels) | Clean, DRY, maintainable | Harder to debug |
| **Nested loops** | Shallow trees (2-3 levels) | Explicit, easy to read | Repetitive code |
| **JavaScript rendering** | Dynamic trees (expand/collapse) | Interactive | SEO issues, slower initial render |
| **Flatten + indent** | Simple display | No recursion | Loses semantic structure |

**Gotchas**:
- Jinja2 has recursion limit (default 1000) - configurable
- Context variable scope: explicitly pass variables to macro
- Performance: 1000+ nodes = consider pagination or virtual scrolling

---

## Architecture Decision Records (Inferred)

### ADR-001: Pickle for Data Persistence

**Context**:
After processing an EPUB, we need to store the structured `Book` object for the web server to load. Options:
1. Keep EPUB and re-parse on every server start
2. Convert to JSON and store as text
3. Use a database (SQLite, PostgreSQL)
4. Serialize with pickle and store as binary file

**Decision**: Use Python's `pickle` module to serialize `Book` objects to `.pkl` files.

**Rationale**:
1. **Simplicity**: Single function call to save/load (`pickle.dump/load`)
2. **Preserves structure**: Nested dataclasses, complex types work out-of-the-box
3. **Fast**: Binary format, faster than JSON parsing
4. **No dependencies**: Pickle is stdlib, no ORM or database drivers
5. **File-based**: Fits with "no database" philosophy

**Consequences**:

**Positive**:
- Zero configuration (no database setup)
- Easy backup (just copy `.pkl` files)
- Version control friendly (can commit processed books)
- Fast development (no schema migrations)

**Negative**:
- **Security risk**: Pickle can execute arbitrary code during deserialization
  - Mitigation: Only load from trusted sources (self-processed EPUBs)
  - WARNING: Never unpickle files from untrusted users!
- **Python-only**: Can't read from other languages (JavaScript, Go)
  - Mitigation: Not needed (single-language project)
- **Version fragility**: Pickle format can change between Python versions
  - Mitigation: Include version field in `Book` object
- **No querying**: Can't search across books without loading all
  - Mitigation: Acceptable for small libraries (<100 books)

**Alternatives Considered**:

**JSON**:
```python
# Would need custom serialization
def book_to_dict(book: Book) -> dict:
    return {
        "metadata": {
            "title": book.metadata.title,
            # ... manually serialize all fields
        },
        # ...
    }
```
- Pros: Human-readable, cross-language, safer
- Cons: More code, slower parsing, no native dataclass support
- **Why rejected**: Too much boilerplate for "90% vibe coded" philosophy

**SQLite**:
```python
# Would need ORM or raw SQL
db.execute("""
    INSERT INTO books (id, title, author, ...)
    VALUES (?, ?, ?, ...)
""")
```
- Pros: Queryable, industry-standard, safer
- Cons: Schema migrations, ORM complexity, overkill for simple use case
- **Why rejected**: Database is "too complex" for this project

**MessagePack / Protocol Buffers**:
- Pros: Binary, cross-language, safe
- Cons: Need schema definitions, more setup
- **Why rejected**: Additional dependency, more complexity

**When to migrate away from pickle**:
1. Multi-user support needed (→ SQLite for user data)
2. Search needed (→ SQLite FTS5)
3. Cross-language clients (→ JSON or protobuf)
4. Security becomes a concern (→ never unpickle untrusted data, migrate to JSON)

**References**:
- Pickle security: https://docs.python.org/3/library/pickle.html#module-pickle
- Save: `reader3.py:286-290`
- Load: `server.py:19-35`

---

### ADR-002: Spine-Based Content Processing (vs TOC-Based)

**Context**:
EPUB files contain two structures:
1. **Spine**: Linear reading order (guaranteed correct by spec)
2. **TOC (Table of Contents)**: Logical navigation (can be missing/malformed)

Should we process chapters based on spine order or TOC order?

**Decision**: Process content based on **spine order**, map TOC to spine.

**Rationale**:
1. **Spec compliance**: EPUB spec guarantees spine is present and ordered
2. **Robustness**: TOC can be missing, empty, or malformed (real-world observation)
3. **Linear reading**: Spine represents actual reading order (e.g., Intro → Ch1 → Ch2 → Epilogue)
4. **TOC is navigation**: TOC is for jumping around, not for defining content
5. **Separation of concerns**: Content (spine) vs navigation (TOC) are different responsibilities

**Implementation**:
```python
# Process spine (linear reading order)
for i, spine_item in enumerate(book.spine):
    item_id, linear = spine_item
    item = book.get_item_with_id(item_id)
    # ... process item
    chapter = ChapterContent(
        id=item_id,
        href=item.get_name(),  # Filename (e.g., "chapter1.html")
        order=i,  # Linear position
        # ...
    )
    spine_chapters.append(chapter)

# Parse TOC separately (navigation tree)
toc_structure = parse_toc_recursive(book.toc)

# Map TOC to spine (in template, via JavaScript)
# TOC has filenames, spine has indices
```

**Consequences**:

**Positive**:
- **Robustness**: Works even if TOC is missing (fallback to spine)
- **Correctness**: Reading order matches author's intent (spine is authoritative)
- **Simplicity**: Spine is flat list, easier to process than TOC tree
- **Prev/Next navigation**: Trivial with linear spine (chapter_index ± 1)

**Negative**:
- **TOC-to-spine mapping**: Need JavaScript to map TOC filenames to spine indices
  - Complexity: `spineMap["chapter1.html"] → 5`
  - Tradeoff: Accepted for cleaner backend
- **Anchor handling**: TOC can have anchors (`chapter1.html#section2`), spine doesn't
  - Mitigation: Preserve anchors in TOC, ignore during mapping

**Alternatives Considered**:

**TOC-Based Processing**:
```python
# Would need to traverse TOC tree, find files
def extract_chapters_from_toc(toc_tree):
    chapters = []
    for entry in toc_tree:
        file = book.get_item_by_href(entry.href)
        # ... process
        if entry.children:
            chapters.extend(extract_chapters_from_toc(entry.children))
    return chapters
```
- Pros: TOC order might match user expectations
- Cons: What if TOC is missing? What's the order for files not in TOC?
- **Why rejected**: Less robust, more complex, no guarantee of correctness

**Hybrid (Spine + TOC titles)**:
```python
# Use spine for order, but try to get title from TOC
toc_titles = {entry.file_href: entry.title for entry in flatten_toc(toc)}
for i, spine_item in enumerate(book.spine):
    title = toc_titles.get(item.get_name(), f"Section {i+1}")
```
- Pros: Best of both worlds (spine order, TOC titles)
- Cons: More complex mapping
- **Why not implemented**: Spine order is enough, TOC titles are shown in sidebar

**Evidence from code**:
- Spine processing: `reader3.py:224-271`
- Fallback TOC: `reader3.py:135-146` (generates TOC from spine when missing)
- JavaScript mapping: `templates/reader.html:127-151`

**Real-world observation**:
Many EPUBs have:
- Perfect spine (always present, always correct)
- Broken TOC (missing, incomplete, wrong filenames)

Example: Some auto-converted EPUBs from PDF have no TOC but perfect spine.

---

### ADR-003: LRU Cache for Book Loading (vs No Cache)

**Context**:
Unpickling a `Book` object takes 50-200ms per request. For typical reading workflow:
1. User visits library → loads all books (10 books = 2 seconds)
2. User clicks book → loads book again (200ms)
3. User navigates chapters → loads book every time (200ms per chapter change)

This creates a poor user experience (slow chapter navigation).

**Decision**: Use `@lru_cache(maxsize=10)` to cache loaded books in memory.

**Rationale**:
1. **Access pattern**: Users read one book at a time, navigate many chapters
2. **Working set**: Typical user has <10 books in library
3. **Immutability**: Books never change after processing (no cache invalidation needed)
4. **Memory acceptable**: 10 books ≈ 50-500MB RAM (reasonable for local deployment)
5. **Simplicity**: One decorator, no external cache, thread-safe

**Implementation**:
```python
from functools import lru_cache

@lru_cache(maxsize=10)
def load_book_cached(folder_name: str) -> Optional[Book]:
    # Cold: 180ms (pickle.load)
    # Warm: <1ms (cache hit)
    with open(path, "rb") as f:
        return pickle.load(f)
```

**Consequences**:

**Positive**:
- **Performance**: 180x speedup on cache hit (180ms → <1ms)
- **User experience**: Chapter navigation feels instant
- **Simplicity**: Built-in, no external dependencies
- **Thread-safe**: lru_cache handles concurrency (FastAPI is async)
- **Introspection**: `cache_info()` shows hits/misses

**Negative**:
- **Memory usage**: Grows with cached books (~50MB per book)
  - Mitigation: maxsize=10 limits growth
- **Stale cache**: If book re-processed, must restart server
  - Mitigation: Acceptable for local use (rare re-processing)
- **No TTL**: Cache never expires (only LRU eviction)
  - Mitigation: Not needed (books immutable)

**Performance measurements**:
```python
# Test script
import time
from server import load_book_cached

# Cold start (cache miss)
start = time.time()
book = load_book_cached("dracula_data")
print(f"Cache miss: {(time.time() - start) * 1000:.1f}ms")  # 180ms

# Warm (cache hit)
start = time.time()
book = load_book_cached("dracula_data")
print(f"Cache hit: {(time.time() - start) * 1000:.1f}ms")   # <1ms
```

**Alternatives Considered**:

**No Cache**:
- Pros: No memory usage, no stale data
- Cons: 180ms per request (unusable for navigation)
- **Why rejected**: User experience is terrible

**Redis**:
```python
import redis
import pickle

r = redis.Redis()

def load_book_cached(folder_name: str):
    # Check Redis
    cached = r.get(folder_name)
    if cached:
        return pickle.loads(cached)

    # Load and cache
    book = load_book_from_disk(folder_name)
    r.set(folder_name, pickle.dumps(book))
    return book
```
- Pros: Multi-process cache, TTL support, eviction policies
- Cons: Infrastructure dependency, network latency, complexity
- **Why rejected**: Overkill for single-user local deployment

**TTL Cache** (cachetools library):
```python
from cachetools import TTLCache, cached

cache = TTLCache(maxsize=10, ttl=300)  # 5-minute TTL

@cached(cache)
def load_book_cached(folder_name: str):
    # ...
```
- Pros: Auto-invalidation after 5 minutes
- Cons: Additional dependency, unnecessary (books don't change)
- **Why rejected**: LRU is sufficient (no stale data problem)

**Cache invalidation** (manual):
```python
# If book re-processed, clear cache
load_book_cached.cache_clear()

# Or clear specific entry (requires custom wrapper)
del cache[folder_name]
```

**When to change**:
1. Multi-user → Redis (shared cache across processes)
2. Large library (>100 books) → Consider database with query cache
3. Dynamic content → TTL cache (auto-invalidation)

**References**:
- Implementation: `server.py:19-35`
- Python docs: https://docs.python.org/3/library/functools.html#functools.lru_cache

---

### ADR-004: Embedded CSS/JS (vs External Files)

**Context**:
Frontend needs styling (CSS) and interactivity (JavaScript). Options:
1. Embedded in HTML (`<style>` and `<script>` tags)
2. External files (`<link rel="stylesheet">` and `<script src>`)
3. Component libraries (React, Vue, Alpine.js)

**Decision**: Embed all CSS and JavaScript directly in HTML templates.

**Rationale**:
1. **Simplicity**: No asset pipeline, no build step, no bundling
2. **Single file**: Everything needed in one HTML file (view source works)
3. **No routing**: No need to serve static files (fewer routes)
4. **Minimal JS**: Only ~30 lines of JavaScript (TOC navigation)
5. **Cacheable**: HTML includes CSS/JS, browser caches entire response

**Implementation**:
```html
<!DOCTYPE html>
<html>
<head>
    <style>
        /* All CSS here */
        body { font-family: Georgia, serif; }
        /* ... 100+ lines of CSS ... */
    </style>
</head>
<body>
    <!-- HTML content -->

    <script>
        /* All JavaScript here */
        function findAndGo(filename) {
            /* ... */
        }
    </script>
</body>
</html>
```

**Consequences**:

**Positive**:
- **No build step**: Edit template, refresh browser (instant feedback)
- **No file serving**: Don't need `/static/` route or middleware
- **Debuggable**: View source shows everything (no source maps needed)
- **Portable**: HTML file is self-contained
- **Fast development**: No webpack, no npm scripts

**Negative**:
- **No caching**: CSS/JS re-downloaded with every page
  - Mitigation: Gzip compression helps, pages are small (~10KB)
- **No reusability**: CSS duplicated in library.html and reader.html
  - Mitigation: Only 2 templates, acceptable duplication
- **No minification**: CSS/JS served uncompressed
  - Mitigation: File sizes are small, local network is fast
- **Hard to test**: JavaScript not in separate file
  - Mitigation: Minimal JS (~30 lines), tested manually

**Metrics**:
```
library.html:  2.1 KB (HTML + CSS)
reader.html:   7.8 KB (HTML + CSS + JS)
Total:        ~10 KB (uncompressed)
With gzip:    ~3 KB (70% compression)
```

**Alternatives Considered**:

**External CSS/JS**:
```html
<link rel="stylesheet" href="/static/style.css">
<script src="/static/app.js"></script>
```

```python
# Would need static file serving
from fastapi.staticfiles import StaticFiles
app.mount("/static", StaticFiles(directory="static"), name="static")
```
- Pros: Cacheable, reusable, testable
- Cons: More files, routing complexity, build step (if bundling)
- **Why rejected**: Overkill for 2 templates and 30 lines of JS

**Component Framework** (React, Vue, Alpine.js):
```html
<!-- Alpine.js example -->
<script src="https://unpkg.com/alpinejs"></script>
<div x-data="{ open: false }">
    <button @click="open = !open">Toggle</button>
    <div x-show="open">Content</div>
</div>
```
- Pros: Reactivity, components, ecosystem
- Cons: Learning curve, dependency, overkill for simple app
- **Why rejected**: No need for reactivity, server-side rendering is enough

**Template inheritance** (Jinja2 blocks):
```jinja2
{# base.html #}
<style>
    /* Shared CSS */
</style>
{% block content %}{% endblock %}

{# library.html #}
{% extends "base.html" %}
{% block content %}
    <!-- Library-specific content -->
{% endblock %}
```
- Pros: DRY (Don't Repeat Yourself), shared styles
- Cons: More complexity, multiple files
- **Why not implemented**: Only 2 templates, small CSS duplication acceptable

**When to change**:
1. More templates (>5) → Extract shared CSS to base template
2. Complex JS (>100 lines) → External file + bundling
3. Interactive features → Consider Alpine.js or HTMX
4. Performance critical → Extract, minify, version assets

**References**:
- Library template: `templates/library.html`
- Reader template: `templates/reader.html`

---

## Patterns & Best Practices

### Pattern 1: Filename Sanitization for User Input

**Problem**: User-provided filenames can contain OS-specific or dangerous characters.

**Solution**:
```python
import os

def sanitize_filename(filename: str) -> str:
    """
    Keep only: alphanumeric, dot, underscore, hyphen
    Remove: path separators, special chars, unicode control chars
    """
    safe = "".join([
        c for c in filename
        if c.isalpha() or c.isdigit() or c in '._-'
    ]).strip()

    # Ensure non-empty
    return safe if safe else "unnamed"

# Usage
original = "my photo (2023).jpg"
safe = sanitize_filename(original)  # "myphoto2023.jpg"

dangerous = "../../etc/passwd"
safe = sanitize_filename(dangerous)  # "etcpasswd"
```

**Why**:
- Prevents path traversal (../)
- Prevents OS-specific issues (NUL byte, :, <, >)
- Ensures filesystem compatibility (Windows, Linux, macOS)

**From codebase**: `reader3.py:199`

---

### Pattern 2: Dual-Key Mapping for Path Resolution

**Problem**: EPUB image paths can be:
- Full internal path: `OEBPS/Images/cover.jpg`
- Basename only: `cover.jpg`
- URL-encoded: `Images/cover%201.jpg`

HTML `<img src>` might reference any of these.

**Solution**: Build a map with multiple keys per image.
```python
from urllib.parse import unquote

image_map = {}

for item in book.get_items():
    if item.get_type() == ebooklib.ITEM_IMAGE:
        original_fname = os.path.basename(item.get_name())
        safe_fname = sanitize_filename(original_fname)

        rel_path = f"images/{safe_fname}"

        # Multiple keys for same value
        image_map[item.get_name()] = rel_path          # Full path
        image_map[original_fname] = rel_path           # Basename
        image_map[unquote(item.get_name())] = rel_path  # Decoded

# Lookup with fallback
src = img.get('src')
src_decoded = unquote(src)
filename = os.path.basename(src_decoded)

if src_decoded in image_map:
    img['src'] = image_map[src_decoded]
elif filename in image_map:
    img['src'] = image_map[filename]
```

**Why**:
- Handles messy EPUB files (inconsistent referencing)
- Handles URL encoding (space → %20)
- Handles both relative and absolute paths

**From codebase**: `reader3.py:206-210`, `reader3.py:242-249`

---

### Pattern 3: Fallback Strategies for Missing Data

**Problem**: Real-world data is messy (missing TOC, missing metadata).

**Solution**: Always have a Plan B.

**Example 1: Missing TOC**
```python
toc = parse_toc_recursive(book.toc)
if not toc:
    # Fallback: generate from spine
    toc = get_fallback_toc(book)
```

**Example 2: Missing Metadata**
```python
def get_one(key):
    data = book_obj.get_metadata('DC', key)
    return data[0][0] if data else None

title = get_one('title') or "Untitled"  # Default
language = get_one('language') or "en"   # Default
```

**Example 3: Missing Body Tag**
```python
body = soup.find('body')
if body:
    html = "".join([str(x) for x in body.contents])
else:
    html = str(soup)  # Fallback: use entire document
```

**Principle**: Never assume data is present. Always have a sensible default.

**From codebase**: `reader3.py:135-146`, `reader3.py:162-163`, `reader3.py:256-260`

---

### Pattern 4: Extract Body Content Only (Remove Wrapping Tags)

**Problem**: EPUB HTML has full document structure (`<html>`, `<head>`, `<body>`), but we want just content for embedding.

**Solution**: Extract body's *contents*, not body itself.
```python
soup = BeautifulSoup(html, 'html.parser')
body = soup.find('body')

# Wrong: str(body) includes <body> tags
# Right: Extract contents only
if body:
    content = "".join([str(x) for x in body.contents])
else:
    content = str(soup)

# Result: <p>...</p><h1>...</h1> (no <body> wrapper)
```

**Why**:
- Embedding in template: `{{ content | safe }}` (no nested `<body>`)
- Clean HTML: No duplicate `<html>` or `<head>` tags
- Styling control: Can apply container styles without fighting EPUB's body styles

**From codebase**: `reader3.py:256-260`

---

### Pattern 5: Progress Logging for Long Operations

**Problem**: EPUB processing takes 10-60 seconds. No output → user thinks it's frozen.

**Solution**: Print progress at each major step.
```python
def process_epub(epub_path: str, output_dir: str) -> Book:
    print(f"Loading {epub_path}...")          # Step 1
    book = epub.read_epub(epub_path)

    print("Extracting images...")             # Step 2
    image_map = extract_images(book, output_dir)

    print("Parsing Table of Contents...")     # Step 3
    toc = parse_toc_recursive(book.toc)

    print("Processing chapters...")           # Step 4
    for i, spine_item in enumerate(book.spine):
        # Process...

    print("\n--- Summary ---")                # Final
    print(f"Title: {book.metadata.title}")
```

**Output**:
```
Loading dracula.epub...
Extracting images...
Parsing Table of Contents...
Processing chapters...

--- Summary ---
Title: Dracula
Authors: Bram Stoker
Physical Files (Spine): 48
TOC Root Items: 27
Images extracted: 5
```

**Why**:
- User feedback (know it's working)
- Debug info (which step failed?)
- Satisfaction (progress feels faster)

**From codebase**: `reader3.py:178`, `191`, `213`, `220`, `308-314`

---

## Lessons Learned

### Lesson 1: Simplicity is a Feature

**What**: This codebase intentionally avoids "best practices" like databases, testing, CI/CD.

**Why it works**:
- Small scope (EPUB reader, not a SaaS platform)
- Single user (no concurrent access)
- Immutable data (books don't change)
- Throwaway mindset ("ask your LLM to change it")

**When to apply**: Side projects, prototypes, internal tools, demos.

**When not to apply**: Production systems, multi-user, financial data, healthcare.

**Quote from author**:
> "Code is ephemeral now and libraries are over, ask your LLM to change it in whatever way you like."

**Takeaway**: Not every project needs enterprise patterns. Sometimes, a 400-line script is the right tool.

---

### Lesson 2: Real-World Data is Messy

**Observation**: EPUBs from the wild have:
- Missing or empty TOC
- Broken image paths
- Missing metadata
- Invalid HTML
- Mixed encodings

**Lesson**: Always assume data is malformed. Build defensively.

**Defensive patterns**:
- Fallback strategies (missing TOC → generate from spine)
- Sensible defaults (missing title → "Untitled")
- Error handling (corrupted pickle → return None, log error)
- Multiple lookup keys (image paths)

**From codebase**: Everywhere (TOC fallback, metadata defaults, image mapping)

---

### Lesson 3: Cache-Aside is Underutilized

**Observation**: Most web apps query the database on every request, even for immutable data.

**Lesson**: If data doesn't change, cache it. `@lru_cache` is one decorator.

**Before caching**:
```python
def get_user_profile(user_id):
    return db.query("SELECT * FROM users WHERE id = ?", user_id)

# Every request: database query (10-50ms)
```

**After caching**:
```python
@lru_cache(maxsize=1000)
def get_user_profile(user_id):
    return db.query("SELECT * FROM users WHERE id = ?", user_id)

# First request: database query (10-50ms)
# Subsequent requests: cache hit (<1ms)
```

**When it works**:
- Immutable data (or rarely-changing)
- High read-to-write ratio
- Bounded working set (maxsize prevents memory explosion)

**When it doesn't**:
- Frequently-changing data (cache is always stale)
- Unbounded keys (memory leak)

---

### Lesson 4: Two-Process Architecture Underrated

**Observation**: Many apps bundle batch processing into the web server.

**Problem**:
- Heavy processing blocks request threads
- Processing errors crash the server
- Can't scale processing independently

**Lesson**: Separate batch processing from serving, even if they share a codebase.

**Example from wild**:
- Rails app with background jobs (Sidekiq)
- Django app with Celery workers
- Node.js with worker threads

**Reader3's approach**:
- CLI tool for processing (heavy, blocking, on-demand)
- Web server for serving (light, async, continuous)

**Tradeoff**: More moving parts, but cleaner separation.

---

### Lesson 5: Inline CSS/JS is Underrated for Small Apps

**Observation**: Modern web dev assumes build tooling (webpack, vite, rollup).

**Lesson**: For <100 lines of JS and <200 lines of CSS, just inline it.

**Benefits**:
- No build step (edit, refresh, done)
- No asset pipeline (fewer moving parts)
- View source works (debugging)

**Tradeoffs**:
- No caching (CSS/JS re-downloaded)
- No minification (larger files)
- No reusability (duplicated across templates)

**When inline is fine**:
- <5 templates
- <100 lines of JS total
- Local/internal deployment (fast network)

**When to bundle**:
- Complex JS (React, Vue, etc.)
- Many templates (>10)
- External users (slow networks)

---

## Reusable Mental Models

### Mental Model 1: "Flat Spine, Hierarchical TOC"

**Concept**: Content is linear (spine), navigation is hierarchical (TOC).

**Analogy**: Book spine is page order (1, 2, 3...), TOC is logical structure (Part I → Chapter 1 → Section 1.1).

**Application**: Any system with linear data + hierarchical navigation.
- Filesystem: Files are flat (inodes), directories are hierarchical
- Git: Commits are linear (DAG), branches/tags are hierarchical
- Video: Frames are linear, chapters are hierarchical

**Key insight**: Don't conflate content order with navigation structure.

---

### Mental Model 2: "Cache-Aside for Immutability"

**Concept**: If data never changes, cache it aggressively.

**Pattern**:
1. Check cache
2. If miss, load from source
3. Store in cache
4. Return

**Application**:
- Static site generation (build once, serve forever)
- Asset fingerprinting (bundle.abc123.js → immutable)
- CDNs (cache static assets)

**Key insight**: Immutability unlocks caching. Make data immutable when possible.

---

### Mental Model 3: "Whitelist, Don't Blacklist"

**Concept**: Security is easier when you explicitly allow, rather than explicitly deny.

**Example**:
```python
# Blacklist (dangerous - easy to miss things)
forbidden_tags = ['script', 'iframe', 'object']
for tag in soup(forbidden_tags):
    tag.decompose()

# Whitelist (safer - everything not allowed is removed)
allowed_tags = ['p', 'h1', 'h2', 'img', 'a']
for tag in soup.find_all():
    if tag.name not in allowed_tags:
        tag.decompose()
```

**Application**:
- HTML sanitization
- Input validation (allowed characters, not forbidden ones)
- API authorization (allow certain roles, not deny others)

**Key insight**: It's easier to enumerate what's safe than what's dangerous.

---

## Conclusion

This codebase, despite being "90% vibe coded," contains valuable patterns:

1. **HTML sanitization** (security + UX)
2. **Recursive tree parsing** (hierarchical data)
3. **Path security** (preventing traversal)
4. **Cache-aside pattern** (performance)
5. **Two-process architecture** (separation of concerns)
6. **Jinja2 recursive macros** (nested rendering)

These patterns are **reusable** across domains:
- Content management systems
- File browsers
- E-commerce (category trees)
- Comment systems
- Admin panels

The **architectural decisions** (pickle, spine-based, LRU cache, embedded CSS) reflect thoughtful tradeoffs:
- Simplicity over scalability
- Pragmatism over purity
- Speed over security (for local use)

The **mental models** (flat content + hierarchical navigation, whitelist > blacklist, immutability enables caching) generalize beyond this codebase.

**Final takeaway**: Even "simple" code contains intelligence. Extracting it makes that intelligence reusable.
