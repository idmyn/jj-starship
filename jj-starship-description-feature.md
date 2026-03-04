# jj-starship: Add change description to prompt output

Feature request/patch plan for [dmmulroy/jj-starship](https://github.com/dmmulroy/jj-starship) to display the working copy description (and optionally the parent's description) in the prompt.

## Motivation

My previous starship jj prompt showed the change description and parent context, but was too slow in large repos because it shelled out to `jj log`. jj-starship uses `jj-lib` directly (no subprocess), so it's fast — but it currently discards the description text, only storing whether it's empty.

## Key findings

### The description is already fetched

In `src/jj.rs` (~line 217), `commit.description()` is already called on the working copy commit, but only a boolean is kept:

```rust
let empty_desc = commit.description().trim().is_empty();
```

Storing the first line as a `String` instead would be **zero additional I/O**.

### Parent description is also cheap

- `wc_commit.parent_ids()` is a free struct field access — the IDs are already loaded.
- Getting the parent's full `Commit` object requires one `get_commit()` call (single backend read, LRU-cached with 100-commit capacity).
- With the default `--ancestor-bookmark-depth 10`, the parent commit is **already loaded** by the BFS traversal in `find_ancestor_bookmarks`, so it's a cache hit — completely free.
- Once loaded, `parent.description()` is just a field access.

## Implementation plan

### 1. `src/jj.rs` — Add fields to `JjInfo`

```rust
pub struct JjInfo {
    // ... existing fields ...
    pub description: String,        // first line of WC description
    pub parent_description: String, // first line of parent description
}
```

Populate at the existing `commit.description()` call site:

```rust
let description = commit.description()
    .lines()
    .next()
    .unwrap_or("")
    .to_string();

let parent_description = commit.parent_ids()
    .first()
    .and_then(|id| repo.store().get_commit(id).ok())
    .map(|c| c.description().lines().next().unwrap_or("").to_string())
    .unwrap_or_default();
```

### 2. `src/output.rs` — Render in `format_jj()`

Add the description (truncated) after the existing status section, e.g.:

```
on 󱗆 yzxv1234 (main~3) [?] "fix the thing"
```

Or for the parent context (when the WC is not directly on a bookmark):

```
on 󱗆 yzxv1234 (main~3) [?] "fix the thing" on "parent description"
```

### 3. `src/main.rs` — Add CLI flags

Following the existing `--no-jj-*` pattern:

- `--no-jj-desc` — hide WC description
- `--no-jj-parent-desc` — hide parent description
- `--jj-desc-length <N>` — max description length (default: 30?)

And corresponding env vars:

- `JJ_STARSHIP_NO_JJ_DESC`
- `JJ_STARSHIP_NO_JJ_PARENT_DESC`
- `JJ_STARSHIP_JJ_DESC_LENGTH`

## Performance impact

**None.** The description data is already in memory (or one cached read away for the parent). No additional subprocess, no additional repo traversal.
