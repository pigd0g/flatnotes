# Plan: Sub-Directory Support for flatnotes

## Problem

flatnotes only indexes `.md` files at the top level of the data directory (`FLATNOTES_PATH`). It uses a flat `glob("*")` pattern and builds all file paths as `storage_path/title.md`. Notes stored in subdirectories are invisible to the app.

This is a fundamental limitation for users with folder-organised vaults (e.g. Obsidian).

## Design Decision: Path-as-Title

The simplest and most backwards-compatible approach is to treat the **relative path** (without extension) as the note's identity ("title"). For example:

| File on disk | Title (identity) |
|---|---|
| `Project Alpha/Notes.md` | `Project Alpha/Notes` |
| `Daily/2026-04-19.md` | `Daily/2026-04-19` |
| `Readme.md` | `Readme` |

This means:
- The `title` field becomes a relative path (e.g. `Project Alpha/Notes`)
- The URL becomes `/note/Project%20Alpha%2FNotes`
- Wikilinks like `[[Project Alpha/Notes]]` work naturally
- Top-level notes remain unchanged (backwards compatible)
- The `/` character is now valid in titles (currently rejected by `is_valid_filename`)

**Alternative considered:** Adding a separate `directory` or `path` field. Rejected because it would require API changes, schema changes, and UI changes in many more places. Path-as-title keeps the API surface identical.

---

## Changes Required

### 1. Server: `server/helpers.py` — Allow `/` in filenames

**Current:** `is_valid_filename()` rejects `<>:"/\|?*`

**Change:** Remove `/` from the invalid characters list. The `/` character is the path separator and must be allowed.

```python
# Before
invalid_chars = r'<>:"/\|?*'

# After
invalid_chars = r'<>:"\|?*'
```

`\` remains invalid (Windows path separator, not needed). `:` and other OS-reserved chars stay blocked.

---

### 2. Server: `server/notes/file_system/file_system.py` — Core recursion logic

This is the main file. Multiple methods need updating:

#### 2a. `_list_all_note_filenames()` — Recursive glob

**Current:** `glob.glob(os.path.join(self.storage_path, "*" + MARKDOWN_EXT))`

**Change:** Use `glob.glob(os.path.join(self.storage_path, "**" + os.sep + "*" + MARKDOWN_EXT), recursive=True)`

Return **relative paths** instead of just filenames. Currently it does `os.path.split(filepath)[1]` which strips the directory. Change to `os.path.relpath(filepath, self.storage_path)`.

```python
def _list_all_note_filenames(self) -> List[str]:
    """Return a list of all note filenames (relative to storage_path)."""
    return [
        os.path.relpath(filepath, self.storage_path)
        for filepath in glob.glob(
            os.path.join(self.storage_path, "**", "*" + MARKDOWN_EXT),
            recursive=True,
        )
    ]
```

#### 2b. `_path_from_title()` — Handle nested paths

**Current:** `os.path.join(self.storage_path, title + MARKDOWN_EXT)`

**Change:** Same logic, but now `title` can contain `/`, which naturally creates subdirectories in the path. The join works as-is, but we need to ensure parent directories exist on **create**.

```python
def _path_from_title(self, title: str) -> str:
    return os.path.join(self.storage_path, title + MARKDOWN_EXT)
```

No change to this method itself, but callers that create files need to `makedirs` first (see 2c).

#### 2c. `create()` — Create parent directories

**Change:** Before writing, ensure the parent directory exists:

```python
def create(self, data: NoteCreate) -> Note:
    filepath = self._path_from_title(data.title)
    os.makedirs(os.path.dirname(filepath), exist_ok=True)
    self._write_file(filepath, data.content)
    return Note(
        title=data.title,
        content=data.content,
        last_modified=os.path.getmtime(filepath),
    )
```

#### 2d. `update()` — Handle directory changes on rename

**Change:** When renaming, if the new title has a different parent directory, create the target directory before the rename:

```python
def update(self, title: str, data: NoteUpdate) -> Note:
    is_valid_filename(title)
    filepath = self._path_from_title(title)
    if data.new_title is not None:
        new_filepath = self._path_from_title(data.new_title)
        os.makedirs(os.path.dirname(new_filepath), exist_ok=True)
        if filepath != new_filepath and os.path.isfile(new_filepath):
            raise FileExistsError(...)
        os.rename(filepath, new_filepath)
        title = data.new_title
        filepath = new_filepath
    # ... rest unchanged
