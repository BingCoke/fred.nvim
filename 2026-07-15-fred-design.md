# FRED Design Specification

**Status:** Accepted direction; implementation pending
**Date:** 2026-07-15
**Project:** `fred.nvim`
**Expansion:** Filesystem Representation Editor

## 1. Summary

FRED is a buffer-native file browser for Neovim. It projects selected parts of a local filesystem root into an ordinary editable buffer. The user expresses a desired namespace with normal text editing, then writes the buffer to plan, preview, confirm, and execute the corresponding filesystem operations.

FRED is independent. It does not depend on, integrate with, or call Oil. It may borrow general ideas such as editable directory buffers, entry identity, asynchronous rendering, operation planning, and confirmation before mutation, but it owns its scanner, snapshots, views, planner, executor, watcher integration, and user interface.

FRED uses a required Rust `cdylib` built locally by the user or plugin manager. The runtime is Rust-first and uses `nvim-oxi`. Lua is a thin stable facade for build integration, native-library loading, and the public `build`, `setup`, `open`, `refresh`, and `apply` functions. FRED does not publish precompiled binaries.

The view is a flat list of complete root-relative paths. It is not an indented tree, sidebar, or connector-glyph hierarchy. A view may materialize different directories to different depths while every displayed line remains a complete path.

Example buffer for `/home/user/project`:

```text
README.md
docs/
docs/design.md
src/
src/init.lua
src/parser.lua
tests/
```

The product description is:

> FRED is a buffer-native file browser for Neovim. Browse files, edit the filesystem.

## 2. Goals

FRED must:

1. Browse files and directories from a configurable local root.
2. Display root-relative entries in a flat, editable buffer.
3. Support a baseline recursion depth plus per-directory expansion and collapse.
4. Support filtering, sorting, ignore rules, bounded scanning, and partial views.
5. Let ordinary text edits express file and directory creation, rename, move, copy, replacement, and deletion.
6. Capture copy provenance from ordinary Vim yank and put operations where Neovim exposes or FRED wraps the operation.
7. Track entry identity independently of its current path.
8. Treat directory copy, move, and deletion as single logical tree operations rather than per-descendant plans.
9. Prevent hidden, filtered, ignored, unscanned, collapsed, or depth-excluded entries from being interpreted as deletions.
10. Share immutable filesystem snapshots and watcher coverage between multiple views of the same canonical root while preserving per-view intent and projection state.
11. Detect external namespace changes and update attached views automatically within materialized coverage.
12. Reject invalid paths, missing sources, wrong source kinds, missing parents, and occupied destinations before mutation when they are directly required by a plan.
13. Never silently overwrite an existing destination.
14. Preview every non-empty operation plan and require one all-or-nothing confirmation.
15. Permit only one FRED mutation anywhere in the Neovim process at a time.
16. Execute filesystem operations serially and stop immediately on the first execution error.
17. Refresh affected views after success or failure on a best-effort basis.
18. Remain responsive while scanning, copying, and rendering large directory sets.
19. Keep the public Lua interface small and stable.
20. Build from source for validated Neovim and `nvim-oxi` combinations.

## 3. Non-Goals For Version One

Version one will not:

- depend on or interoperate with Oil;
- provide an indented tree, connector glyphs, or a permanent sidebar layout;
- recursively follow directory symlinks;
- support SSH, S3, archives, containers, or other remote backends;
- expose a public backend registration interface;
- edit file contents inside the FRED buffer;
- track file-content versions through hashes, size, or modification time;
- infer external rename identity from inodes or platform file keys;
- provide ACID semantics across a batch of filesystem operations;
- automatically retry or undo a partially executed batch;
- provide a persistent transaction journal, rollback command, or crash recovery;
- provide a filesystem-level undo command;
- publish precompiled native binaries;
- support Neovim 0.10 or earlier;
- guarantee preservation of every platform-specific metadata feature during copy;
- guarantee atomic cross-filesystem moves;
- provide Dired-style persistent marks, arbitrary shell commands, recursive grep, archive management, or image galleries;
- silently overwrite an existing destination;
- treat filtering, sorting, ignoring, expansion, collapse, or depth changes as filesystem operations.

These constraints keep version one centered on paths, identity, immutable snapshots, views, desired-state planning, and a small fail-stop executor.

## 4. Why FRED Is Independent

Oil's current model binds one buffer to one directory URL and treats that buffer's lines as the direct children of one directory. Listing, cache ownership, rendering, parsing, mutation destinations, and watching derive from that single parent URL.

A flat editable view represents entries owned by many directories and may materialize different subdirectories to different depths. It requires per-row root-relative paths, shared root snapshots, multi-directory collision checks, subtree normalization, selective watcher coverage, and multi-view refresh semantics.

