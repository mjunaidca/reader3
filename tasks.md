# Reader3 Implementation Tasks

## Overview

This document provides a step-by-step task breakdown for building Reader3 from scratch using the specification and architecture plan. Tasks are organized by implementation phase, with each task including:

- Clear acceptance criteria
- Estimated effort (T-shirt sizing: S/M/L/XL)
- Dependencies on other tasks
- File locations for reference

**Total Estimated Effort**: 20-30 hours for experienced developer

## Phase 1: Project Setup & Infrastructure

**Goal**: Bootstrap project with dependencies and basic structure

### Task 1.1: Initialize Python Project
**Effort**: S (30 min)

**Steps**:
```bash
# Create project directory
mkdir reader3
cd reader3

# Create pyproject.toml
cat > pyproject.toml <<EOF
[project]
name = "reader3"
version = "0.1.0"
description = "Simple EPUB reader web app"
readme = "README.md"
requires-python = ">=3.10"
dependencies = [
    "beautifulsoup4>=4.14.2",
    "ebooklib>=0.20",
    "fastapi>=0.121.2",
    "jinja2>=3.1.6",
    "uvicorn>=0.38.0",
]
EOF

# Initialize uv
uv sync
```

**Acceptance Criteria**:
- [x] `pyproject.toml` exists with correct dependencies
- [x] `uv.lock` generated
- [x] `uv run python` works

**Reference**: `/home/user/reader3/pyproject.toml`

---

### Task 1.2: Create Directory Structure
**Effort**: S (15 min)

**Steps**:
```bash
mkdir -p templates
touch reader3.py
touch server.py
touch templates/library.html
touch templates/reader.html
```

**Structure**:
```
reader3/
├── pyproject.toml
├── uv.lock
├── reader3.py          # EPUB processor
├── server.py           # Web server
└── templates/
    ├── library.html    # Library view template
    └── reader.html     # Reader view template
```

**Acceptance Criteria**:
- [x] All files created
- [x] Directory structure matches spec

---

### Task 1.3: Write README
**Effort**: S (20 min)

**Content**:
- Project description (EPUB reader for LLM workflows)
- Installation instructions (uv-based)
- Usage examples (process EPUB, start server)
- License (MIT)

**Acceptance Criteria**:
- [x] README.md exists
- [x] Contains usage examples
- [x] Explains the "read with LLMs" use case

**Reference**: `/home/user/reader3/README.md`

---

## Phase 2: Data Model Layer

**Goal**: Define domain objects with type safety

### Task 2.1: Define Data Structures
**Effort**: M (1 hour)

**Implementation** (reader3.py):
```python
"""
Parses an EPUB file into a structured object that can be used to serve the book via a web interface.
"""

from dataclasses import dataclass, field
from typing import List, Dict, Optional
from datetime import datetime

@dataclass
class ChapterContent:
    """
    Represents a physical file in the EPUB (Spine Item).
    A single file might contain multiple logical chapters (TOC entries).
    """
    id: str           # Internal ID (e.g., 'item_1')
    href: str         # Filename (e.g., 'part01.html')
    title: str        # Best guess title from file
    content: str      # Cleaned HTML with rewritten image paths
    text: str         # Plain text for search/LLM context
    order: int        # Linear reading order


@dataclass
class TOCEntry:
    """Represents a logical entry in the navigation sidebar."""
    title: str
    href: str         # original href (e.g., 'part01.html#chapter1')
    file_href: str    # just the filename (e.g., 'part01.html')
    anchor: str       # just the anchor (e.g., 'chapter1'), empty if none
    children: List['TOCEntry'] = field(default_factory=list)


@dataclass
class BookMetadata:
    """Metadata"""
    title: str
    language: str
    authors: List[str] = field(default_factory=list)
    description: Optional[str] = None
    publisher: Optional[str] = None
    date: Optional[str] = None
    identifiers: List[str] = field(default_factory=list)
    subjects: List[str] = field(default_factory=list)


@dataclass
class Book:
    """The Master Object to be pickled."""
    metadata: BookMetadata
    spine: List[ChapterContent]  # The actual content (linear files)
    toc: List[TOCEntry]          # The navigation tree
    images: Dict[str, str]       # Map: original_path -> local_path

    # Meta info
    source_file: str
    processed_at: str
    version: str = "3.0"
```

**Acceptance Criteria**:
- [x] All dataclasses defined
- [x] Type hints complete
- [x] Docstrings present
- [x] Recursive TOCEntry.children field works
- [x] Book.version set to "3.0"

**Testing**:
```python
# Smoke test
metadata = BookMetadata(title="Test", language="en")
chapter = ChapterContent(id="ch1", href="c1.html", title="Chapter 1",
                         content="<p>Hello</p>", text="Hello", order=0)
toc = TOCEntry(title="Chapter 1", href="c1.html", file_href="c1.html", anchor="")
book = Book(metadata=metadata, spine=[chapter], toc=[toc], images={},
            source_file="test.epub", processed_at=datetime.now().isoformat())
assert book.version == "3.0"
```

---

## Phase 3: EPUB Processing Layer

**Goal**: Convert EPUB to Book object

### Task 3.1: HTML Cleaning Utilities
**Effort**: M (1 hour)