```

Also consider: after a rename that moves a file out of a directory, the old directory may be empty. Optionally clean up empty directories (nice-to-have, not critical).

#### 2e. `delete()` — Clean up empty parent directories

**Change:** After deleting, walk up and remove empty directories up to (but not including) `storage_path`:

```python
def delete(self, title: str) -> Note:
    is_valid_filename(title)
    filepath = self._path_from_title(title)
    os.remove(filepath)
    # Clean up empty parent directories
    parent = os.path.dirname(filepath)
    while parent != self.storage_path and os.path.isdir(parent):
        if os.listdir(parent):
            break
        os.rmdir(parent)
        parent = os.path.dirname(parent)
```

#### 2f. `_strip_ext()` — No change needed

Already works: `os.path.splitext("Project Alpha/Notes.md")[0]` → `"Project Alpha/Notes"` ✅

#### 2g. `_get_by_filename()` — Now receives relative paths

**Current:** `self.get(self._strip_ext(filename))` where `filename` was just `"Notes.md"`

**After:** `filename` is now `"Project Alpha/Notes.md"`, and `_strip_ext` correctly produces `"Project Alpha/Notes"` ✅

No change needed.

#### 2h. `_add_note_to_index()` — Store relative path as filename

**Current:** `filename=note.title + MARKDOWN_EXT`

**After:** This still works because `note.title` is now `"Project Alpha/Notes"` so `filename` becomes `"Project Alpha/Notes.md"` — a relative path. The Whoosh index stores and matches on this correctly. ✅

No change needed.

#### 2i. `_sync_index()` — `idx_filepath` construction

**Current:** `idx_filepath = os.path.join(self.storage_path, idx_filename)`

**After:** `idx_filename` is now a relative path like `"Project Alpha/Notes.md"`, so `os.path.join` correctly resolves to the full path. ✅

No change needed.

#### 2j. `_search_result_from_hit()` — title extraction

**Current:** `title = self._strip_ext(hit["filename"])`

**After:** `hit["filename"]` is now `"Project Alpha/Notes.md"`, so `_strip_ext` gives `"Project Alpha/Notes"`. The `_path_from_title(title)` call later correctly resolves the full path. ✅

No change needed.

---

### 3. Server: `server/notes/models.py` — No change needed

The `title` field is already a `str`. It will now accept paths with `/`.

---

### 4. Server: `server/main.py` — No change needed

The route `@router.get("/api/notes/{title}")` uses FastAPI's path parameter. By default, `{title}` does NOT match `/` characters. This needs to be changed to a **path parameter**:

**Current:** `@router.get("/api/notes/{title}", ...)`

**Change:** `@router.get("/api/notes/{title:path}", ...)`

The `:path` modifier tells FastAPI to match `/` characters in the parameter. Without it, a request for `/api/notes/Project Alpha/Notes` would 404.

Apply the same change to all routes that use `{title}`:

```python
@router.get("/api/notes/{title:path}", ...)
@router.patch("/api/notes/{title:path}", ...)
@router.delete("/api/notes/{title:path}", ...)
```

The UI route also needs updating:

```python
@router.get("/note/{title:path}", include_in_schema=False)
```

---

### 5. Server: `server/notes/file_system/file_system.py` — Index schema version bump

Because the `filename` field now stores relative paths (which are longer and structurally different), bump the index schema version to force a clean rebuild:

**Change:** `INDEX_SCHEMA_VERSION = "6"`

This ensures old flat-index data gets cleared and rebuilt with the new recursive scan on first boot.

---

### 6. Server: Path traversal security

Now that `/` is allowed in titles, we must prevent path traversal attacks (e.g. `title="../../etc/passwd"`).

**Add a validation helper** in `helpers.py`:

```python
def is_valid_filename(value):
    """Raise ValueError if the declared string contains invalid characters
    or attempts path traversal."""
    invalid_chars = r'<>:"\|?*'
    if any(invalid_char in value for invalid_char in invalid_chars):
        raise ValueError(
            "title cannot include any of the following characters: "
            + invalid_chars
        )
    # Prevent path traversal
    if ".." in value.split(os.sep) and ".." in value.split("/"):
        raise ValueError("title cannot contain path traversal sequences")
    # Prevent absolute paths
    if os.path.isabs(value):
        raise ValueError("title cannot be an absolute path")
    return value
```

Or more robustly: after constructing the full path, verify it's still under `storage_path`:

```python
def _safe_path(self, title: str) -> str:
    filepath = self._path_from_title(title)
    real_storage = os.path.realpath(self.storage_path)
    real_filepath = os.path.realpath(filepath)
    if not real_filepath.startswith(real_storage + os.sep) and real_filepath != real_storage:
        raise ValueError("Path traversal detected")
    return filepath
