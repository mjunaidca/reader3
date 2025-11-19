# Reader3 Specification

## Problem Statement

Reading EPUB books alongside Large Language Models (LLMs) is currently cumbersome. The workflow requires:
- Manually extracting text from EPUB files
- Navigating complex EPUB structures to find specific chapters
- Copy-pasting content from various tools into LLM conversations
- Managing multiple applications for a simple reading task

**The core pain**: There's no lightweight, self-hosted tool optimized for "reading along with AI" that makes it trivial to access and copy chapter text for LLM conversations.

## System Intent

### What
Reader3 is a lightweight, self-hosted EPUB reader that processes EPUB books into a clean web interface, displaying one chapter at a time for easy reading and text extraction.

### Why
To enable seamless "reading along with LLMs" by providing:
- Chapter-by-chapter navigation without EPUB client software
- Clean text rendering optimized for copy-paste to LLM conversations
- Minimal setup and maintenance (no database, no complex configuration)
- Self-hosted privacy (books never leave your machine)

### Target Users
- Readers who discuss books with LLMs (ChatGPT, Claude, etc.)
- People wanting a simple EPUB reader without bloated features
- Users who prefer self-hosted solutions over cloud readers
- Developers who want to modify reading tools to their preferences

### Core Value Proposition
"90% vibe coded" simplicity - intentionally minimal, easily modifiable, no pretense of being a production-grade system. Code is ephemeral; ask your LLM to customize it however you want.

## Functional Requirements

### FR-1: EPUB Processing
**What**: Convert EPUB files into structured, web-serveable format

**Inputs**:
- EPUB file path (EPUB2 or EPUB3 format)
- Output directory name (auto-generated from filename)

**Outputs**:
- Pickled Book object (`book.pkl`)
- Extracted images in `images/` subdirectory
- Processed HTML content with rewritten image paths

**Success Criteria**:
- All chapters extracted and ordered correctly
- Table of contents hierarchy preserved
- Images accessible via relative paths
- Metadata (title, author, language, etc.) extracted
- Dangerous HTML tags removed (script, iframe, form, etc.)
- Both HTML and plain text versions available per chapter

**Processing Steps**:
1. Parse EPUB structure using ebooklib
2. Extract metadata (title, authors, publisher, etc.)
3. Process images (extract, sanitize filenames, build path map)
4. Parse table of contents (recursive structure)
5. Process spine content (linear reading order)
6. Clean HTML (remove dangerous tags, rewrite image paths)
7. Extract plain text for each chapter
8. Serialize to pickle file

### FR-2: Library Management
**What**: Display all processed books in a browsable library view

**Inputs**:
- Scan current directory for folders ending in `_data`
- Load `book.pkl` from each folder

**Outputs**:
- Grid view of books with:
  - Title
  - Author(s)
  - Number of sections
  - "Read Book" link

**Success Criteria**:
- Only valid processed books displayed
- Books with missing/corrupted pkl files ignored gracefully
- Responsive grid layout (mobile-friendly)
- Direct navigation to first chapter

**Edge Cases**:
- Empty library shows helpful message
- Corrupted pickle files logged but don't crash app
- Folders without book.pkl ignored

### FR-3: Chapter Reading Interface
**What**: Display book content one chapter at a time with navigation

**Inputs**:
- Book ID (folder name)
- Chapter index (0-based)

**Outputs**:
- Two-pane layout:
  - **Sidebar**: Table of contents (hierarchical), book title, library link
  - **Main**: Chapter HTML content, prev/next navigation, chapter counter

**Success Criteria**:
- Clean typography (Georgia serif, 1.15em, 1.8 line-height)
- Images rendered inline with proper paths
- Active TOC item highlighted
- Previous/Next buttons disabled at boundaries
- Chapter counter shows "Section X of Y"
- TOC items clickable (JavaScript mapping to chapter indices)

**Navigation Requirements**:
- Linear navigation: Previous/Next buttons
- TOC navigation: Click any TOC item to jump to that chapter
- Breadcrumb navigation: "Back to Library" link

### FR-4: Image Serving
**What**: Serve images from processed EPUB books

