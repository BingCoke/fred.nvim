# FRED Design Specification

**Status:** Accepted direction; implementation pending
**Date:** 2026-07-15
**Last revised:** 2026-07-17
**Project:** `fred.nvim`
**Expansion:** Filesystem Representation Editor

## 1. Summary

FRED is a buffer-native file browser for Neovim. It projects selected parts of a local filesystem root into an ordinary editable buffer. The user expresses a desired namespace with normal text editing, then writes the buffer to plan, preview, confirm, and execute the corresponding filesystem operations.

FRED is independent. It does not depend on, integrate with, or call Oil. It may borrow general ideas such as editable directory buffers, entry identity, asynchronous rendering, operation planning, and confirmation before mutation, but it owns its scanner, snapshots, views, planner, executor, watcher integration, and user interface.

FRED uses a required Rust `cdylib` built locally by the user or plugin manager. The filesystem core is Rust-first and uses `nvim-oxi` for snapshots, identity, scanning, intent, planning, execution, and keyed Neovim buffer reconciliation. Lua owns the stable Instance facade, resolved configuration, actions, keymaps, attach callbacks, registry lookup, native Neovim layouts, lifecycle autocmds, and presentation policy: it filters hidden files and sorts the bounded catalog supplied by Rust, then submits only an ordered visible EntryId sequence for Rust to render. FRED supports only explicitly validated Neovim 0.12 patch-version and pinned `nvim-oxi` revision pairs. It does not publish precompiled binaries or provide a pure-Lua fallback.

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
4. Support sorting, hidden-file controls, ignore rules, bounded scanning, and partial views.
5. Let ordinary text edits express file and directory creation, rename, move, copy, replacement, and deletion.
6. Capture copy provenance from ordinary Vim yank and put operations where Neovim exposes or FRED wraps the operation.
7. Track entry identity independently of its current path.
8. Treat directory copy, move, and deletion as single logical tree operations rather than per-descendant plans.
9. Prevent hidden, ignored, unscanned, collapsed, or depth-excluded entries from being interpreted as deletions.
10. Share immutable filesystem snapshots and watcher coverage between multiple views of the same canonical root while preserving per-view intent and projection state.
11. Detect external namespace changes and update attached views automatically within materialized coverage.
12. Reject invalid paths, missing sources, wrong source kinds, required parents that neither exist nor are created by the same plan, and occupied destinations before mutation.
13. Never silently overwrite an existing destination.
14. Preview every non-empty operation plan and require one all-or-nothing confirmation.
15. Permit only one FRED mutation anywhere in the Neovim process at a time.
16. Execute filesystem operations serially and stop immediately on the first execution error.
17. Refresh affected views after success or failure on a best-effort basis.
18. Remain responsive while scanning, copying, and rendering large directory sets.
19. Expose a stable Lua Instance lifecycle with explicit actions, buffer metadata, attach callbacks, and layout-independent display control.
20. Build from source for validated Neovim and `nvim-oxi` combinations.
21. Render optional metadata columns such as icons, permissions, size, and timestamps without changing editable path syntax.
22. Bound in-memory scan and metadata cache retention with explicit time, memory, and entry-count cleanup policies.
23. Let one instance own one editable buffer while displaying that buffer in multiple windows and sharing RootSession data by reference.
24. Let users create named or unnamed instances, inherit configuration, choose action functions per keymap mode, and reveal files through normal flat expansion.
25. Automatically destroy undisplayed instances after their configured delay while discarding only unapplied buffer state and never mutating the filesystem.

## 3. Non-Goals For Version One

Version one will not:

- depend on or interoperate with Oil;
- provide an indented tree, connector glyphs, or a permanent sidebar layout;
- recursively follow directory symlinks;
- support SSH, S3, archives, containers, or other remote backends;
- expose a public backend registration interface;
- edit file contents inside the FRED buffer;
- use file-content hashes, size, or modification time as namespace identity or conflict versions (size and timestamps may still be displayed as metadata columns);
- infer external rename identity from inodes or platform file keys;
- provide ACID semantics across a batch of filesystem operations;
- automatically retry or undo a partially executed batch;
- provide a persistent transaction journal, rollback command, or crash recovery;
- provide a filesystem-level undo command;
- publish precompiled native binaries;
- support Neovim 0.11 or earlier, nightly builds, or unlisted Neovim 0.12 patch releases;
- guarantee preservation of every platform-specific metadata feature during copy;
- guarantee atomic cross-filesystem moves;
- provide Dired-style persistent marks, arbitrary shell commands, recursive grep, archive management, or image galleries;
- silently overwrite an existing destination;
- provide a generic temporary text-filter feature; hidden-file visibility uses `view.hidden_files_visible` and `is_hidden_file` instead;
- treat sorting, hidden-file display, ignoring, expansion, collapse, layout, reveal, or navigation changes as filesystem operations.

These constraints keep version one centered on paths, identity, immutable snapshots, views, desired-state planning, and a small fail-stop executor.

## 4. Why FRED Is Independent

Oil's current model binds one buffer to one directory URL and treats that buffer's lines as the direct children of one directory. Listing, cache ownership, rendering, parsing, mutation destinations, and watching derive from that single parent URL.

A flat editable view represents entries owned by many directories and may materialize different subdirectories to different depths. It requires per-row root-relative paths, shared root snapshots, multi-directory collision checks, subtree normalization, selective watcher coverage, and multi-view refresh semantics.