Oil's maintainer described recursive display as incompatible with Oil's current architecture in [issue #26](https://github.com/stevearc/oil.nvim/issues/26#issuecomment-1379844295). [PR #305](https://github.com/stevearc/oil.nvim/pull/305#pullrequestreview-1941933152) also demonstrated that merely allowing path separators in existing entries can miss unseen destination conflicts and risk data loss.

FRED therefore borrows concepts, not runtime modules or private interfaces. If any Oil source code is copied rather than independently reimplemented, its MIT license and copyright notice must be preserved.

## 5. Core Invariants

The implementation and tests must preserve these invariants.

### 5.1 Buffer Intent Mutates Only After Confirmed Write

Editing, deleting, putting, expanding, collapsing, filtering, sorting, refreshing, undoing, or re-rooting a FRED buffer does not mutate the filesystem by itself.

A non-empty FRED buffer plan may mutate only after:

1. the user invokes `:write` or `:FredApply`;
2. current buffer edits are captured;
3. the planner returns a valid plan;
4. the complete logical plan is shown;
5. the user confirms it;
6. the process-global mutation lock is acquired;
7. the plan's `edit_base_revision` matches the initiating View's current edit base, its `planned_root_revision` matches the RootSession's current published revision, and changedtick plus intent generation still match;
8. direct namespace probes are repeated under the lock and final preflight succeeds.

`:FredLink` is an explicit immediate helper outside the buffer planner. It is the only version-one FRED command that intentionally mutates without `:write`, and it must acquire the same process-global mutation lock.

### 5.2 Projection Absence Is Not Deletion

An entry absent from the current buffer because of ignore rules, filters, sorting, baseline depth, local collapse, scan limits, scan cancellation, scan failure, or missing watcher coverage remains part of filesystem state.

Absence from a projection cannot generate deletion. Only removal of a bound entry identity from an editable projection generates delete intent.

### 5.3 Path Is Not Identity

Every scanned entry receives an opaque `EntryId` scoped to its shared RootSession snapshot lineage. The ID does not derive from the displayed path, inode, or platform file key. Editing a path changes an entry's desired location but not its identity.

### 5.4 Direct Namespace Checks Authorize Operations

A partial or degraded browse snapshot does not globally disable apply. Before planning, the runtime captures immutable direct namespace-probe results for every source, destination, and parent required by the current intent. The pure planner consumes those probe results without performing I/O. Final preflight repeats the same probes under the process-global mutation lock.

An operation is blocked when its required namespace state cannot be established, including a missing source, wrong source kind, missing or non-directory parent, parent traversal through a symlink, invalid path, or occupied destination.

Directory COPY_TREE and DELETE_TREE do not require advance enumeration of every descendant.

### 5.5 Watchers Are Advisory But Active

Watchers actively trigger coalesced rescans and view updates. Watcher events can be missed, merged, delayed, or unavailable, so watcher health is not a correctness prerequisite for apply.

Watcher failure produces a visible degraded state. Direct final preflight remains authoritative for the current plan.

### 5.6 Only One FRED Mutation Runs At A Time

One process-global mutation lock covers all RootSessions, Views, public API apply calls, mappings, callbacks, and `:FredLink`.

The lock is acquired after preview confirmation and before final preflight. It remains held through serial execution and result publication. Preview does not hold the lock.

This global gate also prevents concurrent mutation between overlapping roots such as `/project` and `/project/src`.

### 5.7 Directory Operations Are Subtree Transformations

A directory rename or move generates one primary directory MOVE. Unedited descendants are rebased in the desired model and do not generate redundant actions.

A directory deletion generates one DELETE_TREE. A directory copy generates one COPY_TREE. The planner does not expand either operation into per-descendant logical actions.

### 5.8 The Final Desired State Determines COPY Versus MOVE

FRED plans from the final captured buffer state, not from the sequence of editing commands:

- source remains plus provenance destination: COPY or COPY_TREE;
- source removed plus one provenance destination: MOVE;
- source removed plus multiple provenance destinations: COPY each destination, then DELETE the source.

### 5.9 Execution Is Serial And Fail-Stop

Execution nodes run serially in dependency order. The first failed system call stops the batch immediately. Later nodes are not run. Completed effects remain. FRED does not automatically retry or reverse them.

System API success is trusted. Completed execution-node results are applied deterministically to the in-memory namespace model before the initiating View is re-rendered. Filesystem refresh is best effort; a refresh error is reported through Neovim without adding a recovery, unknown-state, or apply-blocking state machine.

## 6. Platform, Build, And Runtime Architecture

### 6.1 Supported Neovim And Platform Combinations

FRED supports only the exact compatibility pairs listed in the build configuration. Each pair contains:

- one exact Neovim patch version;
- one pinned `nvim-oxi` revision.

The build helper accepts only a listed pair and rejects every unlisted patch/revision combination rather than guessing a feature selection. Other Neovim patch versions, including nightly, are unsupported until they are explicitly validated and added to the list. Neovim 0.10 and earlier remain unsupported.

FRED actively supports and tests:

- Linux;
- Windows.

macOS and other platforms are best effort when the project compiles. Platform-specific behavior is added when concrete failures are reported rather than promised in advance.

### 6.2 Local Build Only

FRED publishes source code, not native binaries. A plugin manager may build it with a Lua helper:

```lua
{
  "BingCoke/fred.nvim",
  build = function()
    require("fred").build()
  end,
}
```

The helper detects the exact running Neovim patch version, selects its listed pinned `nvim-oxi` revision, invokes `cargo build --release`, and installs or locates the resulting platform-native library. Artifacts for incompatible listed pairs must not overwrite one another.

Upgrading Neovim or the pinned `nvim-oxi` revision may require rebuilding FRED.

### 6.3 Rust-First Runtime

Rust owns:

- user commands and mappings;
- FRED buffers, windows, extmarks, virtual text, and preview UI;
- runtime registration, RootSession, and View state;
- scanning, watching, path handling, projection, identity, provenance, intent capture, planning, execution, and refresh;
- background tasks and main-thread notification.

Lua owns only:

- the build helper;
- native library discovery and loading;
- a small stable public facade.

### 6.4 Background Work And Task State

Version one does not use Tokio or a general-purpose asynchronous runtime.

Each RootSession is in one of three conceptual states:

```text
Idle
Scan
Apply
```

`Idle` means no RootSession filesystem task is running. `Scan` covers ordinary browse, expansion, and refresh scans. `Apply` covers final preflight, execution, and the initiating session's result publication.

Rules:

- a newer ordinary scan request may cancel and replace an older scan generation;
- confirmed apply cancels the initiating RootSession's ordinary scan;
- scan events from obsolete generations are ignored;
- scans do not publish a competing snapshot while that RootSession is applying;
- watcher events received during apply are coalesced for the post-apply refresh;
- every post-lock outcome, including final-preflight failure, leaves Apply through one common finalization path that processes queued refresh demand before releasing the global mutation lock;
- another apply or `:FredLink` receives a busy error while the global mutation lock is held.

Long work uses:

```text
std::thread
std::sync::mpsc::sync_channel
AtomicBool cancellation
nvim_oxi::libuv::AsyncHandle
```

Worker threads never call Neovim functions directly. They enqueue bounded events and use `AsyncHandle::send()` only to wake the main loop. Because wakeups may coalesce, the main-thread callback drains all currently available events from the queue.

Parallelism may be added later only where benchmarks demonstrate a real bottleneck.

## 7. User Experience

### 7.1 Opening FRED

```vim
:Fred [root]
```

If `root` is omitted, FRED uses the current working directory. The root is canonicalized and associated with a shared RootSession. The root itself is not represented by an editable row.

Multiple FRED buffers may open the same canonical root. They share immutable snapshots, scan cache, and watcher coverage while retaining independent projection and intent.

Lua interface:

```lua
require("fred").build()
require("fred").setup(opts)
require("fred").open(root, opts)
require("fred").refresh(bufnr)
require("fred").apply(bufnr)
```

Each View uses a private URI containing the canonical root and an opaque view identifier, for example:

```text
fred:///percent-encoded-canonical-root?view=opaque-id
```

The URI identifies a View and is never treated as a filesystem path.

FRED buffers use:

```text
buftype=acwrite
swapfile=false
```

### 7.2 Buffer Syntax And Canonical Path Encoding

Each editable line contains exactly one encoded root-relative path.

```text
README.md
src/
src/init.lua
```

Rules:

- directories end with `/`;
- files and symlinks do not end with `/`;
- `/` is the logical separator on every platform;
- paths are relative to the View root;
- empty components, absolute paths, literal or encoded `.`, literal or encoded `..`, NUL, and normalized root escape are invalid;
- normal displayable Unicode remains unchanged;
- literal `%` is encoded as `%25`;
- newline is encoded as `%0A` and carriage return as `%0D`;
- other control bytes and native filename units that cannot be displayed directly use a reversible canonical escape with uppercase hexadecimal digits;
- malformed escapes are invalid;
- encoded path separators are invalid, including encoded `/` and Windows-native separators;
- decoding occurs exactly once and independently for each component;
- decode-then-reencode must reproduce the exact buffer text, so aliases such as `%41` for `A` are invalid;
- decoded components are validated before joining them to the root;
- native Windows names use reversible conversion rather than lossy replacement;
- type, status, and marks are virtual text and are not part of editable syntax.

FRED does not indent entries or draw connector glyphs. Directories and descendants are related by path prefixes, not visual tree structure.

### 7.3 Commands And Mappings

Default commands:

```text
:Fred [root]                 open a View
:FredApply                   plan, preview, confirm, and execute
:FredRefresh                 force filesystem reconciliation
:FredDepth {n|all}           change baseline recursion depth
:FredExpand [depth]          locally expand the current directory
:FredCollapse                locally collapse the current directory
:FredFilter {pattern}        set a temporary projection filter
:FredFilter!                 clear the temporary filter
:FredPasteInto               paste a FRED-aware yank into the current directory
:FredNew {file|dir}          insert an explicitly typed pending entry
:FredLink {from} {to}        immediately create a symlink outside the planner
```

Suggested default mappings:

```text
<CR>       open a file; re-root on a directory
-          open the parent root after resolving dirty state
gp         paste a FRED-aware yank into the current directory
zo         expand the current directory one level
zc         collapse the current directory
zO         recursively materialize the current directory
zC         recursively collapse the current directory
R          force refresh
<leader>s  apply changes
q          close after resolving dirty state
```

Normal Vim editing remains the primary interface for create, rename, move, copy, replacement, and delete.

### 7.4 Write Semantics

Full-buffer `:write`, `:update`, and `:wq` invoke the same apply behavior as `:FredApply`.

Rules:

- every non-empty plan is previewed and confirmed;
- `:wq` closes only after successful apply or a successful empty plan;
- preview cancellation, validation failure, stale-plan failure, lock contention, or final-preflight failure leaves `'modified'` set;
- successful apply clears `'modified'` and establishes a new ordinary undo baseline;
- an empty plan reprojects and sorts the View, clears `'modified'`, and returns without preview;
- alternate-file writes, ranged writes, and append writes are rejected;
- FRED does not export its path list through ordinary write syntax.

### 7.5 Opening Entries

`<CR>` on a file opens it using normal Neovim buffer behavior.

`<CR>` on a directory re-roots the current FRED buffer to that directory. If the View has uncommitted intent, FRED requires apply, discard, or cancel before changing roots.

FRED lists and opens symlinks using ordinary Neovim behavior but never recursively scans through a directory symlink. The link remains a leaf entry in FRED.

### 7.6 Local Expansion And Collapse

A View has a configurable baseline depth plus local directory overrides.

Example with baseline depth `0`:

```text
README.md
src/
tests/
```

Running `zo` on `src/` materializes only its direct children:

```text
README.md
src/
src/init.lua
src/lib/
tests/
```

This adds scan and watcher coverage for `src/`, but not for `tests/` or `src/lib/` until those directories are materialized.

Collapse captures current edits first and hides only clean descendants. Entries with pending intent, conflicts, or validation errors remain pinned and visible with any required ancestor context. Projection changes never create filesystem intent.

## 8. Expressing File Operations

### 8.1 Create

A new unbound line creates an entry at its path:

```text
notes.txt        CREATE file
assets/          CREATE directory
```

`:FredNew file` and `:FredNew dir` insert explicitly typed pending rows when normal line syntax is insufficient for the desired workflow.

Missing parent directories are not created implicitly for an ordinary file row. The parent must already exist or have its own pending directory row.

Symlink creation is not represented by a pending buffer row; it uses the immediate `:FredLink` helper described in Section 19.

### 8.2 Rename And Move

Editing the path of an existing identity expresses rename or move:

```text
src/parser.lua
```

changed to:

```text
src/core/parser.lua
```

produces:

```text
MOVE src/parser.lua -> src/core/parser.lua
```

The destination parent must exist or be created by the same plan.

### 8.3 Delete And DELETE_TREE

Removing an existing bound file or symlink row expresses DELETE.

Removing a directory row expresses one logical DELETE_TREE with remove-tree semantics equivalent to deleting the complete directory and its contents. The planner does not enumerate descendants or generate one delete action per child.

When a directory row is removed:

- unedited descendants under that directory are implicitly included in DELETE_TREE;
- those descendants are hidden on the next buffer synchronization so the View does not display paths that will not exist in the desired state;
- descendants explicitly moved outside the deleted subtree are pinned and evacuated before DELETE_TREE;
- undoing the directory-row removal restores its descendant projection;
- filesystem mutation still waits for confirmed write.

DELETE_TREE does not require the directory to be expanded or fully scanned. Preview shows the directory path and indicates that all contents are included. Cached counts may be displayed, but FRED does not calculate counts as a prerequisite.

### 8.4 Copy Provenance From Normal Editing

Extmarks track positions through supported buffer edits but do not travel through yank or put. FRED therefore records provenance separately.

On Neovim 0.11:

- FRED installs buffer-local `p` and `P` wrappers;
- the wrappers preserve normal put behavior while recording the register, source EntryIds, and inserted range;
- `TextYankPost` records FRED-aware yank and delete provenance;
- insertions not observed through the wrappers are ordinary unbound CREATE declarations, even if their text matches an existing row.

On Neovim 0.12, FRED may additionally use `TextPutPre` and `TextPutPost` to cover native put paths exposed by Neovim.

Typical behavior:

```text
yy + p + edit destination
```

If the source remains in the final desired state, the result is COPY or COPY_TREE. If the source is removed and one provenance-linked destination remains, the result is MOVE.

A `dd` followed by an observed `p` may rebind the uniquely removed identity and behave as MOVE after the destination path is edited.

FRED never guesses ambiguous provenance.

### 8.5 Multiple Provenance Destinations

If a removed source has multiple provenance-linked destinations, FRED does not arbitrarily choose one destination to inherit a MOVE identity.

Example final plan:

```text
COPY a.txt -> b.txt
COPY a.txt -> c.txt
DELETE a.txt
```

All copies precede source deletion. The same rule applies to directory sources using COPY_TREE and DELETE_TREE.

### 8.6 Paste Into Directory

`:FredPasteInto`, mapped to `gp`, copies the most recent FRED-aware yank into the directory under the cursor.

Example:

```text
xx
src/
```

Yank `xx`, move to `src/`, and press `gp`:

```text
xx
src/
src/xx        [COPY from xx]
```

The actual copy waits for confirmed write.

For a multi-entry yank, FRED keeps only top-level selected sources. If a selected directory contains another selected entry, the descendant is not copied redundantly. A directory destination still produces one COPY_TREE for the selected directory.

If the generated destination exists or conflicts with a pending destination, `gp` selects the first available `_N` suffix:

```text
xx             -> xx_1
xx_1           -> xx_2
report.txt     -> report_1.txt
archive.tar.gz -> archive.tar_1.gz
.env           -> .env_1
assets/        -> assets_1/
```

Candidate checks include current filesystem entries, hidden or unprojected entries discoverable in the destination parent, pending destinations, the current paste batch, and destination-filesystem collision behavior. Final execution still uses no-replace behavior; a destination occupied before apply produces a collision instead of automatic renaming.

### 8.7 Replacement And Destination Collision

A destination collision never implies overwrite.

CREATE, COPY, COPY_TREE, MOVE, and symlink creation use exclusive or no-replace system behavior. If the destination is occupied, execution reports a collision and stops.

Replacement is planned only when the user explicitly removes the original target identity from the desired state. A type change such as `name` to `name/`, or the reverse, is not inferred from a trailing slash edit. It requires explicit removal of the old identity and creation of the new typed entry.

REPLACE is a logical preview grouping for that explicit desired-state transition. Its executor steps remain serial and fail-stop; version one does not promise rollback if removal succeeds and installation fails.

### 8.8 COPY_TREE

Directory copy is one logical operation:

```text
COPY_TREE source/ -> destination/
```

The planner does not pre-enumerate descendants, create a manifest, hash contents, or emit per-file COPY nodes. A deep internal `copy_tree(from, to)` module or platform adapter performs the traversal needed by the operating system implementation.

COPY_TREE writes directly to the final desired destination. It does not use an intermediate copy location. If copying fails partway, a partial destination may remain. The executor stops, reports the error, and refreshes affected views on a best-effort basis.

A directory cannot be copied to itself or to any descendant of itself after canonical path comparison.

### 8.9 Cross-Filesystem Move

A same-filesystem MOVE uses the platform rename operation when possible.

When rename reports a cross-filesystem condition, the logical MOVE expands internally into:

```text
COPY or COPY_TREE source -> final destination
DELETE or DELETE_TREE source
```

Source deletion runs only after the copy operation reports success. If source deletion fails, both source and destination may remain. FRED stops and reports the partial result.

A directory cannot be moved to itself or into its own descendant.

## 9. Directory Move Semantics

Consider this base state:

```text
src/
src/main.lua
src/lib/
src/lib/parser.lua
```

Renaming `src/` to `source/` produces one directory MOVE:

```text
MOVE src/ -> source/
```

The desired paths of unedited descendants automatically become:

```text
source/main.lua
source/lib/
source/lib/parser.lua
```

They do not produce separate actions.

If a descendant's final path is outside the parent's final subtree, it is evacuated first:

```text
MOVE src/lib/parser.lua -> parser.lua
MOVE src/ -> source/
```

If a descendant remains inside the parent's final subtree, the parent moves first and the descendant operation uses a rebased source path:

```text
MOVE src/ -> source/
MOVE source/lib/ -> source/core/
```

Ordering is therefore conditional, not unconditionally outermost-first or innermost-first. Planner output contains no redundant moves for unchanged descendants.

Rename swaps, cycles, and case-only renames may use unique internal same-directory temporary names. These names are executor details and are not part of the desired state or logical preview.

## 10. Shared RootSession And Per-View State

### 10.1 Immutable Snapshots

A published filesystem snapshot is immutable and carries a monotonically increasing revision. A scan generation builds one candidate snapshot plus provisional rendering batches without mutating `current_snapshot`.

Provisional batches and individual scopes may render as they arrive, but a generation publishes exactly one new immutable `current_snapshot` only after every requested scope reaches a terminal complete, partial, or failed result. Partial and failed scopes remain explicitly non-authoritative inside that single published result. Late or cancelled generations cannot publish, and provisional data never updates a View's edit base.

Published snapshots remain alive while referenced by dirty Views and are released when no View needs them.

### 10.2 RootSession

All Views of the same canonical root share one Rust RootSession:

```text
RootSession {
  canonical_root
  current_snapshot: Arc<Snapshot>
  root_revision
  scan_cache
  directory_status
  watcher_state
  coverage_reference_counts
  task_state: Idle | Scan | Apply
}
```

The RootSession does not own a mutation lock. The mutation lock is process-global because different canonical roots can overlap.

### 10.3 View

Each FRED buffer owns independent View state:

```text
View {
  view_id
  bufnr
  view_generation
  baseline_depth
  local_expansions
  filter
  sort
  projection
  intent
  intent_generation
  dirty_state
  edit_base: Arc<Snapshot>
  cursor_identity
  last_changedtick
}
```

A clean View may advance its edit base whenever a new snapshot is published. A dirty View keeps the immutable snapshot against which its current intent was captured.

Two Views over the same root may use different depth, expansion, filter, and sort settings and may both contain edits.

### 10.4 Revision And Rebase Behavior

When a new snapshot is published:

- clean Views reproject from it;
- dirty Views compare their immutable edit base, the new snapshot, and their desired namespace;
- non-overlapping namespace changes merge automatically;
- conflicting paths, kinds, identities, or destinations remain visible and block the affected plan;
- the dirty View advances its edit base only after a successful rebase.

File contents, size, and modification time are not part of this conflict model.

### 10.5 View Lifecycle

The runtime owns the authoritative registry keyed by opaque `view_id`, with validated secondary lookup by `bufnr`.

Buffer wipeout, re-root, and explicit close invalidate the old View generation, release coverage references, and detach it from its RootSession. Asynchronous events carry root, View, and generation identity rather than trusting a reusable buffer number.

### 10.6 Overlapping RootSessions

After any successful or partially successful mutation, the runtime invalidates scan generations and schedules refresh for every attached RootSession whose root overlaps an affected path. This keeps parent-root and child-root Views consistent even though they do not share one RootSession.

## 11. Identity And Provenance

Every rendered existing row has an extmark carrying its opaque RootSession-scoped `EntryId`. Marks and statuses bind to EntryId, not line number.

Identity rules:

- editing text preserves identity while the extmark remains associated with that row;
- moving text preserves identity only when Neovim actually moves the extmark with it;
- yank and put never copy an extmark;
- a provenance-linked put receives a new pending identity with `copy_from = EntryId`;
- source removal plus one destination may normalize that destination to MOVE;
- two existing rows cannot carry the same identity;
- if an extmark is lost, FRED may rebind only through a unique, unconsumed, unchanged base path;
- ambiguous lost identity is never guessed;
- external rename is represented conservatively as deletion of the old path and creation of the new path.

Hard links are separate namespace entries and receive separate EntryIds even when the operating system reports shared underlying storage.

## 12. Scanner, Coverage, And Watchers

### 12.1 Scanner Events And Atomic Publication

The scanner is a cancellable Rust worker task. A request includes:

```text
root
required directory scopes
baseline depth
local expansion overrides
ignore matcher
max_entries
max_directories
generation
```

The worker emits bounded events such as:

```text
ScanBatch
ScanProgress
DirectoryComplete
DirectoryFailed
ScanComplete
ScanCancelled
```

Requirements:

- emit at most `render_batch_size` entries per batch;
- attach a generation to every event;
- discard events from obsolete generations;
- record each requested scope as complete, partial, or failed for that generation;
- never follow a directory symlink;
- stop at configured limits and report partial coverage;
- publish exactly one immutable snapshot only after every requested scope in the generation has reached a terminal result;
- retain partial and failed scopes as non-authoritative status inside the published generation result;
- keep Neovim calls on the main thread.

Default limits:

```lua
max_entries = 50000
max_directories = 10000
render_batch_size = 500
```

Reaching a limit or failing an unrelated directory produces a visible partial state; it does not globally disable apply.

### 12.2 Coverage Union

A RootSession scans and watches the union of directories required by attached Views.

Coverage includes:

- each View's baseline depth;
- local expansions;
- directories needed to display pinned intent, conflict, and validation-error rows;
- ancestor context needed by those rows.

Temporary filters do not reduce coverage because they alter projection only.

Coverage is reference-counted. When no attached View needs a directory, its watcher may be released and its cached state may cease to be authoritative. COPY_TREE and DELETE_TREE do not add descendant enumeration coverage.

### 12.3 Watcher Behavior

FRED automatically attempts to establish watchers for covered directories; version one has no watcher toggle. Established watchers monitor covered directories. Events are debounced and coalesced by affected directory, then trigger incremental scan requests.

FRED-generated mutations also produce watcher events. The executor reports affected paths so watcher events can be merged with explicit post-execution refresh.

If watcher coverage cannot be established, FRED displays a degraded warning and leaves manual refresh available. Missing watcher coverage alone does not prevent apply.

## 13. Ignore, Filter, Sort, And Depth

Ignore rules use gitignore-like ordered matching with `!` negation, directory patterns, and root anchoring. Ignoring affects browse traversal and projection, never deletion authority and never the contents copied or deleted by a logical tree operation.

Temporary filters operate only on already scanned entries. Entries with intent, conflict, or validation errors remain visible.

Baseline depth semantics:

- `0`: entries directly under root;
- `1`: direct entries plus children of direct directories;
- `all`: no depth limit other than configured scan limits.

Local expansion depth is relative to the selected directory. `zo` expands one level; `zO` requests recursive materialization subject to scan limits.

Changing ignore rules requests a new browse scan. Changing filter or sort only reprojects. Changing baseline depth or local expansion updates coverage. None of these changes generates filesystem intent.

Default ordering is stable lexical ordering by normalized relative path. Destination collision checks use actual platform/filesystem behavior, while displayed spelling remains unchanged.

## 14. Planner

The planner is a pure Rust module. It receives immutable inputs and returns either a valid logical plan or non-confirmable diagnostics. It never performs filesystem I/O or mutates the filesystem.

Inputs:

```text
View edit-base snapshot and edit_base_revision
current published snapshot and planned_root_revision
captured View intent
current changedtick and intent generation
immutable direct namespace-probe results for required sources, destinations, and parents
path and platform rules
execution policy
```

Outputs:

```text
ValidPlan {
  edit_base_revision
  planned_root_revision
  changedtick
  intent_generation
  ordered operation DAG
  logical preview groups
  affected paths
}

or

Diagnostics {
  validation errors
  conflicts
}
```

Logical operations shown to users include:

```text
CREATE
COPY
COPY_TREE
MOVE
DELETE
DELETE_TREE
REPLACE
```

Internal execution nodes may include:

```text
mkdir
create_empty_file
copy_file
copy_tree
rename
rename_via_unique_same_directory_name
remove_file
remove_tree
create_symlink
```

`create_symlink` is used only by the immediate `:FredLink` path, not a FRED buffer plan.

Planner validation rejects:

- duplicate final destinations;
- absolute or root-escaping buffer paths;
- empty, `.`, or `..` components;
- malformed or noncanonical path encoding;
- encoded separators;
- platform-invalid paths;
- destinations that collide under the destination filesystem's behavior;
- destination occupied by an identity not explicitly removed;
- source that disappeared or changed kind;
- missing or non-directory parent;
- parent traversal through a symlink;
- directory COPY_TREE or MOVE to itself or its descendant;
- dependency cycles that cannot be resolved with a unique same-directory rename name.

The planner does not reject ordinary file-content, size, or modification-time changes.

A destination occupied by another planned identity is allowed only when that identity is moved away or explicitly removed earlier in the same plan. Final execution still uses exclusive/no-replace operations.

### 14.1 Final-State Provenance Normalization

The planner normalizes provenance from the final desired state:

```text
source retained + destination added
  -> COPY or COPY_TREE

source removed + one destination added
  -> MOVE

source removed + multiple destinations added
  -> COPY each destination, then DELETE or DELETE_TREE source
```

The normalization chooses operations that reach the final namespace without treating the user's edit sequence as an execution script.

## 15. Operation Ordering

The operation DAG enforces:

1. parent directory creation before child creation;
2. all copies from a source before deleting that source;
3. descendant evacuation before parent directory MOVE or DELETE_TREE when the descendant ends outside the parent's final subtree;
4. parent MOVE before descendant edits that remain inside the parent's final subtree;
5. unique temporary rename steps before completing a swap, cycle, or case-only rename;
6. destination copy success before source deletion for cross-filesystem MOVE;
7. explicit target removal before installing REPLACE when required by the desired state;
8. irreversible deletes as late as dependencies permit.

All execution nodes run serially. Version one does not execute independent nodes concurrently.

## 16. Preview And Confirmation

Validation errors and conflicts produce a non-confirmable diagnostic report and location list. The initiating View remains dirty, and its edits and undo history are preserved. Diagnostics never enter the confirmation flow.

Only a valid non-empty plan opens one confirmation preview grouped by logical operation:

```text
CREATE
COPY
COPY_TREE
MOVE
DELETE
DELETE_TREE
REPLACE
```

Each row shows the source, destination, and kind. Tree operations are displayed as one logical row. Cached informational counts may be displayed, but FRED does not enumerate a tree merely to populate preview metadata.

Confirmation is all-or-nothing. The user cannot deselect individual operations because doing so could invalidate dependencies or produce a namespace different from the edited buffer. To omit an operation, the user cancels, edits the buffer, and applies again.

Every valid non-empty plan uses this single confirmation. An empty valid plan reprojects, sorts, clears `'modified'`, and returns without opening preview.

## 17. Executor And Failure Semantics

The executor consumes a validated plan and is the only module allowed to mutate planned FRED buffer operations.

After the user confirms a valid plan:

1. acquire the process-global mutation lock;
2. cancel the initiating RootSession's ordinary scan and enter Apply state;
3. compare the plan's changedtick and intent generation with the initiating View;
4. compare `edit_base_revision` with the initiating View's current edit-base revision;
5. compare `planned_root_revision` with the RootSession's current published revision;
6. repeat every required source, destination, and parent namespace probe under the lock;
7. execute the plan serially only if all checks still match;
8. apply completed execution-node results deterministically to the in-memory namespace model and publish the resulting immutable model revision;
9. invalidate overlapping RootSessions and queue best-effort filesystem refresh;
10. run the common post-lock finalization path: leave Apply, process queued refresh demand, and release the global lock.

Changedtick, intent-generation, revision, or final-probe mismatch aborts before mutation. That final-preflight outcome preserves the initiating View's intent, buffer text, `'modified'`, and undo history, but still runs the common finalization path.

### 17.1 Serial Fail-Stop Execution

After execution starts:

1. run one node at a time;
2. record the system API result;
3. stop immediately on the first error;
4. do not run later nodes;
5. do not automatically retry;
6. do not automatically undo completed nodes.

The report groups:

```text
COMPLETED
FAILED
UNSTARTED
```

All destination creation and installation uses exclusive or no-replace behavior. A destination that appears after final preflight causes the relevant system call to fail rather than overwrite it.

COPY_TREE may leave a partial final destination when its underlying operation fails. Rename cycles may leave an internal unique name if execution or the process stops partway. These are reported as actual filesystem results, not hidden behind compensating behavior.

### 17.2 Success

When every node reports success, FRED trusts those system API results.

It then:

- applies every completed node effect deterministically to the in-memory namespace model;
- publishes a new immutable model revision;
- clears the initiating View's intent;
- re-renders the initiating View from that model;
- clears `'modified'`;
- establishes a new ordinary undo baseline;
- invalidates every RootSession overlapping an affected path;
- queues best-effort filesystem refresh;
- reports success;
- exits through the common post-lock finalization path.

A refresh error is reported through Neovim and may be corrected by a later watcher event or `:FredRefresh`. It does not create a special degraded, unknown, reconciliation-required, or apply-blocking state.

### 17.3 Execution Failure

On the first execution error, FRED:

1. stops the batch;
2. reports the failed system call through Neovim;
3. records completed, failed, and unstarted nodes;
4. applies only completed-node effects deterministically to the in-memory namespace model and publishes the resulting immutable model revision;
5. abandons the initiating View's failed and unstarted intent from that apply attempt;
6. re-renders the initiating View from the updated in-memory model, clears `'modified'`, and establishes a new ordinary undo baseline;
7. invalidates overlapping RootSessions and queues best-effort filesystem refresh;
8. shows the execution report;
9. exits through the common post-lock finalization path.

A failed node is not assumed successful; a later refresh may reveal partial side effects left by that failed system call. The user edits the re-rendered model and starts a new apply if desired. FRED does not offer an automatic retry action.

Other dirty Views retain their own intent and rebase against the next published snapshot.

If refresh itself fails, FRED reports that error through Neovim. It does not replay the abandoned plan, restore the old buffer intent, or introduce a degraded, unknown, reconciliation-required, recovery, or apply-blocking state.

### 17.4 Process Kill Or Power Loss

FRED does not attempt crash or power-loss recovery. A process kill may leave partially completed operations, partial copy destinations, duplicate cross-filesystem move results, or temporary rename names.

On the next open, FRED performs an ordinary scan and displays what exists. It does not infer, continue, or reverse an interrupted batch.

## 18. Refresh And Conflicts

Refresh compares:

```text
View edit base
filesystem namespace now
View desired namespace
```

Rules:

- disk namespace changed, user unchanged: accept disk state;
- user changed, disk namespace still matches the edit base: keep user intent;
- source path disappeared: conflict;
- source kind changed: conflict;
- desired destination appeared or became occupied: conflict;
- both disk and user changed the same namespace identity incompatibly: conflict;
- ordinary file content, size, or modification time changed: not a FRED conflict;
- external rename: represent conservatively as old-path deletion plus new-path creation.

Watcher events trigger incremental refresh automatically. `:FredRefresh` forces reconciliation of the View's required browse coverage.

A dirty View is never silently overwritten by ordinary refresh. Conflicted entries remain visible regardless of filters or collapse.

The initiating View after a partial execution is the intentional exception: its failed and unstarted intent is abandoned, completed-node effects update the in-memory namespace model, and the View is re-rendered from that model before best-effort filesystem refresh.

## 19. Symlink Policy And Immediate Helper

FRED scanning never recursively follows a directory symlink. Existing symlinks are leaf entries that may be listed, opened, renamed, copied, or deleted as links.

This prevents recursive cycles, alias-based duplicate traversal, and accidental ownership of a target outside the View root.

Symlink creation is an immediate helper rather than a buffer-planned operation:

```vim
:FredLink {from} {to}
```

Semantics:

- `from` is passed to the platform symlink call as the link target;
- relative `from` remains relative and is interpreted normally from the created link's parent when the link is followed;
- `to` may be an absolute or relative path inside or outside a FRED root;
- relative `to` resolves against the current buffer directory;
- for a FRED buffer, the buffer directory is the View root;
- for a normal file buffer, it is the file's parent directory;
- for an unnamed or non-file buffer, it falls back to the current window working directory;
- the destination is created with exclusive/no-replace behavior;
- the command acquires the process-global mutation lock and reports busy if another mutation is active;
- the command executes immediately and does not wait for `:write`;
- after acquiring the lock, the command uses one finally-style exit path that releases it after success and every failure;
- if the symlink system call succeeds or may have partially mutated the namespace, the command invalidates overlapping RootSessions and queues or performs best-effort refresh before releasing the lock;
- operation and refresh errors are reported directly through Neovim without introducing an unknown, degraded, recovery, or apply-blocking state.

## 20. Internal Modules

Suggested structure:

```text
Cargo.toml
build.rs
src/
  lib.rs
  config.rs
  runtime.rs
  apply_gate.rs
  root_session.rs
  snapshot.rs
  view.rs
  task.rs
  scanner.rs
  watcher.rs
  coverage.rs
  model.rs
  identity.rs
  provenance.rs
  path.rs
  projection.rs
  edit.rs
  planner.rs
  conflict.rs
  preview.rs
  executor.rs
  tree_ops.rs
  symlink.rs
  report.rs
  ignore.rs
lua/
  fred/init.lua
plugin/
  fred.lua
```

The modules expose small internal interfaces with substantial behavior behind them:

- `path` owns canonical buffer/native path conversion and validation;
- `snapshot` owns immutable revisioned namespace state;
- `root_session` owns shared scan, watcher, and coverage state;
- `view` owns per-buffer projection, edit base, and intent;
- `planner` maps immutable namespace inputs to a logical DAG;
- `tree_ops` exposes single deep COPY_TREE and DELETE_TREE operations while hiding platform traversal;
- `executor` is the sole planned-mutation caller;
- `apply_gate` owns the process-global mutation lock;
- `runtime` owns View and RootSession registration and overlap invalidation.

Rust module interfaces remain internal. `src/lib.rs` exposes the native Lua module through `nvim-oxi`. `lua/fred/init.lua` is the stable public Lua facade and build loader.

No speculative backend seam is exposed in version one.

## 21. Public Configuration

Initial configuration shape:

```lua
require("fred").setup({
  depth = 0,
  ignore = {
    ".git/",
  },
  sort = {
    by = "path",
    case_sensitive = false,
  },
  limits = {
    max_entries = 50000,
    max_directories = 10000,
    render_batch_size = 500,
  },
  mappings = {
    open = "<CR>",
    parent = "-",
    paste_into = "gp",
    expand = "zo",
    collapse = "zc",
    expand_recursive = "zO",
    collapse_recursive = "zC",
    refresh = "R",
    apply = "<leader>s",
    close = "q",
  },
})
```

Baseline `depth = 0` keeps initial scanning and watcher coverage shallow. Users may raise it globally or materialize selected directories locally.

Watcher establishment is automatic in version one and is not configurable. Establishment failure produces a degraded warning without blocking apply.

Confirmation is not configurable in version one: every non-empty plan previews and confirms once.

Symlink traversal is not configurable in version one: directory symlinks are always leaves.

Configuration is validated during setup. Invalid values produce explicit errors rather than silent fallback.

## 22. Error Handling

Errors fall into four classes.

### 22.1 Scan Or Watch Error

The buffer remains readable. Failed or limited browse scopes are marked partial or degraded. Manual refresh and scope reduction remain available.

Scan or watcher failure does not globally disable apply. A plan is blocked only when its own required source, destination, or parent state cannot be checked.

### 22.2 Validation Or Final-Preflight Error

No filesystem action runs. The buffer remains dirty, edits and undo history are preserved, and errors are attached to affected rows and listed in a non-confirmable diagnostic report or location list.

A stale changedtick, intent generation, `edit_base_revision`, or `planned_root_revision` also returns here and requires replanning. If this failure occurs after lock acquisition, the common finalization path leaves Apply, processes queued refresh demand, and releases the global mutation lock.

### 22.3 Conflict

No filesystem action runs. Conflicted entries remain visible in the non-confirmable diagnostic report and require refresh, rebase, or explicit user edits.

Typical conflicts are missing sources, changed source kind, occupied destinations, invalid parents, and incompatible namespace changes. Conflicts never open the confirmation preview.

### 22.4 Execution Failure

The executor stops immediately, reports the system error, and runs no later operation. Completed-node effects update the in-memory namespace model deterministically; failed and unstarted initiating-View intent is abandoned; the initiating View is re-rendered from that model; filesystem refresh remains best effort.

A refresh error is reported through Neovim without blocking a later apply or creating a special state. FRED does not claim that the desired batch completed when an execution node failed.

## 23. Testing Strategy

### 23.1 Rust Unit Tests

- canonical path encoding and decoding;
- decode-then-reencode equality;
- malformed escapes, `%41` aliases, encoded separators, literal and encoded `.`/`..`, absolute paths, and root escape rejection;
- Linux byte-path and Windows native-name conversion used by supported builds;
- EntryId allocation and extmark-loss rebinding rules;
- provenance normalization for retained source, one removed-source destination, and multiple removed-source destinations;
- ignore, filter, sort, depth, expansion, collapse, scan cancellation, scan failure, scan limits, and watcher gaps never generating deletion;
- immutable direct namespace-probe inputs for missing sources, wrong kinds, occupied destinations, missing or non-directory parents, and symlink parent traversal;
- plans carrying distinct `edit_base_revision` and `planned_root_revision` tokens;
- common post-lock finalization after preflight failure, success, and execution failure;
- directory-row deletion hiding unedited descendants;
- descendant evacuation from DELETE_TREE and parent MOVE;
- directory COPY_TREE/MOVE-to-self and descendant rejection;
- conditional nested directory move ordering;
- duplicate targets and explicit replacement;
- rename swaps, cycles, and case-only renames;
- DAG topological ordering;
- ignore rule negation and browse traversal semantics;
- namespace-only three-way refresh and conflict classification;
- file content, size, and mtime changes not producing conflicts;
- external rename represented as delete plus create;
- immutable edit-base retention and candidate snapshot publication;
- RootSession coverage reference counting.

### 23.2 Integration Tests

- recursive and selectively materialized local scanning;
- one immutable snapshot publication only after every requested scope in a scan generation reaches a terminal result;
- partial and failed scopes remaining non-authoritative inside the published generation result;
- cancellation and late-generation rejection;
- partial scan limits while unrelated known intents remain applicable;
- watcher establishment failure while direct apply checks remain usable;
- ignore, filter, sort, depth, expansion, collapse, scan cancellation, scan failure, scan limits, and watcher gaps never generating deletion;
- ordinary file and directory CREATE, MOVE, DELETE, COPY, and REPLACE;
- focused `:FredNew file` and `:FredNew dir` behavior;
- DELETE_TREE without descendant enumeration;
- COPY_TREE as one logical plan and a deep executor operation;
- COPY_TREE partial failure leaving an actual partial destination;
- symlink leaf behavior and no-follow scanning;
- immediate `:FredLink` with absolute and relative paths;
- hard links represented as separate entries;
- Unicode and unusual filenames;
- permission and file-lock failures;
- immutable pre-plan namespace probes repeated under the global lock;
- destination appearing between planning and execution without overwrite;
- source missing or changing kind between planning and execution;
- missing destination parent, non-directory parent, and parent traversal through a symlink;
- independent edit-base and planned-root revision mismatch rejection;
- rename swaps and case-only rename internal names;
- common post-lock finalization after final-preflight failure, success, and execution failure;
- completed-node results updating the in-memory model after success and partial failure;
- refresh failure reporting without an apply-blocking special state;
- fail-stop behavior at every executor node;
- no later operation after the first error;
- partial failure retaining completed effects and abandoning failed/unstarted initiating intent;
- cross-filesystem file and directory MOVE;
- cross-filesystem copy success followed by source deletion failure;
- process restart performing an ordinary scan of partial results.

### 23.3 Headless Neovim Tests

- build and load with every exact Neovim patch-version and pinned `nvim-oxi` revision pair listed in build configuration;
- Linux and Windows build/load coverage for every listed pair;
- `require("fred").build()` availability;
- setup configuration validation;
- command and default-mapping registration;
- focused `:FredNew file` and `:FredNew dir` commands;
- opening multiple Views of one canonical root;
- immutable edit-base snapshots in dirty Views;
- process-global mutation lock across same, unrelated, and nested roots;
- successful and failed `:FredLink` paths both releasing the global lock, invalidating and refreshing overlapping RootSessions when namespace may have changed, and allowing a later apply or `:FredLink`;
- different depth and expansion settings per View;
- union watcher coverage across Views;
- unconditional watcher establishment attempts, degraded establishment failure, and manual refresh;
- watcher-driven clean refresh and dirty namespace rebase;
- overlapping RootSession invalidation after mutation;
- `buftype=acwrite` full `:write`, `:update`, and `:wq` behavior;
- rejection of alternate-file, ranged, and append writes;
- valid non-empty plans requiring one confirmation;
- invalid/conflicted planning results opening non-confirmable diagnostics and preserving edits;
- empty plan reprojection and modified-state clearing;
- preview cancellation preserving edits;
- final-preflight failure leaving Apply and releasing the global lock while preserving dirty state;
- undo and redo before apply;
- undo baseline reset after successful apply and execution-failure model re-render;
- cursor and identity preservation after refresh;
- ignore, filter, sort, depth, collapse, cancellation, scan failure, limits, and watcher gaps never implying deletion;
- Neovim 0.11 buffer-local `p`/`P` provenance for listed 0.11 pairs;
- Neovim 0.12 put-event integration for listed 0.12 pairs;
- unobserved matching insertions remaining CREATE;
- `gp` into a directory and deterministic `_N` destinations;
- collapse preserving pinned entries;
- partial scan warnings without global apply disablement;
- large-batch rendering responsiveness;
- worker notification through `AsyncHandle` queue draining.

### 23.4 Property Tests

Randomly generated snapshots and intents must preserve:

- no duplicate final destinations;
- no buffer-planned operation outside the View root;
- no missing DAG dependencies;
- topological sortability after rename-cycle resolution;
- no directory COPY_TREE or MOVE into itself;
- all provenance copies before removed-source deletion;
- no operation after the first injected execution error;
- ignore, filter, sort, depth, expansion, collapse, scan cancellation, scan failure, limits, and watcher gaps do not change filesystem intent;
- no silent overwrite under injected destination races;
- missing or non-directory parents and symlink parent traversal always reject the affected plan;
- a scan generation publishes at most one immutable current snapshot and only after all requested scopes are terminal;
- every post-lock outcome leaves Apply and releases the global mutation lock;
- completed execution-node results deterministically produce the same in-memory namespace model;
- shared coverage equals the union of attached View requirements;
- dirty View rebase always retains its immutable edit base until rebase completes.

## 24. Performance Requirements

- Scanning and rendering must not block Neovim beyond one bounded main-thread batch.
- Root, depth, expansion, ignore, or explicit refresh changes cancel obsolete scan generations.
- Late events from cancelled generations are ignored.
- Large COPY_TREE operations run on a worker thread through the deep tree operation module.
- Provisional scan results may be displayed without becoming an authoritative dirty-View merge base.
- Stable global sorting occurs after the relevant scan generation reaches terminal state; interim order may be provisional.
- Re-rendering preserves cursor by EntryId when possible.
- Multiple Views share snapshot and watcher data rather than rescanning one canonical root independently.
- Memory use scales linearly with covered entries, retained edit-base snapshots, and visited directories.
- Snapshot versions are released when no dirty View references them.
- Version one uses no Tokio runtime and no speculative thread pool.

## 25. Implementation Sequence

1. Establish the Cargo workspace, exact Neovim patch-version and pinned `nvim-oxi` build pairs, Linux/Windows build paths, Lua `build`/load facade, setup validation, command and mapping registration, and headless Neovim test harness.
2. Implement canonical path algebra, reversible filename encoding, immutable Snapshot, RootSession/View state, runtime registry, and read-only flat rendering.
3. Implement the Rust worker channel, bounded provisional batches, cancellation, generation checks, `AsyncHandle` notification, and one atomic snapshot publication after every requested scope in a generation is terminal.
4. Add baseline depth, local expansion/collapse, coverage reference counting, ignore rules, filtering, sorting, partial-state UI, unconditional watcher establishment attempts, and degraded watcher reporting.
5. Add immutable dirty-View edit bases, namespace-only three-way rebase, overlapping RootSession invalidation, and conflict diagnostics.
6. Add EntryId extmarks, Neovim 0.11 `p`/`P` wrappers, Neovim 0.12 put-event support, `:FredNew file`, `:FredNew dir`, ordinary edit capture, provenance normalization, `gp`, and directory-row descendant hiding.
7. Implement immutable direct namespace probes, the pure planner with both revision tokens, COPY_TREE/DELETE_TREE logical operations, nested directory normalization, descendant evacuation, self-subtree rejection, replacement validation, collision checks, and cross-filesystem MOVE expansion.
8. Build `acwrite` integration, write-form rejection, non-confirmable validation/conflict diagnostics, all-or-nothing confirmation for valid plans, stale-plan checks, and repeated direct probes in final preflight.
9. Implement the process-global mutation lock, common post-lock finalization, deterministic in-memory model updates from completed nodes, serial fail-stop execution, deep platform tree operations, no-replace destinations, rename-cycle names, affected-path refresh, and partial-failure reporting.
10. Add immediate `:FredLink`, buffer-directory path resolution, and global-lock integration.
11. Run the complete regression, failure-injection, multi-view, overlap, Linux/Windows path, watcher, and large-tree suites before declaring version one stable.

Each phase must preserve read-only usability before mutation support is enabled.

## 26. Acceptance Criteria

Version one is ready when:

1. FRED builds locally as a Rust `cdylib` and loads on Linux and Windows for every exact Neovim patch-version and pinned `nvim-oxi` revision pair listed in build configuration.
2. The Lua facade exposes `build`, `setup`, `open`, `refresh`, and `apply`; setup validates configuration and registers documented commands and mappings.
3. `:Fred` opens a responsive flat View of a local root using `buftype=acwrite`.
4. Multiple Views of one canonical root share immutable snapshots while retaining independent projection and intent.
5. A scan generation publishes exactly one immutable current snapshot only after all requested scopes are terminal; partial and failed scopes remain non-authoritative in that result.
6. Dirty Views retain immutable edit-base snapshots across external namespace changes.
7. Ignore, filter, sort, depth, expansion, collapse, scan cancellation, scan failure, scan limits, and watcher gaps never imply deletion.
8. Watcher establishment is automatic; establishment failure produces a degraded warning without globally blocking unrelated valid apply operations.
9. External namespace changes refresh clean Views and rebase dirty Views within covered scope.
10. Same-named files in different directories retain distinct EntryIds.
11. Ordinary edits and `:FredNew file`/`:FredNew dir` create, move, rename, replace, and delete files and directories from final desired state.
12. Listed Neovim 0.11 pairs use buffer-local `p`/`P`, and listed 0.12 pairs use supported put events, without guessing unobserved insertion origins.
13. Source retained produces COPY; one destination with removed source produces MOVE; multiple destinations with removed source produce copies followed by source deletion.
14. `gp` copies selected top-level entries into a directory and generates deterministic `_N` names.
15. DELETE_TREE and COPY_TREE each appear as one logical operation without advance descendant enumeration.
16. Removing a directory row hides unedited descendants and evacuates explicitly moved descendants before DELETE_TREE.
17. Directory COPY_TREE and MOVE to self or a descendant are rejected.
18. Directory MOVE emits one parent move plus only explicitly required descendant operations in conditional dependency order.
19. Immutable planner probes and repeated final-preflight probes reject missing sources, wrong source kinds, occupied destinations, missing or non-directory parents, and parent traversal through symlinks.
20. Destination collision never overwrites, including a destination that appears after final preflight.
21. Invalid or conflicted planning results open non-confirmable diagnostics and preserve dirty edits; only valid non-empty plans open one all-or-nothing confirmation preview; an empty valid plan clears modified state without preview.
22. Every plan carries distinct edit-base and planned-root revisions, and stale revision, changedtick, or intent-generation checks abort before mutation.
23. Every post-lock outcome leaves Apply, processes queued refresh demand, and releases the process-global mutation lock.
24. The global lock prevents concurrent apply or `:FredLink` across same, unrelated, and overlapping roots.
25. Cross-filesystem file and directory MOVE executes as copy to the final destination followed by source deletion only after copy success.
26. The executor stops on the first execution error and runs no later operation.
27. Completed-node results update the in-memory namespace model deterministically; partial execution reports completed, failed, and unstarted operations, abandons failed/unstarted initiating-View intent, and re-renders from that model.
28. Filesystem refresh remains best effort; refresh failure reports through Neovim without adding a special state or blocking a later apply.
29. Successful and partially failed mutation invalidates and refreshes overlapping RootSessions on a best-effort basis.
30. File content, size, and mtime changes do not create FRED namespace conflicts.
31. External rename is handled conservatively without inode/file-key identity inference.
32. `:FredLink {from} {to}` executes immediately with buffer-directory relative resolution and exclusive destination creation; every success or failure releases the global lock, namespace-changing outcomes invalidate and refresh overlapping RootSessions on a best-effort basis before release, and a later apply or `:FredLink` can proceed.
33. All Rust unit, integration, property, and headless Neovim tests pass.
34. FRED runs without Oil installed and contains no Oil runtime dependency.

## 27. Residual Risks

- Filesystem races can still occur after direct final preflight begins.
- A fail-stop batch may partially complete before a system call fails.
- FRED intentionally does not undo completed operations after partial failure.
- Direct COPY_TREE failure may leave a partial final destination.
- Cross-filesystem MOVE is non-atomic and may leave both source and destination if source deletion fails.
- A process kill or power loss may leave partial state or a temporary rename name.
- Platform-specific metadata may not be preserved by copy operations.
- Linux and Windows path behavior still requires substantial automated testing.
- Other compiling platforms may expose unhandled platform-specific behavior until concrete reports arrive.
- OS watcher behavior and limits differ; degraded watcher coverage must remain visible.
- Very large repositories may require users to narrow baseline depth, local expansion, or ignore rules.
- Retained immutable snapshots increase memory use while dirty Views remain open.
- Extmark and provenance behavior under complex edits requires headless and property testing.
- `nvim-oxi` couples the native build to the exact patch-version/revision pairs listed in build configuration; every new pair requires explicit validation and may require a rebuild.

These risks are explicit constraints. The design favors a small namespace planner, direct platform operations, one global mutation gate, serial fail-stop execution, and best-effort refresh over complex guarantees for rare interruption scenarios.