**Inputs**:
- Book ID (folder name)
- Image filename

**Outputs**:
- Image file response (JPEG, PNG, GIF, SVG, etc.)

**Success Criteria**:
- Images render in chapter content
- Path traversal attacks prevented (use `os.path.basename`)
- 404 error for missing images
- No authentication required (local-only server)

## Non-Functional Requirements

### NFR-1: Simplicity
**Priority**: P0 (Core Philosophy)

**Characteristics**:
- No database (file-based storage only)
- No user accounts (single-user, local-only)
- No complex configuration (hardcoded defaults)
- Minimal dependencies (5 Python packages)
- Total codebase: ~425 lines of Python + HTML

**Rationale**: "Code is ephemeral now and libraries are over, ask your LLM to change it in whatever way you like." The system should be simple enough to understand and modify in minutes.

### NFR-2: Performance
**Priority**: P1

**Characteristics**:
- LRU cache for book loading (maxsize=10)
- Pickle deserialization ~50-200ms (fast enough)
- No database query latency
- Static image serving with direct file reads

**Success Criteria**:
- Library page loads < 200ms (cached)
- Chapter navigation < 100ms (cached book)
- First chapter load < 500ms (cold start)
- Image loads < 50ms (local filesystem)

### NFR-3: Usability
**Priority**: P1

**Characteristics**:
- Clean, distraction-free reading interface
- Responsive design (mobile, tablet, desktop)
- Keyboard-friendly navigation
- Copy-paste optimized (plain text extraction)
- Visual hierarchy (serif body, sans-serif UI)

**Design Principles**:
- Content-first (max-width: 700px, ample padding)
- High contrast (black text on white background)
- Readable typography (1.8 line-height, justified text)
- Minimal UI chrome (no unnecessary buttons)

### NFR-4: Security
**Priority**: P2 (Low - local-only deployment assumed)

**Current Protections**:
- Path traversal prevention (`os.path.basename` for user inputs)
- Filename sanitization (alphanumeric + ._- only)
- Dangerous HTML tag removal (script, iframe, form)

