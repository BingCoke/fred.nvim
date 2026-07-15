# FRED Design Specification

**Status:** Proposed for user review  
**Date:** 2026-07-15  
**Project:** `fred.nvim`  
**Expansion:** Filesystem Representation Editor

## 1. Summary

FRED is a buffer-native file browser for Neovim. It projects a filesystem subtree into an ordinary editable buffer, lets the user express a desired filesystem state with normal text editing, then plans, previews, validates, and applies the corresponding file operations.

FRED is an independent plugin. It does not depend on, integrate with, or call Oil. It may borrow general ideas from Oil, such as editable directory buffers, extmark-backed entry identity, asynchronous rendering, action planning, and confirmation before mutation. FRED owns its scanner, state model, parser, planner, executor, recovery journal, and user interface.

FRED's default view is a flat recursive list of root-relative paths. It is not a tree, sidebar, or hierarchy widget.

Example buffer for `/home/user/project`:

```text
README.md
docs/
docs/design.md
src/
src/init.lua
src/parser.lua
tests/
tests/parser_spec.lua
```

The product description is:

> FRED is a buffer-native file browser for Neovim. Browse files, edit the filesystem.

## 2. Goals

FRED must:

1. Browse files and directories from a configurable root.
2. Display root-relative entries in a flat, editable buffer.
3. Support configurable recursion depth, filtering, sorting, and ignore rules.
4. Let ordinary text edits express file creation, directory creation, rename, move, and deletion.
5. Support explicit duplication for file and directory copy operations.
6. Track entry identity independently of its current path.
7. Treat directory rename and move as subtree operations rather than per-descendant operations.
8. Prevent hidden, filtered, unscanned, or depth-excluded entries from being interpreted as deletions.
9. Detect destination collisions and external filesystem changes before mutation.
10. Preview the complete operation plan before execution.
11. Avoid permanently deleting an existing target before its replacement is ready.
12. Record destructive execution in a recovery journal and report partial failure honestly.
13. Remain responsive while scanning and rendering large directory trees.
14. Keep the public Lua interface small and stable.

## 3. Non-Goals For Version One

Version one will not:

- depend on or interoperate with Oil;
- provide a tree, collapsible hierarchy, or sidebar layout;
- follow directory symlinks during recursive scanning;
- support SSH, S3, archive, container, or other remote backends;
- expose a public backend registration interface;
- provide full ACID transactions across multiple filesystem operations;
- support cross-filesystem directory moves;
- edit file contents inside the FRED buffer;
- modify owner, group, permissions, or timestamps;
- provide Dired-style persistent marks, arbitrary shell commands, recursive grep, archive management, or image galleries;
- silently overwrite existing destinations;
- treat filtering, sorting, ignoring, or recursion depth changes as file operations.

These can be considered later without changing FRED's core identity and state model.

## 4. Why FRED Is Independent

Oil's current model binds one buffer to one directory URL and treats that buffer's lines as the directory's direct children. Listing, cache buckets, rendering, parsing, mutation destinations, and watching all derive from that single parent URL.

A flat recursive editable view instead represents a collection of entries owned by many directories. It requires per-row canonical paths, collection-owned snapshots, multi-directory conflict checks, subtree normalization, and multi-directory refresh semantics.