Oil's maintainer described recursive display as incompatible with Oil's current architecture in [issue #26](https://github.com/stevearc/oil.nvim/issues/26#issuecomment-1379844295). [PR #305](https://github.com/stevearc/oil.nvim/pull/305#pullrequestreview-1941933152) also demonstrated that merely allowing path separators in existing entries can miss unseen destination conflicts and risk data loss.

FRED therefore borrows concepts, not runtime modules or private interfaces. If any Oil source code is copied rather than independently reimplemented, its MIT license and copyright notice must be preserved.

## 5. Core Invariants

The implementation and tests must preserve these invariants.

### 5.1 Buffer Intent Mutates Only After Confirmed Write

Editing, deleting, putting, expanding, collapsing, sorting, changing hidden-file visibility, refreshing, revealing, navigating, hiding, or destroying a FRED buffer does not mutate the filesystem by itself.

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

An entry absent from the current buffer because of ignore rules, Lua hidden-file filtering, Lua sorting, baseline depth, local collapse, scan limits, scan cancellation, scan failure, or missing watcher coverage remains part of filesystem state.

Lua submits only a generation-bound ordered sequence of visible EntryIds. Rust validates that sequence and treats every omitted catalog EntryId as projection absence, never as deletion intent. Only user removal of a bound identity from the editable projection generates delete intent. Rust independently preserves dirty intent, conflict, and validation-error rows while reconciling the ordinary browse order. Metadata columns are projection-only: a missing, stale, or failed metadata value cannot create, remove, rename, or delete intent.

### 5.3 Path Is Not Identity

Every scanned entry receives an opaque `EntryId` scoped to its shared RootSession snapshot lineage. The ID does not derive from the displayed path, inode, or platform file key. Editing a path changes an entry's desired location but not its identity.

### 5.4 Direct Namespace Checks Authorize Operations

A partial or degraded browse snapshot does not globally disable apply. Before planning, the runtime captures immutable direct namespace-probe results for every source, destination, existing required parent, and planned-parent chain required by the current intent. For a parent created by the same plan, probes establish that the planned path is either absent or occupied by a base identity that the plan explicitly removes or moves away before creating the directory; an occupied path must still match the base identity's expected path and kind under FRED's ordinary path/kind checks. Probes also establish that the chain's nearest existing ancestor is a directory reached without traversing a symlink. The pure planner consumes those probe results without performing I/O. Final preflight repeats the same probes under the process-global mutation lock.

An operation is blocked when its required namespace state cannot be established, including a missing source, wrong source kind, required parent that neither exists nor is created by the plan, non-directory parent or nearest existing ancestor, parent traversal through a symlink, invalid path, or occupied destination. A parent that the same valid plan creates is not rejected merely because it is currently absent.

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

### 5.10 Instance Destruction Discards Only Unapplied Buffer State

An instance may lose every displaying window while its buffer remains loaded and modified. That transition starts the instance cleanup delay; it does not apply, discard, confirm, or mutate the filesystem. The user may reopen the buffer and apply before the timer expires.

When explicit destruction, external buffer wipeout, or the cleanup timer destroys the instance, FRED discards that buffer text, View intent, undo history, and edit-base reference. Shared RootSession data is released by reference counting and remains alive while any other instance or task requires it. Destroying unapplied text is not a filesystem operation and never enters planner or executor flow.

One instance owns one buffer and View. Multiple windows may display that buffer, and the cleanup timer cannot run while any window still displays it.

## 6. Platform, Build, And Runtime Architecture

### 6.1 Supported Neovim And Platform Combinations

FRED supports only exact Neovim 0.12 compatibility pairs listed in the build configuration. Each pair contains:

- one exact Neovim 0.12 patch version;
- one pinned `nvim-oxi` revision using its `neovim-0-12` feature.

The build helper accepts only a listed pair and rejects every unlisted patch/revision combination rather than guessing a feature selection. Neovim 0.11 and earlier, nightly builds, and unlisted 0.12 patches are unsupported. The first candidate is Neovim 0.12.4, but it becomes a supported pair only after the FRED compatibility harness validates every direct Neovim call used by the internal `nvim_adapter`, including buffer access, range extmarks, inline virtual text, changedtick access, and main-thread notification.

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

Upgrading Neovim or the pinned `nvim-oxi` revision may require rebuilding FRED. The Lua facade may load far enough to expose `build()` and an actionable diagnostic, but `fred.new()` fails when the required compatible native library is missing or cannot load. There is no degraded pure-Lua browse mode or precompiled-binary fallback.

### 6.3 Rust-First Runtime

Rust owns:

- canonical paths, immutable snapshots, RootSession, View, identity, provenance, intent, conflict, and validation state;
- scanning, metadata collection, watching, coverage, planning, execution, refresh, and shared cleanup accounting;
- bounded catalog batches for every entry in the View-requested scopes, including entries that Lua may classify as hidden;
- generation/revision-bound one-shot order validation and keyed buffer, range-extmark, virtual-text, cursor, and dirty-row reconciliation;
- buffer parsing, direct namespace probes, and the process-global mutation gate through one internal `nvim_adapter` for every direct Neovim call;
- worker tasks, generation checks, and main-thread notification.

Lua owns:

- build integration and native-library discovery/loading;
- `setup()` configuration capture and direct/child instance inheritance;
- the public Instance object and live-instance registry lookups;
- action functions, mode-grouped buffer-local keymaps, FileType metadata, and `on_attach`;
- native Neovim layout construction, window ownership, open/hide/toggle/focus behavior, highlight definitions, and Neovim lifecycle autocmds;
- per-instance presentation state and the current View catalog mirror;
- `is_hidden_file` evaluation, hidden-directory subtree propagation, stable single-key sorting, and submission of ordered visible EntryIds.

The boundary passes opaque InstanceId/ViewId handles, bounded generation-tagged EntryFacts, validated presentation values, a sealed `CatalogFrameToken`, atomic ordered-EntryId submissions, buffer/window identifiers, and explicit operation results. Lua submits either one `{ token, ordered_visible_ids }` value or nothing; it never submits rendered text, extmark positions, metadata decorations, or filesystem intent. Rust validates exact token currency, EntryId membership, uniqueness, and transaction atomicity. Any unique subset of the frame's EntryIds is a valid projection, so Rust does not claim to detect a semantically mistaken Lua omission. Rust remains authoritative for identity, planning, deletion intent, protected dirty rows, and rendering. Lua does not reimplement filesystem planning, and Rust does not execute user callback tables. Rust modules outside `nvim_adapter` do not call `nvim_oxi::api` directly; this keeps the exact-patch compatibility surface explicit and testable.

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

Worker threads never call Neovim functions directly. They enqueue bounded events and use `AsyncHandle::send()` only to wake the main loop. Because wakeups may coalesce, each main-thread callback drains only one entry-count-limited or time-limited batch. If events remain, it calls `AsyncHandle::send()` again before returning. A producer that continues filling the bounded channel therefore cannot turn one callback into an unbounded main-thread drain.

Parallelism may be added later only where benchmarks demonstrate a real bottleneck.

## 7. User Experience

### 7.1 Public Instance API

FRED exposes reusable instances rather than making a module-level `open()` call the primary lifecycle boundary. `:Fred [root]` remains a convenience command that creates an unnamed instance and opens it.

```lua
local fred = require("fred")

fred.build()
fred.setup(opts)

local explorer = fred.new({
  name = "project_files",
  profile = "project",
  root = "/project",
  layout = "float",
})

explorer:open()
```

The stable Lua surface includes:

```lua
fred.new(opts)
fred.get_instance(instance_id)
fred.get_instance_by_name(name)
fred.get_instance_by_buf(bufnr)
fred.actions
```

Each instance exposes:

```lua
instance:id()
instance:name()
instance:state()
instance:root()
instance:initial_root()
instance:config()
instance:bufnr()
instance:windows()

instance:new(opts)
instance:open(opts)
instance:hide(opts)
instance:toggle(opts)
instance:focus(opts)
instance:destroy()

instance:refresh()
instance:apply()
instance:reveal(relative_path, opts)
instance:reveal_current_file(opts)
instance:navigate(relative_directory)
instance:relative_path(absolute_path)
instance:contains(absolute_path)

instance:set_columns(columns)
instance:hidden_files_visible()
instance:set_hidden_files_visible(value)
instance:toggle_hidden_files()
instance:sort()
instance:set_sort(sort_spec)
```

`root` is resolved and canonicalized during `new()`, not during `open()`. If omitted, direct `fred.new()` uses the current working directory at that moment, while `parent:new()` uses `parent:root()`, including a root changed by `navigate()`. `open()` accepts display options such as `layout`, but never silently changes the instance root.

A direct `fred.new(opts)` resolves configuration as:

```text
latest setup configuration
  + explicit new options
```

`instance:new(opts)` creates a child instance and resolves configuration as:

```text
parent instance resolved configuration
  + snapshot of the parent's current hidden_files_visible and sort
  + explicit child options
```

The current presentation values become part of the child's immutable construction snapshot; later parent changes do not update the child. Explicit child options override the copied values. SortSpec is an explicit exception to every deep-merge rule: a setup SortSpec, direct-instance override, copied parent-current SortSpec, or explicit child override is independently validated as a table containing exactly `by`, `direction`, and `case_sensitive`, and the selected table replaces the previous SortSpec wholesale. No SortSpec field is inherited from another table. This inheritance keeps columns, keymaps, layout, cleanup, presentation behavior, and callbacks consistent while navigating through newly created instances. The inherited `on_attach` list already contains setup callbacks and is not prefixed with setup callbacks a second time.

`name` is optional. Every instance has an opaque unique InstanceId; a non-empty name must be unique among live instances. Creating a second live instance with the same name is an explicit error. Destroying the old instance releases the name for reuse. Automatically created child instances are unnamed unless the action supplies a name.

The resolved configuration is immutable after `new()`. `instance:config()` returns a read-only construction snapshot; `instance:hidden_files_visible()` and `instance:sort()` return read-only copies of committed runtime presentation state. Runtime changes use explicit methods such as `set_columns`, `set_hidden_files_visible`, `set_sort`, `navigate`, and per-call `open({ layout = ... })`; FRED does not expose a generic `set_config()` or `update_config()`.

In Created state, presentation setters only validate and update Lua PresentationState. There is no View or catalog yet, so they perform no intent capture, metadata request, ordered-ID submission, buffer synchronization, or history reset. `open()` creates the View from those values. Open and lifecycle-Hidden instances use the transaction rules in Section 13. Destroyed instances reject presentation operations through the ordinary lifecycle guard.

### 7.2 Instance, Buffer, Window, And Lifecycle Model

One instance owns at most one FRED buffer and one View, while that buffer may be displayed by zero or more windows across any number of tabpages:

```text
Instance
  InstanceId
  resolved configuration
  initial_root
  current_root
  buffer: zero or one
  View: zero or one
  windows: zero or more
  cleanup timer
```

Multiple instances may point at the same canonical root. They own distinct buffers, Views, intent, undo history, columns, expansion state, and Lua presentation state while sharing the root's immutable snapshots, scan cache, metadata cache, and watcher coverage through RootSession references.

The public lifecycle states are:

```text
Created
  instance exists; root and configuration are resolved; buffer does not exist

Open
  buffer exists and at least one window displays it

Hidden
  buffer exists and no window displays it; instance cleanup timer may run

Destroyed
  terminal state; buffer, View, windows, timers, name registration, and RootSession references are released
```

Transitions are driven by observed Neovim state as well as instance methods:

```text
new()                         -> Created
open()                        -> Open
last displaying window gone  -> Hidden
open() while Hidden           -> Open
cleanup delay expires         -> Destroyed
buffer delete or wipeout      -> Destroyed
destroy()                     -> Destroyed
```

Fred uses `WinNew`, `BufWinEnter`, `WinEnter`, `WinClosed`, and `BufHidden` as display-reconciliation triggers; event-local counters are not authoritative. Reconciliation derives displays from valid windows that actually contain the instance buffer. Because `WinClosed` fires before the closing window disappears from window queries, its reconciliation is deferred to the next main-loop turn. Any reconciled window displaying the buffer cancels the cleanup timer, and the timer starts only after reconciliation finds no display. A validated `BufDelete` or `BufWipeout` for the instance's current buffer bypasses display reconciliation and invokes the terminal destroy path directly.

Instance cleanup is configured per instance:

```lua
cleanup = {
  instance = {
    delay_ms = 300000,
  },
}
```

`delay_ms` is not keyboard-idle time. It is the delay after the buffer ceases to be displayed anywhere. Reopening the buffer cancels the timer. `delay_ms = 0` disables automatic instance destruction.

When the timer expires, FRED calls the instance's terminal destroy path. Unapplied buffer edits and intent are discarded without mutating the filesystem. FRED does not prompt, auto-apply, or block navigation merely because the old instance buffer is modified: the user may return to that buffer and apply before destruction, or allow the instance to expire and discard the edits. Shared snapshots and cache data remain protected while the live dirty View references them; destroying the instance intentionally drops that View and releases its references.

A forced external buffer deletion has the same discard semantics. This is not filesystem undo or recovery: text that was never successfully applied simply ceases to exist with the destroyed buffer.

Each View buffer uses a private URI containing the canonical current root and opaque instance/View identity, for example:

```text
fred:///percent-encoded-canonical-root?instance=opaque-id&view=opaque-id
```

FRED buffers use:

```text
buftype=acwrite
swapfile=false
filetype=fred
```

### 7.3 Layouts And Display Ownership

The instance option is named `layout`; `mode` is reserved for Neovim keymap modes such as `n`, `i`, and `v`.

```lua
local explorer = fred.new({
  layout = "float",
})

explorer:open()
explorer:open() -- focuses an existing display in the current tabpage
explorer:open({ layout = "vsplit", new_display = true })
```

`new({ layout = ... })` sets the instance default. If the current tabpage already displays the instance buffer, `open()` focuses one such window and does not create another display; a per-call `layout` is ignored in that case. If the current tabpage has no display, `open({ layout = ... })` creates one there even when another tabpage displays the same buffer. `open({ new_display = true, layout = ... })` explicitly creates an additional display. Per-call options do not mutate resolved configuration.

Version one constructs every layout directly with native Neovim Lua APIs. The internal `layout` module returns ordinary buffer/window ownership records plus hide, restore, and focus operations. It does not expose a speculative adapter interface or depend on NUI. Detailed visual styling is not part of the core lifecycle contract.

Initial layouts are:

```text
buffer
float
split
vsplit
tab
```

Window ownership rules are:

- `buffer` borrows the current window. Hiding restores the previous buffer, then the alternate buffer, then a scratch buffer; it never deletes the borrowed window.
- `float` owns the created floating window. Hiding closes the float and retains the instance buffer.
- `split` and `vsplit` own the created split window. Hiding closes that split when safe; if external layout changes made it the last ordinary window in the tabpage, FRED restores a previous/alternate/scratch buffer instead.
- `tab` owns its created tabpage. Hiding closes that tabpage when another tabpage exists; if it is the final tabpage, FRED restores a previous/alternate/scratch buffer in the last ordinary window.
- FRED never forcibly deletes the final ordinary window merely to hide an instance.

`instance:hide()` hides the instance from the current window only. `instance:hide({ winid = ... })` targets one explicit window. Calling `hide()` when the current window does not display the instance is an error rather than a guess. No public hide-all or ambiguous `close()` method exists.

`instance:toggle(opts)` is a small current-tab wrapper:

```text
find every window in the current tabpage that displays this instance buffer
if one or more exist: hide all of those current-tab displays
otherwise: open(opts), using opts.layout or the instance default
```

This internal multi-window hide is specific to `toggle`; other tabpages are unaffected and no public hide-all API is added.

The default directory-selection action creates a child instance, creates its buffer, and transfers the triggering Fred window to that child buffer instead of stacking another float or split. The parent loses that window, becomes Hidden if no other window displays it, and starts its cleanup timer. Window buffer replacement uses normal Neovim behavior; FRED does not maintain a separate back/forward history or override native buffer, alternate-buffer, or jumplist behavior.

### 7.4 Filetype And Buffer Metadata

Before setting `filetype=fred`, FRED populates one namespaced buffer-local metadata table:

```lua
vim.b[bufnr].fred = {
  instance_id = "opaque-id",
  name = "project_files",
  profile = "project",
  view_id = "opaque-id",
  root = "/project",
  layout = "float",
  state = "open",
  root_revision = 12,
  metadata_revision = 4,
}
```

The table contains stable metadata, not the instance object. Lua tables read through `vim.b` do not preserve instance metatables as an object identity boundary. Users retrieve the live object through `fred.get_instance()`, `fred.get_instance_by_name()`, or `fred.get_instance_by_buf()`.

`filetype` is always exactly `fred`. Custom instance categories use optional `name` and `profile` metadata instead of creating filetypes such as `fred_project`. This keeps one `ftplugin/fred.lua`, syntax definition, and FileType integration surface.

Initialization order is observable and fixed:

```text
1. create the buffer and View
2. populate vim.b[bufnr].fred
3. apply buffer-local options
4. install resolved buffer-local keymaps
5. set filetype=fred and run FileType autocmds
6. run setup-derived on_attach callbacks
7. run instance-specific on_attach callbacks
8. begin initial scan and rendering
```

This order lets FileType autocmds inspect instance metadata and override configured mappings, while instance-specific attach callbacks retain final precedence.

### 7.5 Attach Callbacks

`on_attach` accepts either one function or a list of functions. FRED normalizes both forms to a list. Ordinary deep table merge is not used for callback lists because `vim.tbl_deep_extend("force", ...)` replaces list elements instead of concatenating them.

For direct instances, callbacks run in this order:

```text
setup.on_attach
then new.on_attach
```

For child instances, the parent's already-resolved callback list is inherited and explicit child callbacks are appended. Setup callbacks are not duplicated.

Each callback runs once when the instance creates its single buffer. Opening more windows for the same buffer does not run attach again.

```lua
on_attach = function(ctx)
  local instance = ctx.instance
  local bufnr = ctx.bufnr
  local actions = ctx.actions
  local config = ctx.config
end
```

The callback context contains:

```lua
{
  instance = instance,
  instance_id = instance:id(),
  bufnr = bufnr,
  actions = require("fred").actions,
  config = instance:config(),
}
```

Attach callbacks are fail-fast. The first callback error stops later callbacks, rolls back windows/buffer/View/timers/RootSession references created for the instance, marks the instance Destroyed, and rethrows the original error to the caller of `open()`.

### 7.6 Buffer Syntax And Canonical Path Encoding

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

#### 7.6.1 Metadata Columns

FRED borrows the useful shape of Oil's column registry and column specifications, while keeping its own flat path-list model. The reference implementation is Oil's [`columns.lua`](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/lua/oil/columns.lua) and [`oil-columns` documentation](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/doc/oil.txt#L412-L494). FRED does not copy Oil runtime code or depend on Oil.

The editable buffer line remains exactly one encoded root-relative path. Columns are rendered as Neovim extmark virtual text at byte column zero with `virt_text_pos = "inline"`; they are not inserted into the buffer line, are not parsed by the planner, and cannot be mistaken for a path component. The path is always the final visible field. EntryId remains an extmark and is never rendered as editable text.

The initial built-in columns are:

- `icon`: a stable icon for file, directory, symlink, and other entry kinds; the built-in provider has kind fallbacks and accepts configured icon strings;
- `permissions`: a platform permission string such as `rwxr-xr-x`; unavailable permissions render `-` and never fail a scan;
- `size`: the entry's byte size, formatted compactly; directory size is not recursively calculated and renders `-` unless directly available;
- `mtime`: last modification time;
- `ctime`: change/status time where the platform exposes it;
- `atime`: last access time where the platform exposes it;
- `birthtime`: creation time where the platform exposes it;
- `type`: the normalized entry kind, useful for display or sorting.

`mtime` is the default timestamp column. `ctime`, `atime`, and `birthtime` are opt-in because their meaning and availability differ across platforms. Version one columns are read-only metadata. In particular, the permissions column does not turn a virtual text edit into `chmod`; a future explicit action may add that behavior without changing buffer path syntax.

Column specifications follow the Oil-like string-or-table shape:

```lua
columns = {
  "icon",
  "permissions",
  { "size", align = "right" },
  { "mtime", format = "%Y-%m-%d %H:%M", width = 16 },
}
```

With that configuration, the screen may look like this; actual glyphs depend on the configured icon strings:

```text
  rwxr-xr-x       384  2026-07-16 10:22  .git/
  rw-r--r--     81.5k  2026-07-16 10:22  2026-07-15-fred-design.md
```

The actual buffer text remains:

```text
.git/
2026-07-15-fred-design.md
```

FRED displays neither Oil's concealed `/NNN` internal-ID prefix nor a `../` parent row. EntryId stays in an extmark, and the View root is intentionally not represented by a row.

A string selects defaults. A table uses the first element as the column name and may set `width`, `align` (`"left"`, `"center"`, or `"right"`), `format` for time columns, `highlight`, `separator`, and column-specific icon options. The path column is implicit and cannot be removed.

The default separator is one display cell. Default minimum display widths are two cells for `icon`, nine for `permissions`, eight for `size`, nine for `type`, and the formatted display width for each timestamp. Missing values use `-` and the same padding/alignment as present values. `width` is a minimum measured with Neovim display width, not byte length; a wider value expands that column for the whole projection rather than being truncated. All rows in one projection therefore use the same stable boundaries. This follows Oil's projection-wide width/alignment pass in [`view.lua`](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/lua/oil/view.lua#L650-L818), adapted to virtual text rather than editable prefix text.

The internal Rust definition is intentionally small:

```text
ColumnDefinition {
  name
  required_metadata
  default_width
  default_alignment
  render(entry, metadata, column_options) -> highlighted text chunks
}
```

Version one registers only built-in columns; it does not expose a public custom-column or backend registration API. Unlike Oil, FRED column definitions do not parse mutable column text because only the path exists in the buffer syntax.

Each column definition declares the metadata it needs. `icon` and `type` use already-known entry kind; `permissions`, `size`, and time columns request stat metadata. Lua's current SortSpec separately declares the one sort field it needs. A RootSession requests the union of metadata fields required by attached View columns and Lua presentation states, just as coverage is the union of their directory scopes. This follows Oil's on-demand `require_stat` approach in [`files.lua`](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/lua/oil/adapters/files.lua#L50-L225). Metadata failures render a placeholder and a non-blocking status rather than making a namespace scan authoritative or failed. Refreshing metadata may re-render clean and dirty Views, but changes to permissions, size, or timestamps never create FRED namespace conflicts.

Changing columns only reprojects the View and requests any missing metadata. It never changes dirty state, edit-base revisions, or filesystem intent. Lua sorting may use `path`, `type`, `size`, `mtime`, `ctime`, `atime`, or `birthtime`; unavailable metadata always sorts after available values and does not block apply.

A metadata revision may cause Lua to submit a new order only for a clean View. A dirty View updates cached metadata values and virtual columns but retains its frozen full-catalog order. When it becomes clean after apply, successful empty-plan synchronization, or ordinary undo removes its intent, Lua recomputes from current metadata and the already-current SortSpec; no deferred requested sort exists. A SortSpec or metadata change that actually changes real-row order clears Neovim undo history and establishes a new baseline. If real-row order is unchanged, presentation state may still advance but history is not cleared.

### 7.7 Actions, Keymaps, And Commands

Keymaps use a Telescope-like table grouped by Neovim mode. Mapping values are functions or `false`; FRED never interprets action-name strings or command strings.

```lua
local actions = require("fred").actions

require("fred").setup({
  keymaps = {
    n = {
      ["<CR>"] = actions.select({
        file = actions.open_file,
        directory = actions.open_new_instance,
        symlink = actions.open_file,
      }),
      ["-"] = actions.open_parent_instance,
      ["zo"] = actions.expand,
      ["zc"] = actions.collapse,
      ["zO"] = actions.expand_recursive,
      ["zC"] = actions.collapse_recursive,
      ["R"] = actions.refresh,
      ["<leader>s"] = actions.apply,
      ["q"] = actions.hide,
      ["g."] = actions.toggle_hidden_files,
    },
    i = {},
    v = {
      ["gp"] = actions.paste_into,
    },
  },
})
```

Keymaps merge by `mode + lhs`. An absent value inherits. A function replaces the inherited mapping. `false` removes an inherited mapping. `nil` is not a deletion marker. Instance and child configuration follow the same merge and inheritance rules as the rest of resolved options.

All built-in and custom actions use:

```lua
action(ctx, opts?)
```

The invocation context is an immutable call snapshot:

```lua
ctx = {
  instance = instance,
  instance_id = instance:id(),
  bufnr = bufnr,
  winid = winid,
  mode = "n",
  entry = entry_under_cursor,
  selection = selected_entries_or_empty_list,
  root = instance:root(),
  cursor = { line = line, column = column },
  actions = require("fred").actions,
  config = instance:config(),
}
```

Actions do not guess a global current Fred buffer. `entry` may be nil when the cursor is not bound to an entry; `selection` is always a list. Custom wrappers pass options directly:

```lua
["<C-v>"] = function(ctx)
  return actions.open_new_instance(ctx, { layout = "vsplit" })
end
```

`actions.select()` is a higher-order function that composes entry-kind handlers without string dispatch:

```lua
["<CR>"] = actions.select({
  file = actions.open_file,
  directory = actions.navigate,
  symlink = actions.open_file,
})
```

The default directory handler is `open_new_instance`; users may replace it with `navigate` or any custom function. Missing handlers and invocation without a valid entry are explicit errors.

Commands remain convenience entry points over instances and actions:

```text
:Fred [root]                 create an unnamed instance and open it
:FredApply                   apply the current Fred instance
:FredRefresh                 refresh the current Fred instance
:FredDepth {n|all}           change baseline recursion depth
:FredExpand [depth]          expand the current directory
:FredCollapse                collapse the current directory
:FredPasteInto               paste a FRED-aware yank into the current directory
:FredNew {file|dir}          insert an explicitly typed pending entry
:FredLink {from} {to}        immediately create a symlink outside the planner
```

There is no general temporary filter command. Hidden-file visibility is handled by `view.hidden_files_visible`, `is_hidden_file`, `instance:set_hidden_files_visible()`, and `actions.toggle_hidden_files`. There is no `:FredSort`, sort picker, or built-in sort action; mappings may call `ctx.instance:set_sort({...})` directly.

### 7.8 Expansion, Reveal, And Navigation

A View has a configurable baseline depth plus ordinary local directory expansions. The projection remains a flat list of complete paths relative to the instance's `current_root`.

Example at `/project` with baseline depth `0`:

```text
README.md
src/
tests/
```

Expanding `src/` materializes one level without changing root:

```text
README.md
src/
src/init.lua
src/lib/
tests/
```

Further expansion continues to render complete root-relative paths such as `src/lib/parser.lua`; FRED never introduces indentation or connector glyphs.

`reveal()` accepts only a validated root-relative path:

```lua
instance:reveal("src/lib/parser.lua")
```

Absolute paths must first pass `instance:relative_path(absolute_path)` or `instance:contains(absolute_path)`. A path outside the root is not a supported reveal input and never causes implicit re-rooting or instance creation.

Reveal performs ordinary View operations only:

1. validate the root-relative target path;
2. install precise ignore overrides for the target and its ancestor chain;
3. materialize or request EntryFacts for each required ancestor and the target without first committing the target into the visible projection;
4. classify the ancestor chain and target as those facts arrive;
5. if any required fact is hidden, commit the same persistent `hidden_files_visible = true` transition as the public setter;
6. wait for the target EntryId to appear in the committed visible projection;
7. move the cursor in the selected instance window.

This order lets reveal traverse ignored or previously uncovered ancestors before hidden classification is possible. A real hidden-file visibility change first captures current intent, reprojects while preserving protected dirty rows, clears Neovim undo history, and establishes a new baseline. Existing unapplied intent remains and keeps `'modified'` set. Reveal does not create a pinned row, temporary reveal mode, or `clear_reveal()` API. Its expansions remain normal View state and users collapse them normally. Enabling hidden-file visibility remains global to the instance and persists until the user toggles it again.

`navigate(relative_directory)` is an explicit alternative to creating a child instance. It changes `current_root` while preserving InstanceId, buffer, windows, resolved config, keymaps, attach effects, columns, current hidden-file visibility, and current SortSpec. It invalidates the old View generation, releases the old RootSession reference, attaches the new RootSession, resets local expansion/projection/cursor identity, updates `vim.b[bufnr].fred.root`, re-renders paths relative to the new root, and establishes a new undo baseline. FileType and `on_attach` do not run again.

For example, navigating from `/project` into `src/` transforms:

```text
README.md
src/
src/init.lua
src/lib/
```

into a flat projection rooted at `/project/src`:

```text
init.lua
lib/
```

`navigate` is not another expansion operation. Because one instance owns one buffer, direct same-instance navigation requires a clean View; the default directory action creates a new instance instead, allowing the old dirty buffer to remain available for later apply until its cleanup timer destroys it.

### 7.9 Write Semantics

Full-buffer `:write`, `:update`, and `:wq` invoke the same apply behavior as `instance:apply()` and `:FredApply`.

The Lua `BufWriteCmd` handler starts the confirmation/apply pipeline and does not return to the write command until that attempt reaches a terminal result. It may pump the Neovim event loop while Rust workers run, but the pipeline does not mutate buffer lines while the callback is active. One post-apply View synchronizer owns reprojection, edit-capture suppression, `'modified'`, and the new undo baseline.

Write outcomes are explicit:

- successful apply or a successful empty plan records a pending clean projection, clears `'modified'`, and returns write success;
- cancellation or any pre-execution failure records no projection, leaves `'modified'` set, and returns without write success;
- execution failure records a pending clean projection from the model containing only completed effects, clears `'modified'` because failed/unstarted intent is abandoned, and reports write failure so `:wq` remains open.

After `BufWriteCmd` returns, the synchronizer applies any pending projection with edit capture suppressed, establishes the ordinary undo baseline, and restores the outcome's intended `'modified'` value. A direct `instance:apply()` or `:FredApply` invocation uses the same synchronizer immediately when no command-event restriction is active.

Rules:

- every non-empty plan is previewed and confirmed;
- `:wq` closes only after successful apply or a successful empty plan;
- preview cancellation, validation failure, stale-plan failure, lock contention, or final-preflight failure leaves `'modified'` set;
- successful apply produces a clean synchronized View and a new ordinary undo baseline through the post-apply synchronizer;
- an empty plan records a synchronized model, produces a clean reprojected and sorted View through the same synchronizer, and returns without preview;
- alternate-file writes, ranged writes, and append writes are rejected;
- hiding an instance or transferring its window does not apply, discard, or prompt; the hidden buffer remains editable state until reopened or destroyed;
- destroying an instance discards unapplied buffer text and intent without filesystem mutation;
- FRED does not export its path list through ordinary write syntax.

Hidden-file visibility changes are presentation synchronizations, not writes. A setter that supplies the already-committed boolean is a no-op. Every actual boolean change captures current intent and, after Rust reconciles the atomic ordered-ID submission, clears Neovim undo history and establishes a new baseline even when visible real-row membership/order happens to remain unchanged. Existing intent and `'modified'` survive. An identical normalized SortSpec is also a no-op. A different SortSpec updates presentation state, but clears history only when its committed result actually changes real-row order.

Normal Vim editing remains the primary interface for create, rename, move, copy, replacement, and delete.

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

Extmarks track positions through supported buffer edits but do not travel through yank or put. FRED therefore records provenance separately on every supported Neovim 0.12 pair.

- FRED installs buffer-local `p` and `P` wrappers because Neovim 0.12 does not expose `TextPutPre` or `TextPutPost`;
- the wrappers execute normal put behavior, preserving the chosen register, count, direction, undo behavior, and cursor semantics, then record the inserted range;
- `TextYankPost` stores provenance for the affected register as its register name, text and register type snapshot, `RootSessionId`, and source EntryIds;
- a wrapper attaches provenance only when the register actually used still has the recorded text and type and belongs to the same RootSession; a mismatch invalidates that register's provenance;
- provenance is valid only in the same RootSession; wrappers do not attach foreign-session provenance;
- insertions not observed with valid same-session provenance are ordinary unbound CREATE declarations, even if their text matches an existing row.

FRED does not guess provenance from matching text or from another RootSession.

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

`:FredPasteInto`, mapped to `gp`, copies the most recent FRED-aware yank into the directory under the cursor. The yank must belong to the current View's RootSession; a parent- or child-instance yank from a different RootSession produces an explicit error. Users can expand directories in the same flat View when source and destination must share provenance.

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

Published snapshots remain alive while referenced by dirty Views and are released when no View needs them. Namespace snapshots contain identity, path, kind, coverage authority, and scan status; optional permission, size, and timestamp observations live in a separate RootSession metadata cache with its own monotonically increasing `metadata_revision`. Metadata enrichment or eviction may re-render columns without advancing `root_revision` or any View edit base.

Cleanup may release only cache objects that are not protected by the current RootSession, an attached View's edit base, pending intent or diagnostics, an active scan/apply task, or watcher coverage. This protection rule applies uniformly to scan-cache rows, metadata records, provisional batches, and obsolete snapshots.

### 10.2 RootSession

All Views of the same canonical root share one Rust RootSession:

```text
RootSession {
  canonical_root
  current_snapshot: Arc<Snapshot>
  root_revision
  scan_cache
  metadata_cache
  metadata_revision
  scan_generation
  metadata_generation
  cache_usage: { estimated_bytes, cached_entry_count }
  cleanup_lru
  directory_status
  watcher_state
  coverage_reference_counts
  metadata_reference_counts
  task_state: Idle | Scan | Apply
}
```

The RootSession does not own a mutation lock. The mutation lock is process-global because different canonical roots can overlap.

### 10.3 Instance

Lua exposes one public Instance object backed by authoritative runtime registration:

```text
Instance {
  instance_id
  optional unique name
  optional profile
  state: Created | Open | Hidden | Destroyed
  initial_root
  current_root
  resolved_config
  presentation_state
  bufnr: optional
  view_id: optional
  displayed_windows: Map<winid, LayoutHandle>
  cleanup_timer: optional
}
```

The Instance owns its one buffer/View lifecycle and holds references to whichever RootSession matches `current_root`. Window count is derived from actual windows displaying the buffer, not from keyboard activity. Named and buffer-based registry lookups validate that the live runtime identity still matches rather than trusting reusable names or buffer numbers.

### 10.4 View And Presentation State

Each FRED buffer has one Rust View plus one Lua presentation state:

```text
Rust View {
  root_session_id
  view_id
  bufnr
  view_generation
  baseline_depth
  local_expansions
  explicit_includes
  columns
  column_layout
  projection
  intent
  intent_generation
  dirty_state
  edit_base: Arc<Snapshot>
  cursor_identity
  last_changedtick
}

Lua PresentationState {
  hidden_files_visible
  sort
  presentation_generation
  catalog_by_entry_id
  hidden_classification_cache
  last_full_catalog_order
  frozen_dirty_full_catalog_order: optional
  next_frame_sequence
  proposed_transition: optional
}
```

Rust seals each catalog with an opaque immutable token:

```text
CatalogFrameToken {
  root_session_id
  view_id
  view_generation
  scan_generation
  root_revision
  metadata_generation
  metadata_revision
  presentation_generation
  intent_generation
  changedtick
  frame_sequence
  sealed_identity
}
```

Every frame-bound field must equal the current authoritative value at commit time. `frame_sequence` increases for each sealed frame of a View, `sealed_identity` makes the token unforgeable and single-use, and a token becomes stale when any bound identity/revision/generation/tick differs. `presentation_generation` advances when an actual hidden-file visibility or SortSpec proposal is accepted for scheduling, when a clean/dirty transition changes the ordering regime, or when a newer proposal supersedes in-flight classification/sorting. The proposal's frames use that new generation; failure may leave committed presentation values unchanged while the generation remains advanced for cancellation safety. Same-value/no-op setters do not advance it. Metadata-only change advances metadata generation/revision rather than presentation generation.

Rust streams bounded EntryFacts for every entry in the View-requested scopes into Lua's token-bound catalog. Lua either submits one atomic `{ token, ordered_visible_ids }` value or submits nothing. Any unique subset of IDs belonging to that frame is structurally valid projection absence. Rust validates exact token equality, membership, uniqueness, and single-use atomicity; it cannot determine whether Lua accidentally omitted an otherwise visible ordinary ID. Callback failure, non-boolean callback return, cancellation, or supersession aborts before submission, so the previous projection remains committed.

When a clean View becomes dirty, Lua freezes its last full-catalog order, including IDs currently filtered from the visible projection. Metadata may continue to update cached values and virtual columns, but neither metadata nor a newer comparator result changes that frozen order. Dirty hidden-file membership changes take a subsequence or superset of the frozen order. IDs discovered after the freeze are placed after the frozen IDs in deterministic normalized-path, exact-path, then EntryId order until the View becomes clean. On becoming clean, Lua discards the freeze and recomputes the full-catalog order from current metadata and the current SortSpec.

Rust merges ordinary browse IDs with protected edit, conflict, and validation rows. An ordinary ID already represented by a protected row is removed from the ordinary sequence before merge. Protected rows retain their current relative order and stable predecessor/successor EntryId anchors from the last committed projection. A protected group is placed between surviving anchors when possible, after a surviving predecessor or before a surviving successor when only one remains, and at the end in current protected-row order when neither survives. Reconciliation produces at most one row for each EntryId. Lua never owns edit identity or deletion authority.

A clean View may advance its edit base whenever a new snapshot is published. A dirty View keeps the immutable snapshot against which its current intent was captured.

Two Views over the same root may use different depth, expansion, hidden-file visibility, sort, and column settings and may both contain edits.

### 10.5 Revision And Rebase Behavior

When a new snapshot is published:

- clean Views reproject from it;
- dirty Views compare their immutable edit base, the new snapshot, and their desired namespace;
- non-overlapping namespace changes merge automatically;
- conflicting paths, kinds, identities, or destinations remain visible and block the affected plan;
- the dirty View advances its edit base only after a successful rebase.

File contents, permissions, size, and timestamps are not part of this conflict model.

### 10.6 Instance And View Lifecycle

The runtime owns authoritative registries keyed by opaque `instance_id` and `view_id`, with validated secondary lookups by optional unique name and `bufnr`.

Window entry/exit updates the Instance's display handles and cleanup timer without changing View intent. Hiding or transferring every window leaves the buffer/View alive in Hidden state. Buffer wipeout, explicit destroy, or instance-delay expiration invalidates the Instance and View generations, discards unapplied text/intent, releases coverage and edit-base references, cancels instance-owned timers/tasks, and detaches from RootSession. Async events carry instance, root, View, and generation identity rather than trusting reusable window or buffer numbers.

Same-instance `navigate` requires a clean View, invalidates only the old View generation, keeps InstanceId/buffer/windows/configuration/current presentation state, attaches a new View to the destination RootSession, updates buffer metadata, and resets the undo baseline. Creating a child instance instead leaves the parent View—including dirty text—intact until it is reopened or destroyed.

### 10.7 Overlapping RootSessions

After any successful or partially successful mutation, the runtime invalidates scan generations and schedules refresh for every attached RootSession whose root overlaps an affected path. This keeps parent-root and child-root Views consistent even though they do not share one RootSession.

## 11. Identity And Provenance

Every rendered existing row has a full-line range extmark carrying its opaque RootSession-scoped `EntryId`. The range spans `[row, 0]` through `[row + 1, 0]` with `right_gravity = false`, `end_right_gravity = true`, `invalidate = true`, and `undo_restore = true`. Deleting the complete row during user edit capture invalidates the mark instead of moving a point mark onto an adjacent row. Renderer-owned reprojection, sorting, and post-apply synchronization run under an internal-sync guard that suppresses edit capture and discards/recreates identity marks; their invalidations never create delete intent. Marks and statuses bind to EntryId, not line number.

Identity rules:

- editing text preserves identity while one valid range extmark remains associated with that row;
- moving text preserves identity only when Neovim moves that exact range with it;
- an invalid range mark observed outside the internal-sync guard represents user removal of its bound row;
- one rendered existing row cannot carry multiple valid existing EntryIds, and two rows cannot carry the same identity;
- yank and put never copy an extmark;
- a provenance-linked put receives a new pending identity with `copy_from = { root_session_id, entry_id }`;
- provenance from another RootSession is never resolved or normalized as COPY/MOVE in the current View;
- source removal plus one same-session destination may normalize that destination to MOVE;
- if an extmark is lost for a reason other than range invalidation, FRED may rebind only through a unique, unconsumed, unchanged base path;
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
required metadata fields from configured columns and Lua presentation SortSpecs
max_entries
max_directories
generation
```

The worker emits bounded events such as:

```text
ScanBatch
MetadataBatch
ScanProgress
DirectoryComplete
DirectoryFailed
ScanComplete
ScanCancelled
```

Requirements:

- emit at most `render_batch_size` entries per batch and expose every entry in each requested scope to its attached Lua presentation catalog, regardless of hidden-file classification;
- seal every frame with the exact `CatalogFrameToken` identity defined in Section 10.4;
- accept at most one atomic `{ token, ordered_visible_ids }` submission for that token, and accept no partial pushes or later append;
- require exact equality for every frame-bound token field, reject stale or reused tokens, and reject unknown or duplicate IDs;
- allow any unique subset of frame IDs as projection absence without claiming semantic knowledge of Lua's intended visibility;
- independently deduplicate and retain protected dirty, conflict, and validation rows according to Section 10.4;
- commit buffer lines, extmarks, virtual text, and cursor reconciliation only after the atomic submission validates;
- attach `scan_generation` to namespace events and `metadata_generation` plus `source_root_revision` to metadata events;
- increment `metadata_generation` whenever required metadata fields change, a metadata request is cancelled, or a new namespace snapshot publishes;
- accept a MetadataBatch only when its metadata generation is current, its source root revision still equals `root_revision`, every field remains required, and EntryId plus encoded path still match the current snapshot;
- discard events from obsolete scan or metadata generations;
- attach metadata observations to EntryId plus the observed encoded path so a late value cannot bind to a moved or replaced entry;
- record each requested scope as complete, partial, or failed for that generation;
- never follow a directory symlink;
- stop at configured limits and report partial coverage;
- publish exactly one immutable snapshot only after every requested scope in the generation has reached a terminal result;
- retain partial and failed scopes as non-authoritative status inside the published generation result;
- keep Neovim calls on the main thread;
- fetch stat metadata only when a configured column or Lua's current SortSpec requires it;
- allow metadata-only enrichment to use the bounded Scan worker/channel without publishing a namespace snapshot or advancing `root_revision`;
- publish accepted metadata batches by advancing only `metadata_revision` and re-rendering affected virtual columns;
- treat metadata collection failure as a per-entry placeholder, not as a namespace scan failure.

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

Hidden-file visibility and Lua filtering do not reduce or expand coverage and never trigger a whole-root recursive scan. Rust still sends all entries from the already requested scopes. Explicit reveal may add a precise ancestor chain and ignore override; collapsing that chain releases the corresponding coverage normally.

Coverage is reference-counted. When no attached View needs a directory, its watcher may be released and its cached state may cease to be authoritative. COPY_TREE and DELETE_TREE do not add descendant enumeration coverage.

Metadata requirements are also reference-counted as the union of attached View columns and Lua presentation SortSpecs. Removing a column or changing sort may release its metadata requirement without reducing directory coverage. Optional metadata that no View requires becomes eligible for cleanup, while namespace entry identity, path, kind, and coverage authority remain intact.

### 12.3 Watcher Behavior

FRED automatically attempts to establish watchers for covered directories; version one has no watcher toggle. Established watchers monitor covered directories. Events are debounced and coalesced by affected directory, then trigger incremental scan requests.

FRED-generated mutations also produce watcher events. The executor reports affected paths so watcher events can be merged with explicit post-execution refresh.

If watcher coverage cannot be established, FRED displays a degraded warning and leaves manual refresh available. Missing watcher coverage alone does not prevent apply.

### 12.4 Shared Cache Cleanup Policy

FRED has no persistent disk cache in version one. Shared cleanup governs Rust-owned scan entries, per-entry metadata, provisional scan batches, and obsolete snapshots retained by references. It is distinct from per-instance `cleanup.instance.delay_ms`, and it never deletes a buffer, Instance, View, or filesystem entry.

Reference ownership is authoritative. Current snapshots, live View edit bases, active scan/apply tasks, watcher coverage, diagnostics, pending intent, and precise reveal coverage protect the shared objects they reference. Destroying an instance intentionally drops its View references; other instances continue to protect shared data independently.

`cached_entry_count` counts namespace rows retained by `scan_cache`; attached metadata records do not add another count, and provisional batches plus snapshots are not part of this count. `estimated_bytes` includes scan-cache rows, metadata records, provisional batches, and uniquely owned retained snapshots.

Cleanup has two layers:

1. Reference cleanup is mandatory. Cancelled or obsolete scan generations release provisional batches immediately. Destroying, navigating, or detaching a View releases its coverage and edit-base references. An object with no protecting reference becomes eligible.
2. Policy cleanup is opportunistic. A bounded sweeper follows the object-class priority defined below and removes least-recently-used eligible objects within each class when they exceed configured idle age, memory estimate, or entry count.

The setup-level shared policy is:

```lua
cleanup = {
  shared = {
    interval_ms = 30000,
    max_idle_ms = 300000,
    max_memory_bytes = 268435456,
    max_cached_entries = 0,
  },
}
```

Per-instance construction may override `cleanup.instance`, but does not create competing shared-cache policies for one RootSession. Shared policy comes from setup and applies process-wide.

`interval_ms` controls only periodic sweeps. `max_idle_ms` uses each object's monotonic `last_used_at`. `max_memory_bytes` is a soft estimated Rust-owned cache budget. `max_cached_entries` counts only `scan_cache` namespace rows. Every value is a non-negative integer; `0` disables only that condition.

Reference cleanup always runs. Policy cleanup is event-driven after cache insertion, View detach, scan completion/cancellation, and watcher coverage changes whenever any threshold is nonzero. Nonzero `interval_ms` adds timer ticks; `interval_ms = 0` disables only timer ticks, not event-driven enforcement of other thresholds.

Expiry first releases obsolete provisional batches and unreferenced snapshots, then unrequired metadata by oldest `last_used_at`, then uncovered scan-cache rows and attached metadata by oldest `last_used_at`. Entry-count pressure uses only the scan-cache-row class. Memory pressure continues through the same ordered classes until the target is met or no eligible object remains.

Limits are soft when all remaining objects are referenced. FRED never evicts live shared state merely to satisfy a limit. Sweeps are bounded, errors are warnings, and cleanup never blocks apply or mutates the filesystem.

## 13. Ignore, Hidden Files, Sort, And Depth

Ignore rules use gitignore-like ordered matching with `!` negation, directory patterns, and root anchoring. Ignoring affects browse traversal and projection, never deletion authority and never the contents copied or deleted by a logical tree operation.

FRED has no general temporary filter. Lua owns hidden-file presentation with one boolean and one classifier:

```lua
view = {
  hidden_files_visible = false,
  is_hidden_file = function(path, ctx)
    return vim.startswith(vim.fs.basename(path), ".")
  end,
}
```

`path` is the canonical encoded root-relative path. `ctx` is immutable and contains only the canonical root and normalized entry kind (`file`, `directory`, `symlink`, or `other`). `is_hidden_file` must return an actual Lua boolean. An exception or non-boolean result is a presentation error. The callback receives no metadata, Instance object, or generic filtering surface.

When `hidden_files_visible` is false, Lua removes an ordinary entry when `is_hidden_file` classifies the entry itself or any covered ancestor directory as hidden. A hidden directory therefore hides its whole currently covered descendant subtree. When the boolean is true, these entries are included. This classification changes projection only: Rust scans only View-requested scopes, sends every entry in those scopes to Lua, and never expands coverage merely to evaluate hidden-file visibility.

`instance:set_hidden_files_visible(value)` validates an actual boolean. `instance:toggle_hidden_files()` inverts the committed value. A same-value call is a no-op. For an actual change on an Open or lifecycle-Hidden instance, FRED captures intent, creates a proposed presentation generation, and schedules bounded/cancellable classification. The public method returns after immediate validation, intent capture, and scheduling; it does not return a promise or async result object. The committed getter value changes only after an atomic ordered-ID submission succeeds. Rust preserves protected dirty/conflict/error rows during reconciliation, then the synchronizer clears undo history and establishes a new baseline while retaining unapplied intent and `'modified'`, even if the visible order happened not to change. `reveal()` persistently enables visibility when needed and waits for that transition before cursor movement.

Sort uses one complete, fixed-shape specification:

```lua
sort = {
  by = "path", -- path|type|size|mtime|ctime|atime|birthtime
  direction = "asc", -- asc|desc
  case_sensitive = false,
}
```

All three fields are required; unknown or missing fields and values are invalid. SortSpec is never deep-merged: every explicit table at setup, direct construction, parent-current snapshot selection, child override, or runtime setter independently contains exactly these three fields and replaces the prior SortSpec wholesale. Lua sorts the catalog after hidden-file filtering. Missing metadata always follows available metadata regardless of direction. Ascending `type` order is directory, file, symlink, other. `direction` reverses only the primary `by` comparison. Normalized path, exact path, and EntryId tie-breakers remain ascending in both directions.

`instance:set_sort(spec)` first normalizes the complete spec. An identical spec is an idempotent no-op even while dirty. A different spec on a dirty View throws a Lua exception and leaves state unchanged; there is no deferred request. On a clean Open or lifecycle-Hidden instance it creates a proposed presentation generation, requests only missing metadata required by its key, and schedules bounded/cancellable ordering. The method returns after immediate validation and scheduling, without an async result object. The getter changes only after the atomic order submission succeeds. Missing values may initially sort last; later metadata may reorder only a clean View. When a View becomes clean after being dirty, Lua reapplies the already-current sort. A changed SortSpec whose committed result leaves real-row order unchanged does not clear history; an actual real-row reorder does.

Baseline depth semantics:

- `0`: entries directly under root;
- `1`: direct entries plus children of direct directories;
- `all`: no depth limit other than configured scan limits.

Local expansion depth is relative to the selected directory. `actions.expand` expands one level; `actions.expand_recursive` requests recursive materialization subject to scan limits.

Changing ignore rules requests a new browse scan. Changing hidden-file visibility only reprojects existing requested coverage. Changing sort changes only Lua presentation state and metadata requirements. Changing baseline depth, local expansion, or precise reveal coverage updates coverage. None of these changes generates filesystem intent.

Default ordering is ascending, case-insensitive stable lexical ordering by normalized relative path. Destination collision checks use actual platform/filesystem behavior, while displayed spelling remains unchanged.

## 14. Planner

The planner is a pure Rust module. It receives immutable inputs and returns either a valid logical plan or non-confirmable diagnostics. It never performs filesystem I/O or mutates the filesystem.

Inputs:

```text
View edit-base snapshot and edit_base_revision
current published snapshot and planned_root_revision
captured View intent
current changedtick and intent generation
immutable direct namespace-probe results for required sources, destinations, existing parents, and planned-parent chains
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
- required parent that neither exists nor is created by the same plan;
- a planned-parent path occupied by an identity that the same plan does not first remove or move away;
- non-directory existing parent or nearest existing ancestor of a planned-parent chain;
- parent traversal through a symlink;
- directory COPY_TREE or MOVE to itself or its descendant;
- dependency cycles that cannot be resolved with a unique same-directory rename name.

The planner does not reject ordinary file-content, permission, size, or timestamp changes.

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

1. an existing planned-parent occupant is removed or moved away before the replacement parent directory is created;
2. every planned parent-directory creation precedes CREATE, COPY, COPY_TREE, MOVE, or REPLACE installing a descendant destination;
3. all copies from a source precede deletion of that source;
4. descendant evacuation precedes parent directory MOVE or DELETE_TREE when the descendant ends outside the parent's final subtree;
5. parent MOVE precedes descendant edits that remain inside the parent's final subtree;
6. unique temporary rename steps precede completion of a swap, cycle, or case-only rename;
7. destination copy success precedes source deletion for cross-filesystem MOVE;
8. explicit target removal precedes REPLACE installation when required by the desired state;
9. irreversible deletes run as late as dependencies permit.

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

Every valid non-empty plan uses this single confirmation. An empty valid plan records a clean synchronized projection and returns without opening preview; the post-apply synchronizer applies the reprojection, sorting, `'modified'`, and undo-baseline effects at the invocation-appropriate time.

## 17. Executor And Failure Semantics

The executor consumes a validated plan and is the only module allowed to mutate planned FRED buffer operations.

After the user confirms a valid plan:

1. acquire the process-global mutation lock;
2. cancel the initiating RootSession's ordinary scan and enter Apply state;
3. compare the plan's changedtick and intent generation with the initiating View;
4. compare `edit_base_revision` with the initiating View's current edit-base revision;
5. compare `planned_root_revision` with the RootSession's current published revision;
6. repeat every required source, destination, existing-parent, planned-parent-state, and nearest-existing-ancestor probe under the lock;
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
- records a clean initiating-View projection from that model for the post-apply synchronizer;
- invalidates every RootSession overlapping an affected path;
- queues best-effort filesystem refresh;
- reports success;
- exits through the common post-lock finalization path.

The post-apply synchronizer re-renders the initiating View with edit capture suppressed, clears `'modified'`, and establishes the new ordinary undo baseline. It runs after `BufWriteCmd` returns or immediately for a direct apply invocation.

A refresh error is reported through Neovim and may be corrected by a later watcher event or `:FredRefresh`. It does not create a special degraded, unknown, reconciliation-required, or apply-blocking state.

### 17.3 Execution Failure

On the first execution error, FRED:

1. stops the batch;
2. reports the failed system call through Neovim;
3. records completed, failed, and unstarted nodes;
4. applies only completed-node effects deterministically to the in-memory namespace model and publishes the resulting immutable model revision;
5. abandons the initiating View's failed and unstarted intent from that apply attempt;
6. records a clean initiating-View projection from the updated model for the post-apply synchronizer;
7. invalidates overlapping RootSessions and queues best-effort filesystem refresh;
8. shows the execution report;
9. exits through the common post-lock finalization path.

A failed node is not assumed successful; a later refresh may reveal partial side effects left by that failed system call. The post-apply synchronizer re-renders the updated model with edit capture suppressed, clears `'modified'`, and establishes a new ordinary undo baseline. A `BufWriteCmd` invocation still reports write failure so `:wq` remains open. The user edits the synchronized model and starts a new apply if desired. FRED does not offer an automatic retry action.

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
- ordinary file content, permissions, size, or timestamps changed: not a FRED conflict;
- external rename: represent conservatively as old-path deletion plus new-path creation.

Watcher events trigger incremental refresh automatically. `:FredRefresh` forces reconciliation of the View's required browse coverage.

A dirty View is never silently overwritten by ordinary refresh. Conflicted entries remain visible regardless of hidden-file settings or collapse.

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
  nvim_adapter.rs
  config.rs
  runtime.rs
  apply_gate.rs
  root_session.rs
  snapshot.rs
  view.rs
  task.rs
  scanner.rs
  metadata.rs
  watcher.rs
  coverage.rs
  columns.rs
  cleanup.rs
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
  fred/config.lua
  fred/instance.lua
  fred/registry.lua
  fred/actions.lua
  fred/keymaps.lua
  fred/lifecycle.lua
  fred/buffer.lua
  fred/presentation.lua
  fred/layout.lua
plugin/
  fred.lua
ftplugin/
  fred.lua
```

The modules expose small internal interfaces with substantial behavior behind them:

- Rust `nvim_adapter` is the only Rust module that calls `nvim_oxi::api` directly and owns the exact-patch-tested buffer, extmark, virtual-text, changedtick, and main-thread notification bindings;
- Rust `path` owns canonical buffer/native path conversion and validation;
- Rust `snapshot` owns immutable revisioned namespace state;
- Rust `root_session` owns shared scan, watcher, coverage, and cache-accounting state;
- Rust `metadata` owns platform metadata collection and normalized optional values;
- Rust `columns` owns built-in column definitions, formatting, alignment, and metadata requirements;
- Rust `cleanup` owns shared candidate selection, object-class ordering, LRU state, soft limits, and bounded sweeps;
- Rust `view` owns per-buffer coverage, keyed projection reconciliation, column layout, edit base, intent, conflicts, diagnostics, and protected-row state;
- Rust `planner`, `tree_ops`, `executor`, and `apply_gate` own mutation planning and execution boundaries;
- Rust `runtime` owns RootSession/View registration, generation identity, overlap invalidation, and native handles exposed to Lua;
- Lua `config` captures setup defaults, validates Lua-only callback/action/layout shapes, and resolves direct/child instance inheritance;
- Lua `instance` owns the public object, root/config identity, and native handle calls;
- Lua `registry` owns InstanceId/name/buffer lookup and terminal removal;
- Lua `actions` and `keymaps` own function-valued actions, context construction, mode/lhs merge, and buffer-local installation;
- Lua `presentation` owns runtime hidden-file visibility and SortSpec state, generation-tagged catalog mirroring, hidden classification/cache and subtree propagation, stable comparison, and ordered EntryId submission;
- Lua `lifecycle` and `buffer` own Neovim autocmd-triggered reconciliation, `vim.b.fred`, FileType ordering, attach callbacks, instance timers, and undo-baseline resets requested by presentation commits;
- Lua `layout` constructs native buffer, float, split, vsplit, and tab displays and returns ordinary window ownership records.

Rust module interfaces remain internal. `src/lib.rs` exposes the native module through `nvim-oxi`; `lua/fred/init.lua` exposes the stable Instance facade. No speculative backend or public custom-column seam is exposed in version one.


## 21. Public Configuration

Initial setup shape:

```lua
local fred = require("fred")
local actions = fred.actions

fred.setup({
  depth = 0,
  ignore = {
    ".git/",
  },
  view = {
    hidden_files_visible = false,
    is_hidden_file = function(path, ctx)
      return vim.startswith(vim.fs.basename(path), ".")
    end,
  },
  columns = {
    "icon",
    -- "permissions",
    -- "size",
    -- { "mtime", format = "%Y-%m-%d %H:%M", width = 16 },
  },
  sort = {
    by = "path",
    direction = "asc",
    case_sensitive = false,
  },
  layout = "buffer",
  cleanup = {
    instance = {
      delay_ms = 300000,
    },
    shared = {
      interval_ms = 30000,
      max_idle_ms = 300000,
      max_memory_bytes = 268435456,
      max_cached_entries = 0,
    },
  },
  limits = {
    max_entries = 50000,
    max_directories = 10000,
    render_batch_size = 500,
  },
  keymaps = {
    n = {
      ["<CR>"] = actions.select({
        file = actions.open_file,
        directory = actions.open_new_instance,
        symlink = actions.open_file,
      }),
      ["-"] = actions.open_parent_instance,
      ["zo"] = actions.expand,
      ["zc"] = actions.collapse,
      ["zO"] = actions.expand_recursive,
      ["zC"] = actions.collapse_recursive,
      ["R"] = actions.refresh,
      ["<leader>s"] = actions.apply,
      ["q"] = actions.hide,
      ["g."] = actions.toggle_hidden_files,
    },
    i = {},
    v = {
      ["gp"] = actions.paste_into,
    },
  },
  on_attach = {},
})
```

Direct `fred.new(opts)` deep-merges setup defaults with explicit instance options, except function/list fields such as `on_attach`, which are normalized and concatenated deliberately, and SortSpec. Every explicitly supplied SortSpec at setup, direct `new()`, or child `new()` is validated independently as exactly `{ by, direction, case_sensitive }` and replaces the previous SortSpec wholesale. `instance:new(opts)` starts from the parent's resolved immutable options, replaces its initial hidden-file visibility and SortSpec with snapshots of the parent's current runtime values, then applies explicit child overrides; an explicit child SortSpec replaces that snapshot rather than merging with it. It does not replay setup callbacks. The resulting child values are part of the child's immutable `config()` snapshot, while later setters affect only its runtime getters.

`cleanup.shared` is setup-only. Supplying it to direct or child `new()` options is invalid; instances override only `cleanup.instance`.

`root` belongs to `new()` options and is not an `open()` option. Direct `fred.new()` defaults it to the current working directory; `parent:new()` defaults it to the parent's current root. `layout` belongs to resolved instance options and may be overridden per `open()`/`toggle()` call. Values are `buffer`, `float`, `split`, `vsplit`, or `tab` in version one. `open({ new_display = true })` is a per-call display option, defaults to `false`, and is never stored in resolved configuration.

`keymaps` is keyed by Neovim mode and lhs. Values are functions or `false`; strings are invalid. Functions replace inherited mappings and `false` removes them. `actions.select()` receives entry-kind action functions and returns the final mapping callback.

`on_attach` accepts one function or a function list. Setup callbacks run before direct instance callbacks; inherited child lists run before explicit child callbacks. The first callback error rolls back and destroys the instance and is rethrown.

`columns` defaults to `{"icon"}`. The path remains the only editable field. `view.hidden_files_visible` and `is_hidden_file` are the complete hidden-file configuration; there is no generic filter or second always-hidden predicate. Reveal may persistently enable hidden-file visibility and add exact ignore overrides without creating pinned rows.

`sort` is the complete three-field SortSpec described in Section 13. All fields are required, unknown fields are errors, and only the built-in single keys are supported. Arbitrary comparators, multi-key sorting, directory-first options, and configurable missing-value placement are not public version-one features.

Watcher establishment, plan confirmation, and no-follow directory-symlink behavior remain automatic and non-configurable in version one.

`cleanup.instance.delay_ms` is a non-negative integer; `0` disables instance auto-destruction. `cleanup.shared` values are non-negative integers; each `0` disables only its shared condition. `interval_ms = 0` disables timer ticks while other nonzero shared thresholds remain event-driven.

Setup validates instance names/options, layout values, keymap mode/lhs/function shapes, action selectors, attach callback lists, `hidden_files_visible`, the `is_hidden_file` callback, complete SortSpecs, columns, time formats, limits, and cleanup values. Invalid configuration produces explicit errors rather than silent fallback.

## 22. Error Handling

Errors fall into six classes.

### 22.1 Scan Or Watch Error

The buffer remains readable. Failed or limited browse scopes are marked partial or degraded. Manual refresh and scope reduction remain available.

Scan or watcher failure does not globally disable apply. A plan is blocked only when its own required source, destination, or parent state cannot be checked.

Cache cleanup is best effort. An individual metadata read or cache release failure is reported without invalidating the namespace snapshot, disabling apply, or deleting filesystem data.

### 22.2 Validation Or Final-Preflight Error

No filesystem action runs. The buffer remains dirty, edits and undo history are preserved, and errors are attached to affected rows and listed in a non-confirmable diagnostic report or location list.

A stale changedtick, intent generation, `edit_base_revision`, or `planned_root_revision` also returns here and requires replanning. If this failure occurs after lock acquisition, the common finalization path leaves Apply, processes queued refresh demand, and releases the global mutation lock.

### 22.3 Conflict

No filesystem action runs. Conflicted entries remain visible in the non-confirmable diagnostic report and require refresh, rebase, or explicit user edits.

Typical conflicts are missing sources, changed source kind, occupied destinations, invalid parents, and incompatible namespace changes. Conflicts never open the confirmation preview.

### 22.4 Execution Failure

The executor stops immediately, reports the system error, and runs no later operation. Completed-node effects update the in-memory namespace model deterministically; failed and unstarted initiating-View intent is abandoned; the initiating View is re-rendered from that model; filesystem refresh remains best effort.

A refresh error is reported through Neovim without blocking a later apply or creating a special state. FRED does not claim that the desired batch completed when an execution node failed.

### 22.5 Instance, Layout, Or Attach Error

Invalid names, roots, layouts, action contexts, keymaps, or lifecycle transitions fail explicitly. An `on_attach` error is fail-fast: FRED stops later callbacks, tears down the partially created windows/buffer/View/timers/RootSession references, marks the instance Destroyed, and rethrows the original error.

A layout-open failure rolls back only resources created by that open attempt. If the instance had no prior buffer/View and buffer initialization cannot complete, it is destroyed; an already valid instance remains usable when an additional window fails to open.

### 22.6 Presentation Error

Immediate public-method validation errors and a different SortSpec on a dirty View throw directly to the caller and leave committed presentation state unchanged; identical normalized values are no-ops. During scheduled catalog processing, an `is_hidden_file` exception or non-boolean return propagates as the direct scheduled Neovim Lua error. FRED does not wrap it with `pcall`, return a structured error, or add another notification. The proposed transition is aborted before any `{ token, ordered_visible_ids }` submission, so the previous committed state and projection remain intact. Stale or superseded presentation generations cancel without submission. Public setters return after immediate validation and scheduling; they do not invent promise-like result objects.

## 23. Testing Strategy

### 23.1 Rust Unit Tests

- canonical path encoding and decoding;
- decode-then-reencode equality;
- malformed escapes, `%41` aliases, encoded separators, literal and encoded `.`/`..`, absolute paths, and root escape rejection;
- Linux byte-path and Windows native-name conversion used by supported builds;
- EntryId allocation, full-line range-extmark invalidation, exact start/end gravity, undo restoration, internal-sync suppression, duplicate-row rejection, and lost-mark rebinding rules;
- register-content-qualified same-RootSession provenance, register overwrite invalidation, and normalization for retained source, one removed-source destination, and multiple removed-source destinations;
- arbitrary projection omissions caused by ignore, hidden-file membership, ordering, depth, collapse, cancellation, failure, limits, or watcher gaps never generating deletion;
- immutable direct namespace-probe inputs for missing sources, wrong kinds, occupied destinations, existing parents, planned-parent absence or planned occupant removal, nearest existing ancestors, and symlink parent traversal;
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
- RootSession coverage reference counting;
- sealed CatalogFrameToken equality, single-use atomic submission, and stale/reused/unknown/duplicate-ID rejection;
- arbitrary unique frame-ID subsets remaining valid projection absence while protected dirty intent, conflict, and validation rows survive deduplicated reconciliation;
- protected-row predecessor/successor anchoring, deterministic fallback, relative-order preservation, and no duplicate EntryId;
- column registry validation, virtual-text rendering, fixed-width alignment, metadata placeholders, and metadata-only refresh;
- column and Lua SortSpec metadata requirements fetching only the required stat fields;
- shared-cleanup candidate protection for current snapshots, live dirty edit bases, active tasks, watcher coverage, diagnostics, pending intent, and precise reveal coverage;
- cleanup age, memory, and entry-count thresholds, including independent `0` disables and `interval_ms = 0` with nonzero event-driven thresholds;
- `max_cached_entries` counting only scan-cache namespace rows while metadata, provisional batches, and snapshots contribute only to the memory estimate;
- object-class eviction priority with least-recently-used ordering inside each class;
- metadata-generation cancellation and MetadataBatch rejection for stale-generation/current-source-revision and current-generation/stale-source-revision cases after column, Lua SortSpec, path, and namespace-revision changes;
- renderer reconciliation consuming already-ordered IDs without owning Lua comparator or classification behavior;
- same-instance navigation rebasing complete paths to the new current root and rejecting dirty View navigation;
- destroying a View releasing shared references without deleting still-referenced RootSession data.

### 23.2 Lua Unit Tests

- normalization and validation of the complete three-field single-key SortSpec, including exact-field checks, no-deep-merge replacement at every configuration boundary, and idempotent equality;
- runtime getters returning read-only committed-state copies without mutating the immutable construction snapshot;
- Created-state setters updating only Lua state without View work, and opened setters committing state only after successful submission;
- `is_hidden_file` receiving only canonical encoded root-relative path plus immutable root/kind facts and rejecting non-boolean returns;
- hidden-file classification caching and invalidation when path, root, kind, callback identity, or presentation generation changes;
- hidden-directory classification propagating to every currently covered descendant while visibility is disabled;
- filter-then-sort behavior for every built-in key, both directions, case sensitivity, missing-values-last, type ordering, primary-only direction reversal, and ascending tie-breakers;
- bounded/cancellable catalog classification and sorting yielding event-loop progress on large catalogs;
- exact stale `presentation_generation` cancellation before atomic submission;
- frozen dirty full-catalog order, membership-only visibility changes, deterministic newly discovered-ID fallback, and clean recomputation;
- visibility state and SortSpec proposals rolling back when validation, callback execution, or boolean-return validation fails;
- same-value hidden visibility and identical normalized SortSpec calls remaining no-ops;
- child construction snapshotting parent current presentation values, applying explicit overrides, and remaining independent of later parent changes;
- direct instance root defaulting to cwd and child root defaulting to the parent's current root.

### 23.3 Integration Tests

- recursive and selectively materialized local scanning;
- one immutable snapshot publication only after every requested scope in a scan generation reaches a terminal result;
- partial and failed scopes remaining non-authoritative inside the published generation result;
- cancellation and late-generation rejection;
- bounded Rust catalog batches containing all entries in requested scopes without hidden-file-driven coverage expansion;
- end-to-end hidden-directory subtree projection and fixed SortSpec ordering across Rust catalog delivery, Lua presentation, and Rust reconciliation;
- sealed atomic ordered-ID submission rejecting stale tokens and duplicate or unknown IDs without partial rendering, while accepting any unique frame-ID subset;
- partial scan limits while unrelated known intents remain applicable;
- watcher establishment failure while direct apply checks remain usable;
- ignore, Lua hidden-file filtering, Lua sorting, depth, expansion, collapse, scan cancellation, scan failure, scan limits, and watcher gaps never generating deletion;
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
- missing destination parents that are not planned, absent and replacement-created parent chains ordered before child installation, non-directory parents or nearest existing ancestors, and parent traversal through a symlink;
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
- process restart performing an ordinary scan of partial results;
- column changes while Views are clean and dirty without changing editable lines or intent;
- metadata refresh after size, permission, or timestamp changes;
- shared cleanup after instance destruction, View navigation, scan cancellation, and watcher coverage release;
- memory and count pressure following object-class priority and least-recently-used ordering within each class;
- entry-count pressure ignoring metadata records, provisional batches, and snapshots;
- exact virtual-column fixture without `/NNN` or `../`, with raw buffer lines containing only paths;
- stale metadata events after cancellation, column changes, SortSpec changes, entry moves, and snapshot publication, including independent stale-generation/current-source-revision and current-generation/stale-source-revision fixtures;
- stale presentation-generation cancellation across visibility, SortSpec, clean/dirty regime, and superseding-transition changes;
- `interval_ms = 0` preserving event-driven memory/count enforcement;
- dirty hidden-file reprojection using frozen full-catalog order, deterministic new-ID fallback, protected-row deduplication/anchors, and omission remaining non-deletion;
- exact reveal through ignored or uncovered and hidden ancestor chains, fact acquisition before classification, persistent hidden-file visibility, history reset, normal expansion state, and no reveal pin;
- child creation snapshotting the parent's current hidden-file visibility and SortSpec into the child's immutable initial configuration;
- same-instance navigation producing a newly rebased flat projection while child-instance navigation preserves the parent View.

### 23.4 Headless Neovim Tests

- build and load with every exact allowlisted Neovim 0.12 patch-version and pinned `nvim-oxi` revision pair;
- Linux and Windows build/load coverage plus a direct-API compatibility spike for every `nvim_adapter` call in each listed pair;
- `require("fred").build()` availability;
- setup, direct-instance, and child-instance configuration validation, including exact complete SortSpecs with wholesale replacement, hidden-file callbacks, runtime getters, Created-state setters, and current-presentation child inheritance;
- real-buffer hidden-directory subtree propagation plus primary-direction and ascending-tie-break SortSpec behavior;
- command registration plus function-valued mode-grouped keymaps with `false` removal and no action strings;
- focused `:FredNew file` and `:FredNew dir` commands;
- one Instance owning one buffer/View while multiple instances and windows share one RootSession safely;
- immutable edit-base snapshots in dirty Views;
- process-global mutation lock across same, unrelated, and nested roots;
- successful and failed `:FredLink` paths both releasing the global lock, invalidating and refreshing overlapping RootSessions when namespace may have changed, and allowing a later apply or `:FredLink`;
- different depth and expansion settings per View;
- union watcher coverage across Views;
- unconditional watcher establishment attempts, degraded establishment failure, and manual refresh;
- watcher-driven clean refresh and dirty namespace rebase;
- overlapping RootSession invalidation after mutation;
- `buftype=acwrite` full `:write`, `:update`, and `:wq` behavior, including a `BufWriteCmd` outcome table, deferred capture-suppressed projection, and execution failure that leaves `:wq` open after synchronizing a clean model;
- rejection of alternate-file, ranged, and append writes;
- valid non-empty plans requiring one confirmation;
- invalid/conflicted planning results opening non-confirmable diagnostics and preserving edits;
- empty plan reprojection and modified-state clearing;
- preview cancellation preserving edits;
- final-preflight failure leaving Apply and releasing the global lock while preserving dirty state;
- undo and redo before apply;
- undo baseline reset after successful apply and execution-failure model re-render;
- dirty `set_hidden_files_visible()`/`toggle_hidden_files()` capture, frozen-order membership change, protected-row reconciliation, history reset after every actual boolean change, preserved intent, and same-value no-op;
- clean sort state change with history reset only on real-row reorder, dirty changed-SortSpec exception, dirty identical-SortSpec no-op, and absence of any deferred requested sort;
- cursor and identity preservation after refresh;
- ignore, Lua hidden-file filtering, Lua sorting, depth, collapse, cancellation, scan failure, limits, and watcher gaps never implying deletion;
- Neovim 0.12 buffer-local `p`/`P` wrappers preserving native put behavior while validating register name, text, type, and same-RootSession provenance;
- register overwrite invalidation, cross-RootSession `gp` rejection, and foreign or unobserved matching insertions remaining CREATE;
- `gp` into a directory and deterministic `_N` destinations;
- collapse preserving pending-intent, conflict, and validation rows while ordinary reveal expansions collapse normally;
- partial scan warnings without global apply disablement;
- large-batch rendering responsiveness;
- bounded/cancellable Lua hidden classification and sorting making observable event-loop progress, and stale presentation generations never committing after supersession;
- worker notification through entry-count- or time-bounded `AsyncHandle` queue batches that re-wake while events remain;
- virtual columns not changing buffer text, changedtick, undo history, or path parsing;
- configuring every built-in column and rejecting invalid column, hidden-file, SortSpec, and cleanup setup values;
- clean metadata sort reordering with history reset, dirty metadata virtual-column refresh without real-line movement, and current-sort reapplication on becoming clean;
- aligned placeholders and display-cell widths for narrow and wide icon glyphs;
- `fred.new()` setup merge, `instance:new()` current-presentation snapshot plus parent-option inheritance, immutable resolved config, runtime presentation getters, and no duplicate setup attach callbacks;
- optional live-name uniqueness and InstanceId/name/buffer registry lookup validation;
- direct `fred.new()` root defaulting to cwd, `parent:new()` root defaulting to the parent's current root, per-open layout selection when creating a display, ignored layout overrides when default `open()` focuses, and root absence from `open()` options;
- one instance buffer displayed in multiple windows and multiple tabpages;
- native Lua layout construction without NUI or layout types leaking into Instance/View/RootSession core types;
- borrowed-buffer restore, owned float/split/tab hide, and final ordinary-window preservation;
- `open()` focusing an existing current-tab display by default, `new_display = true` creating an additional display, `hide()` targeting current/explicit windows, and no public close/hide-all;
- `toggle()` hiding every current-tab display including floats while leaving other tabs untouched, or opening when none exist;
- default directory action creating a child instance and transferring the triggering window without custom history;
- Created/Open/Hidden transitions derived by reconciliation triggered by `WinNew`, `BufWinEnter`, `WinEnter`, deferred `WinClosed`, and `BufHidden`, with validated `BufDelete`/`BufWipeout` entering terminal Destroyed directly;
- instance `delay_ms` cancellation on redisplay, `0` disablement, and terminal destruction after zero-window delay;
- hidden dirty buffers remaining available for later apply before expiry, then discarding text/intent without filesystem mutation on destruction;
- fixed `filetype=fred`, complete `vim.b[bufnr].fred` before FileType autocmds, and live instance lookup by buffer;
- keymap installation before FileType, then setup attach callbacks, then instance callbacks;
- `on_attach` function/list normalization, one execution per buffer, and fail-fast rollback/destruction;
- action `ctx` construction, optional opts, explicit entry/selection behavior, `actions.select()` function dispatch, and the renamed `actions.toggle_hidden_files` mapping;
- the public Instance/action surface exposing only the approved hidden-file methods and action, with no compatibility aliases for superseded names;
- absence of a sort command, picker, or built-in sort action and direct custom mappings through `ctx.instance:set_sort()`;
- `is_hidden_file` exceptions and non-boolean returns propagating directly through Neovim while the previous committed state/projection remains intact;
- reveal accepting only root-relative paths, establishing ignore overrides and ancestor EntryFacts before classification, persistently enabling hidden-file visibility with history reset, expanding ancestors, and moving the cursor;
- same-instance clean navigation preserving buffer/window/config while resetting View/RootSession/projection/undo baseline;
- layout-open failure rollback for a new or already valid instance.

### 23.5 Property Tests

Randomly generated snapshots and intents must preserve:

- no duplicate final destinations;
- no buffer-planned operation outside the View root;
- no missing DAG dependencies;
- topological sortability after rename-cycle resolution;
- no directory COPY_TREE or MOVE into itself;
- all register-qualified same-RootSession provenance copies before removed-source deletion;
- register overwrite or cross-RootSession provenance never generating COPY or MOVE;
- user line deletion invalidating exactly one full-line EntryId range while internal synchronization generates no filesystem intent;
- every accepted ordered-ID submission using a current single-use token and a unique subset of frame IDs, with omission never authorizing deletion and protected rows never disappearing or duplicating;
- hidden-directory classification hiding every covered descendant when hidden files are not visible;
- no operation after the first injected execution error;
- ignore, Lua hidden-file filtering, Lua sorting, depth, expansion, collapse, scan cancellation, scan failure, limits, watcher gaps, reveal, layout, and instance lifecycle do not change filesystem intent;
- no silent overwrite under injected destination races;
- a required parent must exist or be created earlier by the same plan, including replacement-created parents whose occupant first vacates, while unrelated occupants, non-directory parents or nearest existing ancestors, and symlink parent traversal reject the affected plan;
- a scan generation publishes at most one immutable current snapshot and only after all requested scopes are terminal;
- every post-lock outcome leaves Apply and releases the global mutation lock;
- completed execution-node results deterministically produce the same in-memory namespace model;
- shared coverage equals the union of attached View requirements;
- dirty View rebase always retains its immutable edit base until rebase completes;
- metadata columns, sorting, and cleanup never change filesystem intent;
- cleanup never evicts a snapshot or cache entry still reachable from a protected View/session state;
- setting any cleanup threshold to `0` disables that threshold without disabling mandatory reference cleanup.
- destroying one instance cannot release shared objects referenced by another instance;
- a live Instance maps to at most one buffer/View even when arbitrarily many windows display it;
- instance cleanup cannot run while any window displays its buffer;
- an unapplied View can disappear only through explicit or timed instance destruction, and that disappearance produces no filesystem operation.

## 24. Performance Requirements

- Scanning and rendering must not block Neovim beyond one bounded main-thread batch.
- Root, depth, expansion, ignore, or explicit refresh changes cancel obsolete scan generations.
- Late events from cancelled generations are ignored.
- Large COPY_TREE operations run on a worker thread through the deep tree operation module.
- Provisional scan results may be displayed without becoming an authoritative dirty-View merge base.
- Lua consumes bounded catalog batches and performs cancellable hidden classification and comparison in bounded main-thread slices that yield observable event-loop progress; stable global ordering is committed after the relevant scan generation reaches terminal state, while interim order may be provisional.
- Atomic ordered-ID submissions are one-shot and sealed-token-bound; a stale presentation generation cancels work before submission and cannot mutate the buffer.
- Re-rendering preserves cursor by EntryId when possible.
- Multiple Views share snapshot and watcher data rather than rescanning one canonical root independently.
- Memory use scales linearly with covered entries, Lua's per-View catalog mirrors/classification caches, retained edit-base snapshots, and visited directories.
- Snapshot versions are released when no dirty View references them.
- Stat metadata is fetched only for columns and Lua SortSpecs that require it; metadata-only refresh must not trigger unnecessary namespace planning.
- Metadata sorting may reorder only clean Views; dirty Views freeze real-line order while virtual columns continue to refresh, then reapply the already-current sort when clean.
- Cleanup is bounded and runs outside the Neovim rendering critical path; memory/count pressure follows the documented object-class priority and uses least-recently-used ordering within each class.
- Instance window bookkeeping is proportional to actual displays and does not duplicate scan/snapshot data.
- Instance cleanup uses one timer only while undisplayed; redisplay cancels it without polling keyboard activity.
- native Lua layout work remains outside scan event draining and the Rust filesystem critical path.
- Version one uses no Tokio runtime and no speculative thread pool.

## 25. Implementation Sequence

All Section 26 acceptance criteria define the version-one contract. Development proceeds through runnable internal milestones so validation does not wait for the complete mutation stack:

```text
M0  exact-pair build/load, native loader, Instance lifecycle, and empty View
M1  read-only scanning, flat projection, Lua hidden-file presentation/sort, and metadata
M2  shared RootSession data, watchers, dirty-View edit bases, and namespace rebase
M3  range EntryIds, edit capture, same-session provenance, pure planner, and preview
M4  synchronous write integration, global mutation gate, executor, and failure reporting
M5  cross-filesystem and rename-cycle hardening plus the complete Linux/Windows matrix
1.0 every Section 26 acceptance criterion passes
```

The implementation sequence within those milestones is:

1. Establish the Cargo workspace, initial Neovim 0.12.4/`nvim-oxi` candidate, `nvim_adapter` compatibility spike, Linux/Windows build paths, local build helper, native loader, and headless Neovim harness.
2. Implement Lua configuration capture, direct/child Instance inheritance including current-presentation snapshots, immutable resolved options, runtime presentation getters/setters, registry/name lookup, function-valued actions/keymaps, `vim.b.fred`, FileType ordering, attach callbacks, and native layout ownership records.
3. Implement InstanceId/ViewId native registration, one-buffer-per-instance lifecycle, authoritative window reconciliation, open/hide/toggle/window transfer, instance timers, and terminal destroy/discard semantics.
4. Implement canonical path algebra, reversible filename encoding, immutable Snapshot, RootSession/View state, runtime registry, and read-only flat rendering.
5. Implement the Rust worker channel, bounded provisional and main-thread event batches, cancellation, generation checks, `AsyncHandle` notification with re-wake, and one atomic snapshot publication after every requested scope in a generation is terminal.
6. Add baseline depth, local expansion/collapse, reference-counted coverage, ignore rules, Rust EntryFact catalog frames and sealed renderer commits, Lua `presentation` hidden-file filtering/subtree propagation/sorting, metadata columns, conditional stat fetching, presentation history resets, reveal/navigation, watcher establishment/degraded reporting, and shared cleanup.
7. Add immutable dirty-View edit bases, namespace-only three-way rebase, overlapping RootSession invalidation, and conflict diagnostics.
8. Add full-line range EntryId extmarks, Neovim 0.12 buffer-local `p`/`P` wrappers, `:FredNew`, edit capture, same-RootSession provenance normalization and `gp`, and directory-row descendant hiding.
9. Implement immutable direct namespace probes including planned-parent chains, the pure planner with both revision tokens, COPY_TREE/DELETE_TREE, nested directory normalization, descendant evacuation, self-subtree rejection, replacement validation, collision checks, and cross-filesystem MOVE expansion.
10. Build synchronous `acwrite`/`BufWriteCmd` integration, write-form rejection, non-confirmable diagnostics, confirmation for valid plans, stale-plan checks, and repeated direct probes in final preflight.
11. Implement the process-global mutation lock, common post-lock finalization, deterministic model updates, serial fail-stop execution, deep platform tree operations, no-replace destinations, rename-cycle names, overlap refresh, and partial-failure reporting.
12. Add immediate `:FredLink`, buffer-directory path resolution, and global-lock integration.
13. Run complete instance-lifecycle, configuration, layout, FileType, action, reveal/navigation, mutation, failure-injection, multi-view, overlap, Linux/Windows, watcher, and large-tree suites before declaring version one stable.

Each phase must preserve read-only usability before mutation support is enabled.

## 26. Acceptance Criteria

Version one is ready when:

1. FRED builds locally as a required Rust `cdylib` and loads on Linux and Windows for every exact allowlisted Neovim 0.12 patch-version and pinned `nvim-oxi` revision pair; every direct `nvim_adapter` call passes the pair's compatibility harness, and missing native libraries produce actionable hard errors without a fallback runtime.
2. The Lua facade exposes `build`, `setup`, `new`, the explicit hidden-file visibility and sort getters/setters, registry lookups, and function-valued `actions`, with no compatibility aliases for superseded hidden-file interfaces; setup validates complete presentation configuration and commands remain convenience wrappers.
3. `:Fred` creates an unnamed instance and opens a responsive flat `filetype=fred`, `buftype=acwrite` View of a local root.
4. Multiple Views of one canonical root share immutable snapshots while retaining independent Lua presentation state, Rust projection reconciliation, and intent.
5. A scan generation publishes exactly one immutable current snapshot only after all requested scopes are terminal; partial and failed scopes remain non-authoritative in that result.
6. Dirty Views retain immutable edit-base snapshots across external namespace changes.
7. Ignore, hidden-file display, sort, depth, expansion, collapse, scan cancellation, scan failure, scan limits, reveal, layout, and watcher gaps never imply deletion.
8. Watcher establishment is automatic; establishment failure produces a degraded warning without globally blocking unrelated valid apply operations.
9. Configured metadata columns render as aligned virtual text without changing editable path lines; FRED displays no `/NNN` ID prefix or `../` row, and built-in icon, permission, size, and timestamp columns handle unavailable metadata with placeholders.
10. Column and Lua SortSpec requirements are fetched as the union of attached View requirements; a MetadataBatch with either a stale metadata generation or a mismatched source root revision is rejected independently, metadata-only changes never create namespace conflicts or deletion intent, and metadata sorting cannot reorder a dirty View's real lines.
11. Shared cache state is released by reference ownership and bounded shared policy; instance `delay_ms` independently destroys only undisplayed instances, and every numeric `0` disables only its documented condition.
12. External namespace changes refresh clean Views and rebase dirty Views within covered scope.
13. Same-named files in different directories retain distinct EntryIds.
14. Ordinary edits and `:FredNew file`/`:FredNew dir` create, move, rename, replace, and delete files and directories from final desired state.
15. Every listed Neovim 0.12 pair uses buffer-local `p`/`P` wrappers that preserve normal put behavior and record only qualified same-RootSession provenance; cross-session `gp` errors and foreign or unobserved insertions remain CREATE.
16. Source retained produces COPY; one destination with removed source produces MOVE; multiple destinations with removed source produce copies followed by source deletion.
17. `gp` copies selected top-level entries into a directory and generates deterministic `_N` names.
18. DELETE_TREE and COPY_TREE each appear as one logical operation without advance descendant enumeration.
19. Removing a directory row hides unedited descendants and evacuates explicitly moved descendants before DELETE_TREE.
20. Directory COPY_TREE and MOVE to self or a descendant are rejected.
21. Directory MOVE emits one parent move plus only explicitly required descendant operations in conditional dependency order.
22. Immutable planner probes and repeated final-preflight probes reject missing sources, wrong source kinds, occupied destinations, required parents neither existing nor created by the plan, unrelated planned-parent occupants, non-directory existing parents or nearest ancestors, and parent traversal through symlinks; a planned occupant vacates before replacement parent creation, which precedes every descendant installation.
23. Destination collision never overwrites, including a destination that appears after final preflight.
24. Invalid or conflicted planning results open non-confirmable diagnostics and preserve dirty edits; only valid non-empty plans open one all-or-nothing confirmation preview; an empty valid plan clears modified state without preview.
25. Every plan carries distinct edit-base and planned-root revisions, and stale revision, changedtick, or intent-generation checks abort before mutation.
26. Every post-lock outcome leaves Apply, processes queued refresh demand, and releases the process-global mutation lock.
27. The global lock prevents concurrent apply or `:FredLink` across same, unrelated, and overlapping roots.
28. Cross-filesystem file and directory MOVE executes as copy to the final destination followed by source deletion only after copy success.
29. The executor stops on the first execution error and runs no later operation.
30. Completed-node results update the in-memory namespace model deterministically; partial execution reports completed, failed, and unstarted operations, abandons failed/unstarted initiating-View intent, and re-renders from that model.
31. Filesystem refresh remains best effort; refresh failure reports through Neovim without adding a special state or blocking a later apply.
32. Successful and partially failed mutation invalidates and refreshes overlapping RootSessions on a best-effort basis.
33. File content, permissions, size, and timestamp changes do not create FRED namespace conflicts.
34. External rename is handled conservatively without inode/file-key identity inference.
35. `:FredLink {from} {to}` executes immediately with buffer-directory relative resolution and exclusive destination creation; every success or failure releases the global lock, namespace-changing outcomes invalidate and refresh overlapping RootSessions on a best-effort basis before release, and a later apply or `:FredLink` can proceed.
36. All Rust unit, Lua unit, integration, property, and headless Neovim tests pass.
37. FRED runs without Oil installed and contains no Oil runtime dependency.
38. A direct instance resolves setup options plus explicit options; a child defaults to the parent's current root, snapshots the parent's current hidden-file visibility and SortSpec into its immutable initial configuration, applies child overrides, and does not duplicate setup callbacks or retain a live link to later parent changes.
39. Each live InstanceId owns at most one buffer/View, optional names are unique, root is resolved during `new()`, and resolved configuration is immutable.
40. One instance buffer may appear in multiple windows/tabpages; `open()` focuses an existing current-tab display by default, while explicit `new_display = true` creates another display without duplicating View or RootSession state.
41. `layout` supports buffer, float, split, vsplit, and tab through native Neovim Lua APIs without NUI or layout-specific types leaking into core Instance/View/RootSession interfaces.
42. Borrowed windows restore prior/alternate/scratch buffers, owned displays close when safe, and FRED never deletes the final ordinary window merely to hide itself.
43. `hide()` targets the current or explicit window, no public `close()`/hide-all exists, and `toggle()` hides all current-tab displays including floats or opens when none exist.
44. Default directory selection creates a child instance, transfers the triggering window, preserves normal Neovim history behavior, and leaves the parent buffer available until instance cleanup.
45. Instance lifecycle derives Created/Open/Hidden display state through authoritative window reconciliation, defers `WinClosed` reconciliation, sends validated `BufDelete`/`BufWipeout` directly to terminal Destroyed, cancels cleanup while any valid window displays the buffer, and treats `delay_ms = 0` as disabled.
46. A hidden dirty buffer may be reopened and applied; terminal instance destruction discards unapplied text/intent and releases references without executing a filesystem operation.
47. Keymaps merge by mode/lhs, accept only functions or `false`, and built-in/custom actions share explicit `action(ctx, opts?)` context.
48. `actions.select()` dispatches entry kinds to action functions; the default opens files normally and directories in inherited child instances.
49. `vim.b[bufnr].fred` is complete before fixed `filetype=fred` autocmds, and users can retrieve live instances by InstanceId, name, or buffer.
50. `on_attach` accepts a function or list, runs once per instance buffer after keymaps and FileType in setup-then-instance order, and any error destroys the partial instance and is rethrown.
51. Reveal validates only root-relative paths, establishes exact ignore overrides and obtains ancestor/target EntryFacts before hidden classification or target projection, persistently enables hidden-file visibility when needed using the same intent-capture and history-reset transition, uses normal expansions without pinning, and never changes root.
52. Same-instance navigation rebases the flat projection to a new current root only from a clean View; default child-instance navigation preserves dirty parent buffers.
53. FRED has no generic temporary filter feature and runs without Oil installed or any Oil runtime dependency.
54. Lua receives bounded catalogs containing every entry in the View-requested scopes, applies the sole hidden-file predicate with hidden-directory subtree propagation, performs the fixed single-key sort, and submits only ordered visible EntryIds without expanding coverage.
55. Rust accepts at most one atomic `{ CatalogFrameToken, ordered_visible_ids }` submission, requires exact equality for every sealed frame identity field, rejects stale/reused tokens and unknown/duplicate IDs, accepts any unique frame-ID subset as projection absence, and preserves/deduplicates protected dirty/conflict/validation rows with stable anchors and deterministic fallback.
56. Hidden-file visibility changes may run while dirty, use the frozen full-catalog order, preserve unapplied intent and `'modified'`, and reset undo history after every actual boolean change even when visible order is unchanged; same-value changes are no-ops. Sort changes reject dirty Views unless the normalized spec is unchanged, never create a deferred request, and reset history only after an actual clean-View real-row reorder.
57. The public SortSpec is exactly one built-in key plus required `direction` and `case_sensitive` fields, is independently complete and wholesale-replacing at every configuration/runtime boundary, sorts missing metadata last, reverses only the primary field, keeps all tie-breakers ascending, and exposes no custom comparator, multi-key, directory-first, or configurable missing-value interface.
58. Lua presentation work is bounded and cancellable, yields event-loop progress on large catalogs, rejects non-boolean hidden callbacks, and cancels stale presentation generations before they can submit or commit.

## 27. Residual Risks

- Filesystem races can still occur after direct final preflight begins. FRED intentionally compares source path and kind rather than inode or platform file ID, so a rare external same-path, same-kind replacement may be treated as the original source.
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
- Retained immutable snapshots increase memory use while dirty Views remain open; soft memory limits cannot evict state that remains referenced.
- Metadata columns and metadata-backed Lua sorting add stat calls and platform-specific normalization cost, especially for large covered roots.
- Lua mirrors each View's bounded entry catalog and classification cache, increasing main-thread work, memory use, and garbage-collection pressure at large coverage limits.
- A slow or failing `is_hidden_file` callback can delay or abort a presentation transaction; its exception is intentionally exposed directly by Neovim.
- Actual hidden-file visibility changes intentionally reset Neovim undo history even when visible rows do not change; clean sort changes reset history only when real-row order changes; unapplied intent survives either reset, but earlier text undo steps do not.
- A failed or unavailable stat may leave a placeholder until the next metadata refresh; it must never be interpreted as a missing namespace entry.
- Extmark and provenance behavior under complex edits requires headless and property testing.
- `nvim-oxi` couples the native build to the exact Neovim 0.12 patch-version/revision pairs listed in build configuration; every new pair requires an explicit `nvim_adapter` compatibility run and may require a rebuild.
- Opening many windows for one instance increases Neovim window/UI cost even though scan and snapshot data remain shared.
- Hidden dirty instances may be destroyed after `cleanup.instance.delay_ms`; unapplied edits are intentionally lost at that point.
- Native jumplist/alternate-buffer behavior depends on normal Neovim window buffer switching and is not supplemented by a Fred history stack.
- User `on_attach`, keymap, hidden-file, and custom action callbacks execute inside Neovim and may fail or introduce user-side latency.

These risks are explicit constraints. The design favors a small namespace planner, direct platform operations, one global mutation gate, serial fail-stop execution, and best-effort refresh over complex guarantees for rare interruption scenarios.