**Implementation** (reader3.py):
```python
from bs4 import BeautifulSoup, Comment

def clean_html_content(soup: BeautifulSoup) -> BeautifulSoup:
    # Remove dangerous/useless tags
    for tag in soup(['script', 'style', 'iframe', 'video', 'nav', 'form', 'button']):
        tag.decompose()

    # Remove HTML comments
    for comment in soup.find_all(string=lambda text: isinstance(text, Comment)):
        comment.extract()

    # Remove input tags
    for tag in soup.find_all('input'):
        tag.decompose()

    return soup


def extract_plain_text(soup: BeautifulSoup) -> str:
    """Extract clean text for LLM/Search usage."""
    text = soup.get_text(separator=' ')
    # Collapse whitespace
    return ' '.join(text.split())
```

**Acceptance Criteria**:
- [x] Dangerous tags removed (script, iframe, form, etc.)
- [x] HTML comments removed
- [x] Plain text extracted correctly
- [x] Whitespace collapsed

**Test Cases**:
```python
html = """
<html>
  <head><script>alert('xss')</script></head>
  <body>
    <p>Hello   world</p>
    <!-- comment -->
    <form><input type="text"></form>
  </body>
</html>
"""
soup = BeautifulSoup(html, 'html.parser')
cleaned = clean_html_content(soup)
assert cleaned.find('script') is None
assert cleaned.find('form') is None
assert extract_plain_text(cleaned) == "Hello world"
```

**Reference**: `reader3.py:72-93`

---

### Task 3.2: TOC Recursive Parser
**Effort**: L (2 hours)

**Implementation** (reader3.py):
```python
import ebooklib
from ebooklib import epub

def parse_toc_recursive(toc_list, depth=0) -> List[TOCEntry]:
    """
    Recursively parses the TOC structure from ebooklib.
    """
    result = []

    for item in toc_list:
        # ebooklib TOC items are either `Link` objects or tuples (Section, [Children])
        if isinstance(item, tuple):
            section, children = item
            entry = TOCEntry(
                title=section.title,
                href=section.href,
                file_href=section.href.split('#')[0],
                anchor=section.href.split('#')[1] if '#' in section.href else "",
                children=parse_toc_recursive(children, depth + 1)
            )
            result.append(entry)
        elif isinstance(item, epub.Link):
            entry = TOCEntry(
                title=item.title,
                href=item.href,
                file_href=item.href.split('#')[0],
                anchor=item.href.split('#')[1] if '#' in item.href else ""
            )
            result.append(entry)
        elif isinstance(item, epub.Section):
             entry = TOCEntry(
                title=item.title,
                href=item.href,
                file_href=item.href.split('#')[0],
                anchor=item.href.split('#')[1] if '#' in item.href else ""
            )
             result.append(entry)

    return result


def get_fallback_toc(book_obj) -> List[TOCEntry]:
    """
    If TOC is missing, build a flat one from the Spine.
    """
    toc = []
    for item in book_obj.get_items():
        if item.get_type() == ebooklib.ITEM_DOCUMENT:
            name = item.get_name()
            title = item.get_name().replace('.html', '').replace('.xhtml', '').replace('_', ' ').title()
            toc.append(TOCEntry(title=title, href=name, file_href=name, anchor=""))
    return toc
```

**Acceptance Criteria**:
- [x] Handles tuple (Section, children) format
- [x] Handles epub.Link format
- [x] Handles epub.Section format
- [x] Recursive children parsed correctly
- [x] Anchor and filename separated
- [x] Fallback TOC works for TOC-less EPUBs

**Test Cases**:
- Test with nested TOC (3 levels deep)
- Test with flat TOC (no children)
- Test with empty TOC (should trigger fallback)
- Test with anchors (e.g., "chapter1.html#section2")

**Reference**: `reader3.py:96-146`

---

### Task 3.3: Metadata Extraction
**Effort**: M (1 hour)

**Implementation** (reader3.py):
```python
def extract_metadata_robust(book_obj) -> BookMetadata:
    """
    Extracts metadata handling both single and list values.
    """
    def get_list(key):
        data = book_obj.get_metadata('DC', key)
        return [x[0] for x in data] if data else []

    def get_one(key):
        data = book_obj.get_metadata('DC', key)
        return data[0][0] if data else None

    return BookMetadata(
        title=get_one('title') or "Untitled",
        language=get_one('language') or "en",
        authors=get_list('creator'),
        description=get_one('description'),
        publisher=get_one('publisher'),
        date=get_one('date'),
        identifiers=get_list('identifier'),
        subjects=get_list('subject')
    )
```

**Acceptance Criteria**:
- [x] Handles missing metadata gracefully (defaults)
- [x] Handles single-value fields (title, language)
- [x] Handles multi-value fields (authors, subjects)
- [x] Returns "Untitled" if no title

**Test Cases**:
- Test with complete metadata
- Test with partial metadata (missing author)
- Test with no metadata (all defaults)

**Reference**: `reader3.py:149-170`

---

### Task 3.4: Image Extraction & Path Mapping
**Effort**: L (2 hours)