Oil's maintainer described recursive display as incompatible with Oil's current architecture in [issue #26](https://github.com/stevearc/oil.nvim/issues/26#issuecomment-1379844295). [PR #305](https://github.com/stevearc/oil.nvim/pull/305#pullrequestreview-1941933152) also demonstrated that merely allowing path separators in existing entries can miss unseen destination conflicts and risk data loss.

FRED therefore borrows concepts, not runtime modules or private interfaces. If any Oil source code is copied rather than independently reimplemented, its MIT license and copyright notice must be preserved.

## 5. Core Invariants

The implementation and tests must preserve these invariants.

### 5.1 Filtering Is Not Deletion

An entry absent from the current buffer because of ignore rules, filters, sorting, recursion depth, scan limits, or scan failure remains part of the filesystem state. Absence from a projection cannot by itself generate a delete operation.

Only an entry identity that was present in the preceding editable projection and then explicitly removed by the user can generate a delete intent.

### 5.2 Path Is Not Identity

Every scanned entry receives an opaque `EntryId` for the workspace session. The ID does not derive from the path or inode. Editing a path changes an entry's desired location but not its identity.

### 5.3 Incomplete Knowledge Cannot Mutate

FRED must disable apply when the initial scan, required directory listing, destination check, or preflight validation is incomplete, cancelled, stale, or failed.

### 5.4 Existing Data Is Staged Before Replacement Or Deletion

FRED must not permanently delete an existing target and then attempt the operation that replaces it. Existing data is moved to same-filesystem staging first and removed only after the complete plan commits.

### 5.5 Watchers Are Advisory

Filesystem watchers only mark a workspace as potentially stale. Every apply performs direct preflight checks against sources, destinations, and parents.

### 5.6 A Directory Move Is A Subtree Transformation

Renaming or moving a directory generates one primary directory move. Unedited descendants are rebased in the desired model and do not generate redundant move actions.

## 6. User Experience

### 6.1 Opening FRED

```vim
:Fred [root]
```

If `root` is omitted, FRED uses the current working directory. The root is canonicalized once and stored in the workspace session. The root itself is not represented by an editable row.

Lua interface:

```lua
require("fred").setup(opts)
require("fred").open(root, opts)
require("fred").refresh(bufnr)
require("fred").apply(bufnr)
```

The buffer uses a private URI such as:

```text
fred:///percent-encoded-canonical-root
```

The URI identifies the workspace only. It is never treated as a filesystem path.

### 6.2 Buffer Syntax

Each editable line contains exactly one encoded root-relative path.

```text
README.md
src/
src/init.lua
```

Rules:

- directories end with `/`;
- files and symlinks do not end with `/`;
- paths use `/` as the logical separator on every platform;
- paths are relative to the workspace root;
- empty components, absolute paths, `.`, `..`, NUL, and normalized root escape are invalid;
- reserved bytes and control characters use a reversible percent encoding;
- type, size, modification time, status, and marks are virtual text and are not part of the editable syntax.

FRED does not indent entries or draw connector glyphs. A directory and its descendants are related by path prefixes, not visual tree structure.

### 6.3 Default Commands

```text
:Fred [root]                 open a workspace
:FredApply                   plan, preview, confirm, and execute
:FredRefresh                 refresh and merge filesystem changes
:FredDepth {n|all}           change recursion depth
:FredFilter {pattern}        set a temporary projection filter
:FredFilter!                 clear the temporary filter
:FredDuplicate               duplicate the current entry as a pending copy
:FredNew {file|dir|link}     insert an explicitly typed new entry
:FredRecover                 inspect an unfinished recovery journal
```

`:write` invokes the same behavior as `:FredApply`. It does not bypass preview, validation, or confirmation.

Suggested default mappings:

```text
<CR>    open a file; re-root on a directory
-       open the parent root after resolving dirty state
D       duplicate the current entry
R       refresh
<leader>s apply changes
q       close after resolving dirty state
```

Normal Vim editing remains the primary interface for rename, move, create, and delete.

### 6.4 Opening Entries

`<CR>` on a file opens it using normal Neovim buffer behavior.

`<CR>` on a directory re-roots the current FRED buffer to that directory. If the workspace has uncommitted edits, FRED requires the user to apply, discard, or cancel before changing roots.

Opening an escaped symlink target outside the workspace root requires confirmation. FRED never recursively scans through that link in version one.

## 7. Expressing File Operations

### 7.1 Create

A new unbound line creates an entry at its path:

```text
notes.txt        create file
assets/          create directory
```

If the entry kind cannot be inferred safely, `:FredNew` creates a typed pending row. Symlinks must be created through `:FredNew link` because their target cannot be represented by the path alone.

Missing parent directories are not created implicitly for an ordinary file row. The parent directory must already exist or have its own pending directory row. This keeps the desired namespace explicit.

### 7.2 Rename And Move

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

The destination parent must exist in the base snapshot or be created by the same plan.

### 7.3 Delete

Removing an existing bound row expresses deletion.

Removing a directory row expresses recursive deletion of the directory's remaining snapshot subtree. Descendants explicitly moved outside that subtree are evacuated first. Other descendant rows do not cancel the directory deletion merely because they remain visible before planning.

The preview must display the number of files, directories, and total known bytes affected by recursive deletion. If the subtree changed or could not be counted completely, apply is blocked until refresh.

### 7.4 Copy

Plain Vim yank and paste does not copy extmarks reliably and therefore cannot safely identify a source entry. FRED will not guess copy provenance from duplicate text.

`:FredDuplicate` creates a new pending row carrying an internal `copy_from = EntryId` token. The user edits that row's destination path. Applying it produces a copy operation.

Copying a directory recursively copies its complete scanned subtree. If the subtree is incomplete or changed after scanning, apply is blocked.

A pasted line without identity is treated as a new empty file or directory declaration, not as a copy.

### 7.5 Type Changes

Changing an existing path from `name` to `name/`, or the reverse, does not convert between file and directory. It is a validation error.

Type replacement requires explicit deletion of the old entry and creation of a new typed entry. It is treated as a destructive replacement during confirmation.

## 8. Directory Move Semantics

Consider this base state:

```text
src/
src/main.lua
src/lib/
src/lib/parser.lua
```

Renaming `src/` to `source/` produces one directory move:

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

If `src/lib/parser.lua` is edited to `parser.lua` while `src/` becomes `source/`, the child is explicitly evacuated:

```text
1. MOVE src/lib/parser.lua -> parser.lua
2. MOVE src/ -> source/
```

If a descendant is renamed but remains within the moved subtree, the parent directory move occurs first and the descendant edit uses the rebased source path afterward.

Nested directory moves are normalized from outermost to innermost. Planner output must never contain both a parent move and redundant moves for unchanged descendants.

## 9. Workspace State Model

A workspace maintains three independent layers.

### 9.1 Base

`base` is the most recent complete, immutable filesystem snapshot:

```text
Entry {
  id: EntryId,
  relpath: EncodedRelativePath,
  kind: file | directory | symlink,
  fingerprint: Fingerprint,
  link_target?: string
}
```

Fingerprint contains:

```text
kind
size
mtime_ns
optional stable file key
optional symlink target
```

### 9.2 Intent

`intent` contains captured user requests:

```text
create
copy
move
remove
```

Intent survives sorting, filtering, depth changes, and refresh conflicts. Entries with pending intent remain projected even if a filter would otherwise hide them.

### 9.3 Projection

`projection` contains the entries currently rendered after applying:

- recursion depth;
- ignore rules;
- temporary filters;
- sorting;
- scan limits;
- pending-intent visibility.

Before projection settings change, FRED captures current buffer edits into intent. Reprojection never creates filesystem intent on its own.

## 10. Identity And Extmarks

Every rendered existing row has an extmark carrying its opaque `EntryId`. Marks and statuses bind to `EntryId`, not line number.

Identity rules:

- editing text preserves identity;
- moving a line preserves identity when the extmark follows it;
- copied text does not copy identity;
- two rows cannot carry the same identity;
- if an extmark is lost, FRED may rebind only through a unique, unconsumed, unchanged base path;
- ambiguous lost identity becomes delete plus create rather than a guessed rename;
- refreshed external rename is recognized only when a stable file key is unique and has no hard-link ambiguity.

Hard links receive separate workspace identities even when they share an inode.

## 11. Scanner

The local scanner has a streaming, cancellable internal interface:

```lua
scan(request, emit_batch, finish) -> cancel
```

Request includes:

```text
root
depth
ignore matcher
follow_symlinks = false
max_entries
max_directories
generation
```

Requirements:

- emit batches of at most 500 entries;
- schedule rendering so Neovim remains responsive;
- attach a generation to every batch;
- discard late batches from cancelled generations;
- record every visited directory as complete or failed;
- never follow a directory symlink;
- exclude FRED staging paths from normal projection;
- stop at configured limits and mark the scan incomplete;
- disable apply after cancellation, error, or limit exhaustion until the user narrows scope or explicitly raises limits and completes a new scan.

Default limits:

```lua
max_entries = 50000
max_directories = 10000
render_batch_size = 500
watch_limit = 512
```

## 12. Ignore, Filter, Sort, And Depth

Ignore rules use gitignore-like ordered matching with `!` negation, directory patterns, and root anchoring. Ignoring affects scanning and projection, never deletion authority.

Temporary filters operate only on already scanned entries. Entries with intent or conflict remain visible.

Recursion depth semantics:

- `0`: entries directly under root;
- `1`: direct entries plus children of direct directories;
- `all`: no depth limit other than safety limits.

Changing ignore rules requires a new scan. Changing filter or sorting only reprojects. Changing depth may scan newly included directories or discard projected descendants, but does not delete them.

Default ordering is stable lexical ordering by normalized relative path. Platform case rules affect collision detection, not displayed spelling.

## 13. Planner

Planner is a pure module. It receives immutable inputs and returns either a complete plan or validation errors. It never mutates the filesystem.

Inputs:

```text
base snapshot
captured intent
current filesystem preflight snapshot
path and platform rules
execution policy
```

Outputs:

```text
ordered operation DAG
risk summary
affected paths
recovery requirements
validation errors
```

Operation nodes include:

```text
mkdir
create_temp_file
create_symlink
copy_to_temp
stage_existing
rename
install_temp
remove_stage
restore_stage
```

Planner must reject:

- duplicate final destinations;
- absolute or root-escaping paths;
- empty, `.`, or `..` components;
- platform-invalid paths;
- case-folding or Unicode-normalization collisions;
- destination occupied by an unchanged entry;
- destination that appeared after the base scan;
- source that disappeared or changed type;
- changed source fingerprint;
- missing or non-directory parent;
- parent traversal through symlink;
- unsupported cross-filesystem directory move;
- partial scan or incomplete subtree;
- dependency cycles that cannot be resolved with safe temporary paths.

A destination occupied by another planned entry is allowed only when that entry is successfully moved away earlier in the same plan.

Rename swaps, cycles, and case-only rename use unique same-directory temporary names.

## 14. Operation Ordering

The DAG enforces:

1. parent directory creation before child creation;
2. descendant evacuation before parent directory move or delete;
3. parent move before edits that remain inside the moved subtree;
4. child deletion or staging before directory removal;
5. temporary rename before completing a path cycle;
6. preparation of replacement content before staging the old target;
7. staging old data before installing replacement data;
8. journal persistence before every destructive step;
9. cleanup of staging only after the complete plan commits.

Independent preparation operations may run concurrently. Destructive commit operations run serially by default.

## 15. Confirmation

Applying changes opens a preview grouped by risk:

```text
CREATE
COPY
MOVE
DELETE
REPLACE
CONFLICT
```

Every row shows source, destination, kind, and relevant subtree counts. Destructive directory operations show aggregate file, directory, and byte counts.

Version one confirmation is all-or-nothing because skipping an operation can invalidate DAG dependencies. The user may cancel, edit the buffer, and apply again.

FRED requires an additional explicit confirmation for:

- recursive directory deletion;
- type replacement;
- copy or move involving more than a configurable entry count;
- any operation using compensating rather than atomic behavior.

## 16. Executor And Recovery

Executor consumes a validated plan and is the only module allowed to mutate the filesystem.

Before execution, it revalidates all source, destination, and parent fingerprints. A mismatch invalidates the plan and returns to conflict handling.

The local executor writes a JSONL write-ahead journal under:

```text
stdpath("state")/fred/transactions/
```

The journal records:

```text
transaction ID
workspace root
plan digest
each operation before execution
operation result
staging paths
compensation action
commit marker
cleanup result
```

Destructive behavior:

- deletion moves the target to a unique same-filesystem staging path;
- replacement prepares the new content first, stages the old target, then installs the new target;
- new file contents are prepared under a same-directory temporary path before installation;
- successful completion writes a commit marker before staging cleanup;
- partial failure leaves the workspace dirty and reports completed, failed, and unstarted operations;
- ambiguous recovery state never automatically deletes either side.

`:FredRecover` offers:

```text
continue
rollback
keep current filesystem state
inspect journal
```

FRED does not claim ACID guarantees. Other processes can still race with execution, and some filesystem operations cannot be reversed perfectly. The design provides preflight, staging, compensation, and explicit recovery rather than silent data loss.

## 17. Refresh And Conflicts

Refresh performs a three-way comparison:

```text
base
filesystem current
user desired
```

Rules:

- disk changed, user unchanged: accept disk state;
- user changed, disk still equals base: keep user intent;
- both changed the same identity: conflict;
- source disappeared: conflict;
- desired destination appeared: conflict;
- type changed: conflict;
- unique stable file key proves external rename: rebind identity;
- otherwise external delete/create remains conservative delete/create.

A dirty buffer is never automatically overwritten. Conflicted entries remain visible regardless of filters.

Watchers coalesce events by directory and mark the workspace stale. Up to `watch_limit`, FRED may watch scanned directories individually. Above that limit, it disables additional watchers and relies on manual refresh plus apply preflight.

## 18. Symlink Policy

Version one uses:

```lua
follow_symlinks = false
```

FRED lists, opens, renames, copies, and deletes the link itself. It never recursively traverses the target.

This prevents:

- cycles;
- root escape through aliases;
- the same subtree appearing under multiple paths;
- ambiguous ownership and duplicate mutation.

A later read-only follow mode may use stable directory keys and a visited set, but writable recursive traversal through symlinks is outside this specification.

## 19. Internal Modules

Suggested module structure:

```text
lua/fred/init.lua
lua/fred/config.lua
lua/fred/workspace.lua
lua/fred/scanner.lua
lua/fred/model.lua
lua/fred/identity.lua
lua/fred/path.lua
lua/fred/projection.lua
lua/fred/view.lua
lua/fred/edit.lua
lua/fred/planner.lua
lua/fred/conflict.lua
lua/fred/confirmation.lua
lua/fred/executor.lua
lua/fred/journal.lua
lua/fred/recovery.lua
lua/fred/ignore.lua
lua/fred/backend/local.lua
plugin/fred.lua
```

Module interfaces should remain internal except for `lua/fred/init.lua`. No speculative backend plugin interface is exposed in version one.

## 20. Public Configuration

Initial configuration shape:

```lua
require("fred").setup({
  depth = "all",
  follow_symlinks = false,
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
    watch_directories = 512,
  },
  confirmation = {
    always = true,
    recursive_delete = true,
    large_operation_entries = 1000,
  },
  mappings = {
    open = "<CR>",
    parent = "-",
    duplicate = "D",
    refresh = "R",
    apply = "<leader>s",
    close = "q",
  },
})
```

Configuration is validated during setup. Invalid safety settings fail closed rather than silently falling back.

## 21. Error Handling

Errors fall into four classes:

### Scan Error

The buffer remains readable but apply is disabled. The status identifies failed directories and offers refresh or scope reduction.

### Validation Error

No filesystem action runs. Errors are attached to affected rows and listed in a location list.

### Conflict

No filesystem action runs. Conflicted entries remain visible and require refresh, rebase, or explicit user edits.

### Execution Failure

The executor stops at the dependency-safe point, persists the journal, leaves the buffer dirty, rescans affected paths when possible, and directs the user to `:FredRecover`.

Errors must never be converted into a clean buffer state unless the filesystem matches the desired state after verification.

## 22. Testing Strategy

### Unit Tests

- path encoding and decoding;
- normalization and root escape rejection;
- Windows drive, UNC, case-folding, and reserved-name behavior;
- EntryId and extmark rebinding;
- filtering and depth changes never generating deletion;
- new row, move, delete, duplicate, and lost identity parsing;
- directory subtree rebase;
- descendant evacuation;
- nested directory moves;
- duplicate targets and destination occupancy;
- rename swaps and case-only renames;
- DAG topological ordering;
- ignore rule negation and directory traversal semantics;
- three-way refresh and conflict classification.

### Integration Tests

- recursive local scanning;
- ordinary file CRUD;
- directory CRUD;
- directory copy and recursive delete;
- symlink loops and root escape;
- hard links;
- Unicode and unusual filenames;
- permission failures;
- destination appearing between scan and apply;
- source changing between preview and execution;
- staging, rollback, and cleanup;
- failure injection at every executor node;
- crash recovery from every journal phase;
- cross-filesystem directory move rejection.

### Headless Neovim Tests

- opening and closing workspaces;
- `:write` invoking apply;
- undo and redo before apply;
- cursor and identity preservation after refresh;
- filters with a dirty buffer;
- copied text not copying identity;
- `:FredDuplicate` preserving copy provenance;
- confirmation cancellation;
- incomplete scan disabling apply;
- large-batch rendering responsiveness.

### Property Tests

Randomly generated snapshots and intents must preserve:

- no duplicate final destinations;
- no operation outside root;
- no missing DAG dependencies;
- topological sortability after temporary-cycle resolution;
- no pre-existing data permanently lost after an injected failure when a compensation path exists;
- filtering and sorting do not change filesystem intent.

## 23. Performance Requirements

- Scanning and rendering must not block Neovim for more than one scheduled batch.
- Root or depth changes cancel prior scan generations.
- Late callbacks from cancelled scans are ignored.
- Large copies, hashes, and subtree counts run outside the main UI loop.
- Provisional scan results may be displayed, but apply remains disabled until scanning completes.
- Stable global sorting occurs after complete scan; interim order may be provisional.
- Re-rendering must preserve cursor by EntryId when possible.
- Memory use scales linearly with scanned entries and visited directories.

## 24. Implementation Sequence

1. Establish the Lua plugin skeleton, configuration validation, commands, and headless test harness.
2. Implement path algebra, local scanner, immutable base snapshots, and read-only flat rendering.
3. Add EntryId extmarks, projection state, filters, depth changes, and cancellation.
4. Implement edit capture for create, move, and delete with a dry-run planner.
5. Add directory subtree normalization, descendant evacuation, and collision validation.
6. Add explicit duplication and copy planning.
7. Build confirmation UI and complete preflight validation.
8. Implement journaled local executor, staging, compensation, and recovery.
9. Add watchers, three-way refresh, and conflict UI.
10. Run the complete regression, failure-injection, and large-tree test suites before declaring version one stable.

Each phase must preserve read-only usability even before mutation support is enabled.

## 25. Acceptance Criteria

Version one is ready when:

1. `:Fred` opens a responsive flat recursive view of a local root.
2. Same-named files in different directories retain distinct identities.
3. Ordinary edits can safely create, move, rename, and delete files and directories.
4. `:FredDuplicate` can copy files and complete directories without guessing source identity.
5. Directory rename emits one parent move plus only explicitly required descendant operations.
6. Filters, ignores, depth changes, partial scans, and watcher limits never imply deletion.
7. Every apply shows a complete plan and reruns preflight before mutation.
8. Existing destination data is staged before destructive replacement.
9. Failure injection demonstrates journaled recovery without silent clean-state claims.
10. Unsupported remote and cross-filesystem directory operations fail closed.
11. All unit, integration, property, and headless Neovim tests pass.
12. FRED runs without Oil installed and contains no Oil runtime dependency.

## 26. Residual Risks

- Filesystem races can still occur after preflight begins.
- Cross-platform path rules, especially Windows case and Unicode behavior, require substantial testing.
- Recovery cannot perfectly compensate for every external modification made during execution.
- Very large repositories may require users to narrow depth or ignore rules.
- Extmark behavior under complex user edits needs headless and property testing.
- Recursive directory copy and delete magnify the impact of incomplete scans, making completeness checks mandatory.
- A future remote backend may not support the same staging and recovery guarantees and must not silently weaken them.

These risks are explicit constraints, not reasons to reuse Oil's incompatible single-directory internals.