```

Use `_safe_path()` in `get()`, `update()`, `delete()` instead of raw `_path_from_title()`.

---

### 7. Client: `client/api.js` — URL encoding

**Current:** `api.get(\`api/notes/${encodeURIComponent(title)}\`)`

**After:** `encodeURIComponent` already encodes `/` as `%2F`. FastAPI's `:path` parameter will decode it back. ✅

No change needed.

---

### 8. Client: `client/router.js` — Vue Router path matching

**Current:** `path: "/note/:title"`

**Change:** `path: "/note/:title(.*)"` — Vue Router uses regex patterns. `(.*)` matches any character including `/`.

```javascript
{
  path: "/note/:title(.*)",
  name: "note",
  component: () => import("./views/Note.vue"),
  props: true,
},
```

---

### 9. Client: UI — Display directory context

Currently the note view and search results show only the title. With nested notes, it would be helpful to show the directory path visually. This is optional/polish:

- In search results, show the directory as a subtle prefix or breadcrumb: `📁 Project Alpha / Notes`
- In the note view header, show the full path
- When creating a new note, allow specifying a path or auto-create in current directory context

These are UX enhancements and can be done incrementally. The core functionality works without them.

---

### 10. Client: Wikilink handling

The `extendedAutolinks.js` plugin for the TOAST UI Editor handles `[[wikilinks]]`. Currently, clicking `[[Some Note]]` navigates to `/note/Some Note`. With subdirectory support:

- `[[Some Note]]` should still resolve to top-level `Some Note` (backwards compatible)
- `[[Project Alpha/Notes]]` should navigate to `/note/Project%20Alpha%2FNotes` — this works if the router is updated as per #8

No change needed to the wikilink plugin if the router supports path segments.

---

### 11. Attachments: No change needed

Attachments are stored in `storage_path/attachments/` — a flat directory. They're not affected by note directory structure. Attachment relative URLs in markdown remain `attachments/filename.jpg`.

However, for notes in subdirectories, the relative URL won't resolve correctly in external markdown viewers (the browser would look for `Project Alpha/attachments/filename.jpg`). This is a known limitation. A fix would be to use absolute URLs or rewrite attachment paths, but this is out of scope for the initial implementation.

---

## Summary of Required Changes

| # | File | Change | Effort |
|---|---|---|---|
| 1 | `server/helpers.py` | Remove `/` from invalid chars, add path traversal check | Small |
| 2 | `server/notes/file_system/file_system.py` | Recursive glob, makedirs on create, path safety, empty dir cleanup, schema version bump | Medium |
| 3 | `server/main.py` | `{title}` → `{title:path}` on 4 routes | Small |
| 4 | `client/router.js` | `:title` → `:title(.*)` | Trivial |
| 5 | Index version | Bump to `"6"` | Trivial |

**Optional (UX polish):**
- Search result UI: show directory breadcrumb
- Note create: allow specifying directory
- Attachment path handling for nested notes

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| Path traversal attack | Validate with `os.path.realpath` check against `storage_path` |
| Existing flat-index incompatible | Bump `INDEX_SCHEMA_VERSION` to force rebuild |
| Breaking change for API consumers | `{title:path}` is backwards-compatible for flat titles (no `/` = same behavior) |
| Empty directories after move/delete | Clean up empty parents on delete (optional on move) |
| Wikilinks with `/` may confuse existing users | Document the new behavior; flat titles still work identically |

## Testing Plan

1. **Backwards compatibility:** Create flat notes (top-level only) — should work identically to before
2. **Nested create:** POST `/api/notes` with title `"Sub Dir/My Note"` — should create `Sub Dir/My Note.md` and parent directories
3. **Nested read:** GET `/api/notes/Sub Dir/My Note` — should return the note
4. **Nested rename:** PATCH with `newTitle: "Other Dir/My Note"` — should move the file
5. **Nested delete:** DELETE `/api/notes/Sub Dir/My Note` — should remove file and empty parent dirs
6. **Search:** Verify nested notes appear in search results
7. **Path traversal:** Try title `"../../etc/passwd"` — should be rejected with 400
8. **Index rebuild:** Start with existing flat index, verify it detects the schema version change and rebuilds with recursive scan
9. **UI navigation:** Navigate to `/note/Sub%20Dir%2FMy%20Note` — should render the note
10. **External file sync:** Add a `.md` file in a subdirectory externally, verify it appears on next search