**Implementation** (reader3.py):
```python
import os
import shutil
from urllib.parse import unquote

def extract_images(book_obj, output_dir: str) -> Dict[str, str]:
    """
    Extract images and build path map.
    Returns: {internal_path: local_relative_path}
    """
    images_dir = os.path.join(output_dir, 'images')
    os.makedirs(images_dir, exist_ok=True)

    image_map = {}

    for item in book_obj.get_items():
        if item.get_type() == ebooklib.ITEM_IMAGE:
            # Normalize filename
            original_fname = os.path.basename(item.get_name())

            # Sanitize filename for OS
            safe_fname = "".join([c for c in original_fname if c.isalpha() or c.isdigit() or c in '._-']).strip()

            # Save to disk
            local_path = os.path.join(images_dir, safe_fname)
            with open(local_path, 'wb') as f:
                f.write(item.get_content())

            # Map keys: both full path and basename
            rel_path = f"images/{safe_fname}"
            image_map[item.get_name()] = rel_path
            image_map[original_fname] = rel_path

    return image_map


def rewrite_image_paths(soup: BeautifulSoup, image_map: Dict[str, str]) -> BeautifulSoup:
    """Rewrite <img src> to local paths."""
    for img in soup.find_all('img'):
        src = img.get('src', '')
        if not src:
            continue

        # Decode URL (part01/image%201.jpg -> part01/image 1.jpg)
        src_decoded = unquote(src)
        filename = os.path.basename(src_decoded)

        # Try to find in map
        if src_decoded in image_map:
            img['src'] = image_map[src_decoded]
        elif filename in image_map:
            img['src'] = image_map[filename]

    return soup
```

**Acceptance Criteria**:
- [x] All images extracted to `images/` folder
- [x] Filenames sanitized (alphanumeric + ._- only)
- [x] Dual-key mapping (full path + basename)
- [x] URL-encoded paths decoded correctly
- [x] Missing images handled gracefully (no crash)

**Test Cases**:
- Test with URL-encoded filenames (e.g., "image%201.jpg")
- Test with nested paths (e.g., "images/chapter1/pic.jpg")
- Test with special characters in filenames
- Test with missing image references (broken src)

**Reference**: `reader3.py:191-210`, `reader3.py:237-249`

---

### Task 3.5: Main EPUB Processing Pipeline
**Effort**: L (2-3 hours)

**Implementation** (reader3.py):
```python
def process_epub(epub_path: str, output_dir: str) -> Book:

    # 1. Load Book
    print(f"Loading {epub_path}...")
    book = epub.read_epub(epub_path)

    # 2. Extract Metadata
    metadata = extract_metadata_robust(book)

    # 3. Prepare Output Directories
    if os.path.exists(output_dir):
        shutil.rmtree(output_dir)
    os.makedirs(output_dir, exist_ok=True)

    # 4. Extract Images & Build Map
    print("Extracting images...")
    image_map = extract_images(book, output_dir)

    # 5. Process TOC
    print("Parsing Table of Contents...")
    toc_structure = parse_toc_recursive(book.toc)
    if not toc_structure:
        print("Warning: Empty TOC, building fallback from Spine...")
        toc_structure = get_fallback_toc(book)

    # 6. Process Content (Spine-based)
    print("Processing chapters...")
    spine_chapters = []

    for i, spine_item in enumerate(book.spine):
        item_id, linear = spine_item
        item = book.get_item_with_id(item_id)

        if not item:
            continue

        if item.get_type() == ebooklib.ITEM_DOCUMENT:
            # Raw content
            raw_content = item.get_content().decode('utf-8', errors='ignore')
            soup = BeautifulSoup(raw_content, 'html.parser')

            # A. Fix Images
            soup = rewrite_image_paths(soup, image_map)

            # B. Clean HTML
            soup = clean_html_content(soup)

            # C. Extract Body Content only
            body = soup.find('body')
            if body:
                final_html = "".join([str(x) for x in body.contents])
            else:
                final_html = str(soup)

            # D. Create Object
            chapter = ChapterContent(
                id=item_id,
                href=item.get_name(),
                title=f"Section {i+1}",
                content=final_html,
                text=extract_plain_text(soup),
                order=i
            )
            spine_chapters.append(chapter)

    # 7. Final Assembly
    final_book = Book(
        metadata=metadata,
        spine=spine_chapters,
        toc=toc_structure,
        images=image_map,
        source_file=os.path.basename(epub_path),
        processed_at=datetime.now().isoformat()
    )

    return final_book
```

**Acceptance Criteria**:
- [x] EPUB loaded successfully
- [x] Metadata extracted
- [x] Output directory created (overwrites if exists)
- [x] Images extracted
- [x] TOC parsed (with fallback)
- [x] All spine items processed
- [x] HTML cleaned and image paths rewritten
- [x] Plain text extracted
- [x] Book object assembled
- [x] Processing progress logged to console

**Test Cases**:
- Test with well-formed EPUB3
- Test with EPUB2
- Test with missing TOC
- Test with missing metadata
- Test with large EPUB (100+ chapters)

**Reference**: `reader3.py:175-283`

---

### Task 3.6: Pickle Persistence
**Effort**: S (30 min)

**Implementation** (reader3.py):
```python
import pickle

def save_to_pickle(book: Book, output_dir: str):
    p_path = os.path.join(output_dir, 'book.pkl')
    with open(p_path, 'wb') as f:
        pickle.dump(book, f)
    print(f"Saved structured data to {p_path}")
```

**Acceptance Criteria**:
- [x] book.pkl created in output_dir
- [x] File is loadable with pickle.load()
- [x] All data structures intact after unpickling

**Test**:
```python
save_to_pickle(book, "test_data")
with open("test_data/book.pkl", "rb") as f:
    loaded = pickle.load(f)
assert loaded.metadata.title == book.metadata.title
assert len(loaded.spine) == len(book.spine)
```