**Known Vulnerabilities** (Accepted for simplicity):
- Pickle deserialization RCE (don't process untrusted EPUBs)
- XSS via book content (HTML marked as `| safe` in template)
- No CSRF protection (not needed for local read-only)
- No authentication (single-user assumption)

**Threat Model**: Assumes trusted user processing trusted EPUB files on a local machine not exposed to networks.

### NFR-5: Maintainability
**Priority**: P2

**Characteristics**:
- Self-documenting code (clear dataclasses, type hints)
- Inline comments for non-obvious logic
- Single-file modules (no complex package structure)
- Standard library + popular packages only

**Developer Experience**:
- Uses `uv` for fast dependency management
- Python 3.10+ for modern syntax
- No build step (pure Python + HTML)
- Instant reload during development (uvicorn --reload)

### NFR-6: Portability
**Priority**: P1

**Characteristics**:
- Cross-platform (Linux, macOS, Windows)
- No OS-specific dependencies
- Filesystem-only persistence (no database drivers)
- Standard HTTP server (accessible from any browser)

**Deployment**:
- Single command to process EPUB: `uv run reader3.py book.epub`
- Single command to start server: `uv run server.py`
- No configuration files required
- No installation beyond `uv` package manager

## Constraints

### External Dependencies
- **Python Packages**:
  - `ebooklib>=0.20` - EPUB parsing
  - `beautifulsoup4>=4.14.2` - HTML parsing and cleaning
  - `fastapi>=0.121.2` - Web framework
  - `jinja2>=3.1.6` - Template rendering
  - `uvicorn>=0.38.0` - ASGI server

- **System Requirements**:
  - Python 3.10+ (for modern type hints, dataclasses)
  - File system with read/write access
  - ~10MB disk space per processed book (varies by images)

### Data Format Constraints
- **Input**: EPUB2 and EPUB3 files only
- **Output**: Pickle protocol 5 (Python 3.8+ compatible)
- **Images**: JPEG, PNG, GIF, SVG (whatever EPUB contains)
- **HTML**: UTF-8 encoded, cleaned subset of XHTML

### Deployment Constraints
- **Single-user**: No multi-user support or session management
- **Local-only**: Designed for 127.0.0.1 (not production-hardened)
- **Synchronous**: No async book processing (blocking CLI)
- **Stateless server**: No reading progress persistence

### Technical Constraints
- **No database**: All data in pickle files and filesystem
- **No authentication**: Assumes trusted local environment
- **No background jobs**: Processing is foreground CLI operation
- **No search index**: Plain text extracted but not searchable

## Non-Goals & Out of Scope

**Explicitly excluded** (inferred from missing implementation):

- **Book editing**: No EPUB modification or annotation features
- **Cloud sync**: No backup, sync, or cross-device features
- **Social features**: No sharing, ratings, or reviews
- **Advanced reader features**: No bookmarks, highlights, notes
- **Format conversion**: No PDF, MOBI, AZW3 support
- **Production deployment**: No HTTPS, no reverse proxy config, no containerization
- **Accessibility**: No screen reader optimization, no ARIA labels
- **Internationalization**: No multi-language UI (English only)
- **Analytics**: No usage tracking, no reading statistics
- **Library management**: No tags, collections, or metadata editing
- **Search**: No full-text search across books or chapters
- **Performance at scale**: Not optimized for thousands of books

## Success Criteria

### Processing Success
- [x] EPUB file successfully parsed without errors
- [x] All spine items converted to ChapterContent objects
- [x] Table of contents hierarchy preserved
- [x] Images extracted and path-mapped correctly
- [x] Metadata (title, authors) extracted
- [x] book.pkl file created and loadable

### Reading Success
- [x] Library shows all processed books
- [x] Clicking "Read Book" loads first chapter
- [x] Chapter content renders with correct formatting
- [x] Images display inline
- [x] Previous/Next navigation works
- [x] TOC shows hierarchical structure
- [x] Active TOC item highlighted
- [x] TOC items navigate to correct chapters

### User Experience Success
- [x] User can process an EPUB in < 1 minute
- [x] User can navigate chapters without confusion
- [x] User can copy chapter text cleanly (no formatting junk)
- [x] User can open multiple books in library
- [x] User can read on mobile device without layout issues

### Developer Experience Success
- [x] Codebase understandable in < 30 minutes
- [x] Developer can modify templates without touching Python
- [x] Developer can add new routes in < 10 lines
- [x] LLM can suggest modifications successfully (given simple codebase)

## Known Gaps & Technical Debt

### Gap 1: No Search Functionality
- **Issue**: Plain text is extracted but not indexed or searchable
- **Evidence**: `text` field in ChapterContent (reader3.py:29) is populated but never queried
- **Impact**: Users cannot find specific passages or terms across chapters
- **Recommendation**: Add simple keyword search across `text` fields, or integrate with SQLite FTS5

### Gap 2: No Reading Progress Persistence
- **Issue**: No bookmark or "last read position" saved
- **Evidence**: Server is stateless, no session management (server.py)
- **Impact**: Users must remember where they left off
- **Recommendation**: Use browser localStorage or server-side session storage

### Gap 3: No Book Management UI
- **Issue**: Books can only be deleted by manually removing folders
- **Evidence**: No DELETE routes or admin interface (server.py)
- **Impact**: Users must use command line to manage library
- **Recommendation**: Add "Delete Book" button in library view

### Gap 4: No Error Handling for Malformed EPUBs
- **Issue**: Processing fails silently or crashes on invalid EPUBs
- **Evidence**: No try/except around `epub.read_epub()` (reader3.py:179)
- **Impact**: User gets stack trace instead of helpful error message
- **Recommendation**: Validate EPUB before processing, show user-friendly errors

### Gap 5: No Tests
- **Issue**: Zero test coverage (unit, integration, e2e)
- **Evidence**: No test files in repository
- **Impact**: Refactoring is risky, regressions undetected
- **Recommendation**: Add pytest tests for:
  - EPUB parsing edge cases
  - TOC recursive parsing
  - Image path rewriting
  - API endpoints

### Gap 6: Pickle Security Risk
- **Issue**: Pickle deserialization can execute arbitrary code
- **Evidence**: `pickle.load()` without validation (server.py:31)
- **Impact**: Malicious .pkl file could compromise system
- **Recommendation**: Migrate to JSON or use restricted unpickler

### Gap 7: No Configuration System
- **Issue**: Host, port, book directory hardcoded
- **Evidence**: `host="127.0.0.1", port=8123` (server.py:110), `BOOKS_DIR = "."` (server.py:17)
- **Impact**: Users must edit code to change settings
- **Recommendation**: Use environment variables or config file (TOML)

### Gap 8: No Logging or Observability
- **Issue**: Minimal logging, no metrics, no error tracking
- **Evidence**: Only print statements (reader3.py:178, 191, 213, 220)
- **Impact**: Debugging production issues is difficult
- **Recommendation**: Use Python logging module, add structured logs

### Gap 9: XSS Vulnerability in Book Content
- **Issue**: Book HTML rendered with `| safe` filter (templates/reader.html:101)
- **Evidence**: No Content Security Policy, no sanitization beyond tag removal
- **Impact**: Malicious EPUB could inject JavaScript
- **Recommendation**: Use bleach library for HTML sanitization, add CSP headers

### Gap 10: No TOC Navigation Without JavaScript
- **Issue**: TOC links require JavaScript to map filenames to indices
- **Evidence**: `onclick="findAndGo('{{ item.file_href }}')"` (templates/reader.html:81)
- **Impact**: Users with JavaScript disabled cannot use TOC
- **Recommendation**: Server-side filename-to-index mapping in route

## Technical Debt Prioritization

| Priority | Debt Item | Impact | Effort | ROI | Location |
|----------|-----------|--------|--------|-----|----------|
| **P0** | Pickle security risk | High | Medium | High | server.py:31 |
| **P1** | No error handling for malformed EPUBs | Medium | Low | High | reader3.py:179 |
| **P1** | No configuration system | Medium | Low | High | server.py:17,110 |
| **P2** | No search functionality | Medium | High | Medium | New feature |
| **P2** | No reading progress | Medium | Medium | Medium | New feature |
| **P2** | No tests | High | High | Medium | New files |
| **P3** | XSS in book content | Low | Medium | Low | templates/reader.html:101 |
| **P3** | No logging/observability | Low | Low | Medium | All files |
| **P3** | No book deletion UI | Low | Low | Low | server.py |
| **P4** | TOC requires JavaScript | Low | Medium | Low | templates/reader.html:81 |

## Regeneration Readiness Assessment

### Can this spec regenerate the system?
**Yes**, with the following coverage:

- [x] All functional requirements documented
- [x] Architecture patterns identified
- [x] Data structures specified
- [x] UI/UX requirements clear
- [x] Non-functional requirements quantified
- [x] Known gaps and debt catalogued
- [ ] API contracts formalized (informal routes exist)
- [ ] Test scenarios specified (zero tests currently)
- [ ] Deployment procedures documented (minimal currently)

### What's missing for complete regeneration?
1. **OpenAPI specification** for API routes (currently informal)
2. **Test scenarios** with expected inputs/outputs
3. **Deployment guide** (Docker, systemd, etc.)
4. **Development setup instructions** (beyond README)
5. **Performance benchmarks** (quantitative targets)
6. **Security threat model** (formal analysis)

### Confidence Level: 85%
A competent developer could rebuild this system from spec in **4-8 hours**, given:
- Familiarity with FastAPI and ebooklib
- Understanding of EPUB structure
- Access to sample EPUB files for testing

The remaining 15% uncertainty comes from:
- Undocumented edge cases in EPUB parsing
- Specific HTML cleaning heuristics (learned through testing)
- TOC-to-Spine mapping logic (complex, underspecified)

## Version History

- **v3.0** (Current): Spine-based processing, recursive TOC, LRU cache
- **v2.x** (Inferred): Likely simpler TOC parsing, no caching
- **v1.x** (Inferred): Basic EPUB extraction, minimal UI

## License

MIT License - Simple, permissive, no strings attached.