**Reference**: `reader3.py:286-290`

---

### Task 3.7: CLI Interface
**Effort**: S (30 min)

**Implementation** (reader3.py):
```python
if __name__ == "__main__":
    import sys
    if len(sys.argv) < 2:
        print("Usage: python reader3.py <file.epub>")
        sys.exit(1)

    epub_file = sys.argv[1]
    assert os.path.exists(epub_file), "File not found."
    out_dir = os.path.splitext(epub_file)[0] + "_data"

    book_obj = process_epub(epub_file, out_dir)
    save_to_pickle(book_obj, out_dir)

    print("\n--- Summary ---")
    print(f"Title: {book_obj.metadata.title}")
    print(f"Authors: {', '.join(book_obj.metadata.authors)}")
    print(f"Physical Files (Spine): {len(book_obj.spine)}")
    print(f"TOC Root Items: {len(book_obj.toc)}")
    print(f"Images extracted: {len(book_obj.images)}")
```

**Acceptance Criteria**:
- [x] Usage message for no args
- [x] File existence check
- [x] Output directory auto-named (e.g., "dracula.epub" → "dracula_data")
- [x] Summary printed after processing
- [x] Non-zero exit on error

**Test**:
```bash
uv run reader3.py dracula.epub
# Should create dracula_data/ with book.pkl and images/
```

**Reference**: `reader3.py:295-314`

---

## Phase 4: Web Server Layer

**Goal**: Serve processed books via FastAPI

### Task 4.1: FastAPI App Setup
**Effort**: S (30 min)

**Implementation** (server.py):
```python
import os
import pickle
from functools import lru_cache
from typing import Optional

from fastapi import FastAPI, Request, HTTPException
from fastapi.responses import HTMLResponse, FileResponse
from fastapi.templating import Jinja2Templates

from reader3 import Book

app = FastAPI()
templates = Jinja2Templates(directory="templates")

BOOKS_DIR = "."
```

**Acceptance Criteria**:
- [x] FastAPI app created
- [x] Jinja2 templates configured
- [x] BOOKS_DIR constant defined
- [x] Book import from reader3.py works

**Test**:
```bash
uv run uvicorn server:app --reload
# Should start without errors
```

**Reference**: `server.py:1-17`

---

### Task 4.2: Book Loading with Cache
**Effort**: M (1 hour)

**Implementation** (server.py):
```python
@lru_cache(maxsize=10)
def load_book_cached(folder_name: str) -> Optional[Book]:
    """
    Loads the book from the pickle file.
    Cached so we don't re-read the disk on every click.
    """
    file_path = os.path.join(BOOKS_DIR, folder_name, "book.pkl")
    if not os.path.exists(file_path):
        return None

    try:
        with open(file_path, "rb") as f:
            book = pickle.load(f)
        return book
    except Exception as e:
        print(f"Error loading book {folder_name}: {e}")
        return None
```

**Acceptance Criteria**:
- [x] LRU cache with maxsize=10
- [x] Returns None for missing books
- [x] Returns None for corrupted pickles (with error log)
- [x] Cache invalidates when server restarts
- [x] Thread-safe (important for async)

**Test**:
```python
# First call: cache miss
book1 = load_book_cached("dracula_data")
# Second call: cache hit (fast)
book2 = load_book_cached("dracula_data")
assert book1 is book2  # Same object
```

**Reference**: `server.py:19-35`

---

### Task 4.3: Library View Route
**Effort**: M (1 hour)

**Implementation** (server.py):
```python
@app.get("/", response_class=HTMLResponse)
async def library_view(request: Request):
    """Lists all available processed books."""
    books = []

    # Scan directory for folders ending in '_data' that have a book.pkl
    if os.path.exists(BOOKS_DIR):
        for item in os.listdir(BOOKS_DIR):
            if item.endswith("_data") and os.path.isdir(item):
                # Try to load it to get the title
                book = load_book_cached(item)
                if book:
                    books.append({
                        "id": item,
                        "title": book.metadata.title,
                        "author": ", ".join(book.metadata.authors),
                        "chapters": len(book.spine)
                    })

    return templates.TemplateResponse("library.html", {"request": request, "books": books})
```

**Acceptance Criteria**:
- [x] Scans for *_data folders
- [x] Loads book.pkl from each folder
- [x] Extracts title, author, chapter count
- [x] Ignores corrupted books (load_book_cached returns None)
- [x] Returns HTML response
- [x] Empty library handled gracefully

**Test**:
```bash
curl http://127.0.0.1:8123/
# Should return HTML with book grid
```

**Reference**: `server.py:37-56`

---

### Task 4.4: Chapter Reader Routes
**Effort**: L (2 hours)

**Implementation** (server.py):
```python
@app.get("/read/{book_id}", response_class=HTMLResponse)
async def redirect_to_first_chapter(book_id: str):
    """Helper to just go to chapter 0."""
    return await read_chapter(book_id=book_id, chapter_index=0)


@app.get("/read/{book_id}/{chapter_index}", response_class=HTMLResponse)
async def read_chapter(request: Request, book_id: str, chapter_index: int):
    """The main reader interface."""
    book = load_book_cached(book_id)
    if not book:
        raise HTTPException(status_code=404, detail="Book not found")

    if chapter_index < 0 or chapter_index >= len(book.spine):
        raise HTTPException(status_code=404, detail="Chapter not found")

    current_chapter = book.spine[chapter_index]

    # Calculate Prev/Next links
    prev_idx = chapter_index - 1 if chapter_index > 0 else None
    next_idx = chapter_index + 1 if chapter_index < len(book.spine) - 1 else None

    return templates.TemplateResponse("reader.html", {
        "request": request,
        "book": book,
        "current_chapter": current_chapter,
        "chapter_index": chapter_index,
        "book_id": book_id,
        "prev_idx": prev_idx,
        "next_idx": next_idx
    })
```

**Acceptance Criteria**:
- [x] `/read/{book_id}` redirects to first chapter
- [x] `/read/{book_id}/{chapter_index}` loads chapter
- [x] 404 for missing book
- [x] 404 for out-of-range chapter
- [x] Prev/Next indices calculated correctly
- [x] Prev/Next disabled at boundaries (None)
- [x] All data passed to template

**Test Cases**:
```bash
# First chapter
curl http://127.0.0.1:8123/read/dracula_data/0

# Last chapter
curl http://127.0.0.1:8123/read/dracula_data/50

# Invalid chapter
curl http://127.0.0.1:8123/read/dracula_data/9999  # 404

# Missing book
curl http://127.0.0.1:8123/read/nonexistent_data/0  # 404
```

**Reference**: `server.py:58-87`

---

### Task 4.5: Image Serving Route
**Effort**: M (1 hour)

**Implementation** (server.py):
```python
@app.get("/read/{book_id}/images/{image_name}")
async def serve_image(book_id: str, image_name: str):
    """
    Serves images specifically for a book.
    The HTML contains <img src="images/pic.jpg">.
    The browser resolves this to /read/{book_id}/images/pic.jpg.
    """
    # Security check: ensure book_id is clean
    safe_book_id = os.path.basename(book_id)
    safe_image_name = os.path.basename(image_name)

    img_path = os.path.join(BOOKS_DIR, safe_book_id, "images", safe_image_name)

    if not os.path.exists(img_path):
        raise HTTPException(status_code=404, detail="Image not found")

    return FileResponse(img_path)
```

**Acceptance Criteria**:
- [x] Images served from correct path
- [x] Path traversal prevented (os.path.basename)
- [x] 404 for missing images
- [x] Correct MIME type (automatic via FileResponse)
- [x] Efficient streaming (FileResponse handles this)

**Security Test**:
```bash
# Should fail (path traversal attempt)
curl http://127.0.0.1:8123/read/../../etc/images/passwd  # 404
```

**Reference**: `server.py:89-105`

---

### Task 4.6: Server Entry Point
**Effort**: S (15 min)

**Implementation** (server.py):
```python
if __name__ == "__main__":
    import uvicorn
    print("Starting server at http://127.0.0.1:8123")
    uvicorn.run(app, host="127.0.0.1", port=8123)
```

**Acceptance Criteria**:
- [x] Server starts on 127.0.0.1:8123
- [x] Startup message printed
- [x] Uvicorn runs in development mode (auto-reload would be nice but not required)

**Test**:
```bash
uv run server.py
# Visit http://127.0.0.1:8123 in browser
```

**Reference**: `server.py:107-110`

---

## Phase 5: Presentation Layer

**Goal**: Create HTML templates for library and reader

### Task 5.1: Library Template
**Effort**: M (1 hour)

**Implementation** (templates/library.html):
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>My Library</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
            background: #f4f4f9;
            margin: 0;
            padding: 40px;
        }
        .container { max-width: 800px; margin: 0 auto; }
        h1 { color: #333; border-bottom: 2px solid #ddd; padding-bottom: 10px; }
        .book-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
            gap: 20px;
            margin-top: 30px;
        }
        .book-card {
            background: white;
            padding: 20px;
            border-radius: 8px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            transition: transform 0.2s;
        }
        .book-card:hover { transform: translateY(-2px); }
        .book-title { font-size: 1.2em; font-weight: bold; color: #2c3e50; margin-bottom: 10px; }
        .book-meta { color: #666; font-size: 0.9em; margin-bottom: 15px; }
        .btn {
            display: inline-block;
            background: #3498db;
            color: white;
            text-decoration: none;
            padding: 8px 15px;
            border-radius: 4px;
            font-size: 0.9em;
        }
        .btn:hover { background: #2980b9; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Library</h1>

        {% if not books %}
            <p>No processed books found. Run <code>reader3.py</code> on an epub first.</p>
        {% endif %}

        <div class="book-grid">
            {% for book in books %}
            <div class="book-card">
                <div class="book-title">{{ book.title }}</div>
                <div class="book-meta">
                    {{ book.author }}<br>
                    {{ book.chapters }} sections
                </div>
                <a href="/read/{{ book.id }}/0" class="btn">Read Book</a>
            </div>
            {% endfor %}
        </div>
    </div>
</body>
</html>
```

**Acceptance Criteria**:
- [x] Responsive CSS grid layout
- [x] Book cards with title, author, chapter count
- [x] "Read Book" button links to first chapter
- [x] Empty state message
- [x] Mobile-friendly (viewport meta tag)
- [x] Hover effects on cards

**Design**:
- Background: #f4f4f9 (light gray)
- Card: white with shadow
- Button: #3498db (blue)
- Grid: responsive, min 250px columns

**Reference**: `templates/library.html`

---

### Task 5.2: Reader Template - Base Layout
**Effort**: L (2 hours)

**Implementation** (templates/reader.html):
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>{{ book.metadata.title }}</title>
    <style>
        /* Layout */
        body { margin: 0; padding: 0; display: flex; height: 100vh; overflow: hidden; font-family: "Georgia", serif; background: #fff; }

        /* Sidebar */
        #sidebar {
            width: 300px;
            background: #f8f9fa;
            border-right: 1px solid #e9ecef;
            overflow-y: auto;
            padding: 20px;
            flex-shrink: 0;
        }
        .nav-header {
            font-family: -apple-system, sans-serif;
            font-weight: bold;
            color: #495057;
            margin-bottom: 15px;
            padding-bottom: 10px;
            border-bottom: 1px solid #dee2e6;
        }
        .nav-home {
            display: block;
            margin-bottom: 20px;
            color: #3498db;
            text-decoration: none;
            font-family: -apple-system, sans-serif;
            font-size: 0.9em;
        }

        /* Main Content */
        #main {
            flex-grow: 1;
            overflow-y: auto;
            position: relative;
            scroll-behavior: smooth;
        }
        .content-container {
            max-width: 700px;
            margin: 0 auto;
            padding: 60px 40px;
            line-height: 1.8;
            font-size: 1.15em;
            color: #212529;
        }

        /* Content Styling */
        .book-content img { max-width: 100%; height: auto; display: block; margin: 20px auto; }
        .book-content h1, .book-content h2, .book-content h3 {
            font-family: -apple-system, sans-serif;
            margin-top: 1.5em;
            color: #333;
        }
        .book-content p { margin-bottom: 1.5em; text-align: justify; }

        /* Navigation Footer */
        .chapter-nav {
            display: flex;
            justify-content: space-between;
            margin-top: 60px;
            padding-top: 20px;
            border-top: 1px solid #eee;
            font-family: -apple-system, sans-serif;
        }
        .nav-btn {
            text-decoration: none;
            color: #3498db;
            font-weight: bold;
            padding: 10px 20px;
            border: 1px solid #3498db;
            border-radius: 4px;
            transition: all 0.2s;
        }
        .nav-btn:hover { background: #3498db; color: white; }
        .nav-btn.disabled { opacity: 0.5; pointer-events: none; border-color: #ccc; color: #ccc; }
    </style>
</head>
<body>
    <!-- Sidebar and main content in next task -->
</body>
</html>
```

**Acceptance Criteria**:
- [x] Two-column flexbox layout (sidebar + main)
- [x] Sidebar fixed width (300px)
- [x] Main content scrollable
- [x] Content max-width 700px (optimal readability)
- [x] Typography: Georgia serif for content, system fonts for UI
- [x] Responsive (viewport meta tag)

**Reference**: `templates/reader.html:1-39`

---

### Task 5.3: Reader Template - TOC Sidebar
**Effort**: L (2 hours)

**Implementation** (templates/reader.html, inside `<body>`):
```html
<div id="sidebar">
    <a href="/" class="nav-home">← Back to Library</a>
    <div class="nav-header">{{ book.metadata.title }}</div>

    <!-- Recursive Macro for TOC -->
    {% macro render_toc(items) %}
        <ul class="toc-list">
        {% for item in items %}
            <li class="toc-item">
                {% set is_active = current_chapter.href == item.file_href %}

                <a href="#" onclick="findAndGo('{{ item.file_href }}')"
                   class="toc-link {% if is_active %}active{% endif %}">
                    {{ item.title }}
                </a>

                {% if item.children %}
                    {{ render_toc(item.children) }}
                {% endif %}
            </li>
        {% endfor %}
        </ul>
    {% endmacro %}

    {{ render_toc(book.toc) }}
</div>
```

**CSS for TOC** (add to `<style>`):
```css
/* TOC Tree */
ul.toc-list { list-style: none; padding-left: 0; margin: 0; }
ul.toc-list ul { padding-left: 20px; } /* Indent children */
li.toc-item { margin-bottom: 8px; }
a.toc-link {
    text-decoration: none;
    color: #495057;
    font-size: 0.95em;
    display: block;
    padding: 4px 0;
    line-height: 1.4;
}
a.toc-link:hover { color: #000; text-decoration: underline; }
a.toc-link.active { color: #d63384; font-weight: bold; }
```

**Acceptance Criteria**:
- [x] Recursive macro renders nested TOC
- [x] Active chapter highlighted
- [x] Clicking TOC item navigates (JavaScript in next task)
- [x] Indentation for nested items (20px per level)
- [x] "Back to Library" link at top

**Reference**: `templates/reader.html:44-94`

---

### Task 5.4: Reader Template - Main Content & Navigation
**Effort**: M (1 hour)

**Implementation** (templates/reader.html, after sidebar):
```html
<div id="main">
    <div class="content-container">
        <div class="book-content">
            {{ current_chapter.content | safe }}
        </div>

        <div class="chapter-nav">
            {% if prev_idx is not none %}
                <a href="/read/{{ book_id }}/{{ prev_idx }}" class="nav-btn">← Previous</a>
            {% else %}
                <span class="nav-btn disabled">← Previous</span>
            {% endif %}

            <span style="color: #999; padding: 10px;">
                Section {{ chapter_index + 1 }} of {{ book.spine|length }}
            </span>

            {% if next_idx is not none %}
                <a href="/read/{{ book_id }}/{{ next_idx }}" class="nav-btn">Next →</a>
            {% else %}
                <span class="nav-btn disabled">Next →</span>
            {% endif %}
        </div>
    </div>
</div>
```

**Acceptance Criteria**:
- [x] Chapter HTML rendered (with `| safe` filter)
- [x] Prev/Next buttons
- [x] Disabled state for buttons at boundaries
- [x] Chapter counter (e.g., "Section 5 of 50")
- [x] Footer navigation below content

**Reference**: `templates/reader.html:98-121`

---

### Task 5.5: Reader Template - JavaScript TOC Navigation
**Effort**: M (1 hour)

**Implementation** (templates/reader.html, before `</body>`):
```html
<script>
    // Helper to map TOC filenames to Spine Indices
    const spineMap = {
        {% for ch in book.spine %}
        "{{ ch.href }}": {{ ch.order }},
        {% endfor %}
    };

    function findAndGo(filename) {
        // The TOC usually has specific filenames e.g. "text/part001.html"
        // Sometimes it has anchors "text/part001.html#header"
        const cleanFile = filename.split('#')[0];
        const anchor = filename.split('#')[1];

        const idx = spineMap[cleanFile];

        if (idx !== undefined) {
            let url = "/read/{{ book_id }}/" + idx;
            window.location.href = url;
        } else {
            console.log("Could not find index for", filename);
        }
    }
</script>
```

**Acceptance Criteria**:
- [x] spineMap generated from book.spine
- [x] findAndGo() function parses filename
- [x] Anchor stripped from filename
- [x] Navigates to correct chapter index
- [x] Logs error if filename not found (edge case)

**Test**:
- Click TOC items
- Verify navigation to correct chapters
- Check browser console for errors

**Reference**: `templates/reader.html:124-151`

---

## Phase 6: Testing & Quality

**Goal**: Ensure system works correctly

### Task 6.1: Manual Integration Test
**Effort**: M (1 hour)

**Test Procedure**:
1. Download sample EPUB (e.g., Dracula from Project Gutenberg)
2. Process EPUB: `uv run reader3.py dracula.epub`
3. Verify output:
   - `dracula_data/` folder exists
   - `dracula_data/book.pkl` exists
   - `dracula_data/images/` contains images
4. Start server: `uv run server.py`
5. Open browser to `http://127.0.0.1:8123`
6. Verify library:
   - Dracula appears in grid
   - Title, author, chapter count correct
7. Click "Read Book"
8. Verify reader:
   - Chapter 1 content displays
   - Images render
   - TOC shows in sidebar
   - Active chapter highlighted
9. Test navigation:
   - Click "Next" → Chapter 2 loads
   - Click "Previous" → Chapter 1 loads
   - Click TOC item → Correct chapter loads
10. Test edge cases:
    - Navigate to last chapter → "Next" disabled
    - Navigate to first chapter → "Previous" disabled
    - Visit invalid URL `/read/dracula_data/999` → 404
    - Visit invalid book `/read/nonexistent/0` → 404

**Acceptance Criteria**:
- [ ] All test steps pass
- [ ] No errors in browser console
- [ ] No errors in server logs
- [ ] Images display correctly
- [ ] Navigation works smoothly

---

### Task 6.2: Unit Tests (Optional but Recommended)
**Effort**: L (3-4 hours)

**Setup**:
```bash
# Add pytest to dependencies
uv add --dev pytest pytest-cov

# Create test file
mkdir tests
touch tests/test_reader3.py
```

**Test Cases** (tests/test_reader3.py):
```python
import pytest
from reader3 import (
    ChapterContent, TOCEntry, BookMetadata, Book,
    clean_html_content, extract_plain_text,
    parse_toc_recursive
)
from bs4 import BeautifulSoup

def test_clean_html_removes_dangerous_tags():
    html = '<html><head><script>alert("xss")</script></head><body><p>Safe</p></body></html>'
    soup = BeautifulSoup(html, 'html.parser')
    cleaned = clean_html_content(soup)
    assert cleaned.find('script') is None
    assert cleaned.find('p') is not None

def test_extract_plain_text():
    html = '<p>Hello   world</p><p>Another  paragraph</p>'
    soup = BeautifulSoup(html, 'html.parser')
    text = extract_plain_text(soup)
    assert text == "Hello world Another paragraph"

def test_toc_entry_with_anchor():
    entry = TOCEntry(
        title="Chapter 1",
        href="chapter1.html#section2",
        file_href="chapter1.html",
        anchor="section2"
    )
    assert entry.file_href == "chapter1.html"
    assert entry.anchor == "section2"

def test_book_version():
    book = Book(
        metadata=BookMetadata(title="Test", language="en"),
        spine=[],
        toc=[],
        images={},
        source_file="test.epub",
        processed_at="2025-01-01T00:00:00"
    )
    assert book.version == "3.0"

# Add more tests...
```

**Run tests**:
```bash
uv run pytest tests/ -v --cov=reader3 --cov=server
```

**Acceptance Criteria**:
- [ ] 80%+ code coverage
- [ ] All critical paths tested
- [ ] Edge cases covered

---

### Task 6.3: Performance Testing (Optional)
**Effort**: M (1-2 hours)

**Test Scenarios**:
1. **Large EPUB Processing**
   - Process EPUB with 100+ chapters
   - Measure time, memory usage
   - Target: < 60 seconds, < 500MB RAM

2. **Cold Start Server**
   - Restart server
   - Load library page
   - Measure time
   - Target: < 1 second

3. **Cached Chapter Load**
   - Load same chapter 10 times
   - Measure p50, p95, p99 latency
   - Target: p95 < 100ms

4. **Cache Pressure**
   - Load 15 different books (exceeds cache size of 10)
   - Verify LRU eviction works
   - No memory leaks

**Tools**:
- `time` command for processing
- `ab` (Apache Bench) for load testing
- `memory_profiler` for memory analysis

---

## Phase 7: Documentation & Deployment

**Goal**: Prepare for users

### Task 7.1: Write Comprehensive README
**Effort**: M (1 hour)

**Sections**:
1. Project description
2. Installation (uv setup)
3. Usage (processing, serving)
4. Features
5. Limitations
6. License
7. Contributing (or "not accepting contributions" per author's note)

**Reference**: `README.md` (existing)

---

### Task 7.2: Add .gitignore
**Effort**: S (10 min)

**Contents**:
```gitignore
# Python
__pycache__/
*.py[cod]
*$py.class
*.so
.Python
build/
dist/
*.egg-info/

# Virtual environments
.venv/
venv/
ENV/

# Processed books
*_data/

# Editor
.vscode/
.idea/
*.swp

# OS
.DS_Store
Thumbs.db
```

**Acceptance Criteria**:
- [ ] Processed books excluded (*_data/)
- [ ] Python artifacts excluded

---

### Task 7.3: Create Sample EPUB for Testing
**Effort**: S (20 min)

**Download**:
- Dracula (EPUB3): https://www.gutenberg.org/ebooks/345
- Save as `sample_dracula.epub`

**Process**:
```bash
uv run reader3.py sample_dracula.epub
```

**Verify**:
- `sample_dracula_data/` created
- Ready for demo

---

## Summary Checklist

### Phase 1: Project Setup ✓
- [x] Task 1.1: Initialize Python project
- [x] Task 1.2: Create directory structure
- [x] Task 1.3: Write README

### Phase 2: Data Model ✓
- [x] Task 2.1: Define data structures

### Phase 3: EPUB Processing ✓
- [x] Task 3.1: HTML cleaning utilities
- [x] Task 3.2: TOC recursive parser
- [x] Task 3.3: Metadata extraction
- [x] Task 3.4: Image extraction & path mapping
- [x] Task 3.5: Main EPUB processing pipeline
- [x] Task 3.6: Pickle persistence
- [x] Task 3.7: CLI interface

### Phase 4: Web Server ✓
- [x] Task 4.1: FastAPI app setup
- [x] Task 4.2: Book loading with cache
- [x] Task 4.3: Library view route
- [x] Task 4.4: Chapter reader routes
- [x] Task 4.5: Image serving route
- [x] Task 4.6: Server entry point

### Phase 5: Presentation ✓
- [x] Task 5.1: Library template
- [x] Task 5.2: Reader template - base layout
- [x] Task 5.3: Reader template - TOC sidebar
- [x] Task 5.4: Reader template - main content
- [x] Task 5.5: Reader template - JavaScript navigation

### Phase 6: Testing (Optional)
- [ ] Task 6.1: Manual integration test
- [ ] Task 6.2: Unit tests
- [ ] Task 6.3: Performance testing

### Phase 7: Documentation
- [x] Task 7.1: Comprehensive README
- [ ] Task 7.2: Add .gitignore
- [ ] Task 7.3: Sample EPUB for testing

---

## Estimated Timeline

| Phase | Tasks | Effort | Time |
|-------|-------|--------|------|
| 1. Setup | 3 | S+S+S | 1 hour |
| 2. Data Model | 1 | M | 1 hour |
| 3. EPUB Processing | 7 | S+M+M+L+L+S+S | 8 hours |
| 4. Web Server | 6 | S+M+M+L+M+S | 6 hours |
| 5. Presentation | 5 | M+L+L+M+M | 7 hours |
| 6. Testing | 3 | M+L+M | 6 hours (optional) |
| 7. Documentation | 3 | M+S+S | 1.5 hours |
| **Total** | **28 tasks** | **Mixed** | **24-30 hours** |

**Core Implementation (without tests)**: ~18-24 hours
**With Tests**: ~24-30 hours

---

## Development Tips

1. **Start with data models** - Get the structure right before processing
2. **Test with real EPUBs early** - Download Dracula after Task 3.5
3. **Use print statements liberally** - Processing can fail silently
4. **Cache invalidation** - Restart server after re-processing books
5. **Browser DevTools** - Check console for JavaScript errors
6. **LLM assistance** - This codebase is simple enough for LLMs to understand and modify

---

## Extension Ideas (Post-MVP)

After completing core tasks, consider:

- **Search**: Add SQLite FTS5 full-text search
- **Bookmarks**: Save reading position (localStorage or DB)
- **Dark Mode**: CSS toggle for night reading
- **Font Controls**: Adjustable font size, family
- **OPDS Support**: Integrate with Calibre
- **Docker Deployment**: Containerize for easy deployment
- **Multi-user**: Add authentication, per-user libraries
- **Format Support**: PDF, MOBI, AZW3 conversion

Remember: "Code is ephemeral now and libraries are over, ask your LLM to change it in whatever way you like."
