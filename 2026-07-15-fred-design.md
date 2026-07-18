# FRED Design Specification

**Status:** Accepted direction; implementation pending
**Date:** 2026-07-15
**Last revised:** 2026-07-17
**Project:** `fred.nvim`
**Expansion:** Filesystem Representation Editor

## 1. Summary

FRED is a buffer-native file browser for Neovim. It projects selected parts of a local filesystem root into an ordinary editable buffer. The user expresses a desired namespace with normal text editing, then ordinarily writes the current visible Open FRED buffer to plan, preview, confirm, and execute that Instance's corresponding filesystem operations. No public apply/save entrypoint exists.

FRED is independent. It does not depend on, integrate with, or call Oil. It may borrow general ideas such as editable directory buffers, entry identity, asynchronous rendering, operation planning, and confirmation before mutation, but it owns its scanner, snapshots, views, planner, executor, watcher integration, and user interface.

FRED uses a required Rust `cdylib` built locally by the user or plugin manager. The filesystem core is Rust-first and uses `nvim-oxi` for native module integration, immutable namespace snapshots, session-scoped `NodeId` identity, scanning, metadata, intent, conflict handling, planning, and execution. Rust never reads or writes a FRED View buffer and does not know whether Lua presents a flat or tree UI. Rust combines each RootSession snapshot with per-View intent into a projection-neutral, tree-shaped `ViewSemanticFrame`. Lua owns the stable Instance facade, resolved configuration, actions, keymaps, attach callbacks, registry lookup, native Neovim layouts, lifecycle autocmds, Instance cleanup timers and hidden-group LRUs, materialization and presentation state, sibling-local sorting, the common projector, the buffer projection engine, range extmarks, metadata decorations, cursor/undo behavior, yank/put provenance, and bidirectional projection codecs. Version one ships only the internal flat codec. Rust RootSession lifetime follows ordinary strong/weak ownership rather than policy-driven retention or eviction. FRED supports only explicitly validated Neovim 0.12 patch-version and pinned `nvim-oxi` revision pairs. It does not publish precompiled binaries or provide a pure-Lua fallback.

The version-one renderer is a flat list of complete View-relative paths. It is not an indented tree, sidebar, or connector-glyph hierarchy. A View may materialize different directories to different depths while every displayed line remains a complete path. This is a Lua rendering contract rather than a Rust namespace limitation: Rust supplies the same `ViewSemanticFrame` and parent/child membership that a future built-in tree codec would consume. Version one exposes no tree mode, projection toggle, placeholder `view_mode`, public custom renderer, or projection-backend registration API.

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
5. Let ordinary text edits express file and directory creation, rename, move, copy, replacement, and deletion, with the user's ordinary write of the current visible Open FRED buffer as the sole buffer-planned mutation entrypoint.
6. Capture copy provenance from ordinary Vim yank and put operations where Neovim exposes or FRED wraps the operation.
7. Track every existing or pending namespace node with one opaque RootSession-scoped `NodeId`, independently of its current path or lifecycle state.
8. Treat directory copy, move, and deletion as single logical tree operations rather than per-descendant plans.
9. Prevent hidden, ignored, unscanned, collapsed, or depth-excluded entries from being interpreted as deletions.
10. Share immutable filesystem snapshots and watcher coverage between exact-root Views and explicit parent/child/navigate lineage Views in one RootSession while preserving per-View root, intent, expansion, and presentation state.
11. Detect external namespace changes and update attached views automatically within materialized coverage.
12. Reject invalid paths, missing sources, wrong source kinds, required parents that neither exist nor are created by the same plan, and occupied destinations before mutation.
13. Never silently overwrite an existing destination.
14. Preview every non-empty operation plan and require one all-or-nothing confirmation.
15. Permit only one FRED mutation anywhere in the Neovim process at a time, whether entered through the current buffer's write handler or `:FredLink`.
16. Execute filesystem operations serially, stop immediately on the first execution error or accepted cooperative cancellation, and never run later operations.
17. Refresh affected views after success, execution failure, or execution cancellation on a best-effort basis.
18. Remain responsive while scanning, copying, and rendering large directory sets.
19. Expose a stable Lua Instance lifecycle with explicit actions, buffer metadata, attach callbacks, and layout-independent display control.
20. Build from source for validated Neovim and `nvim-oxi` combinations.
21. Let Lua render optional metadata columns such as icons, permissions, size, and timestamps without changing editable path syntax or exposing any buffer-layout concept to Rust.
22. Release the whole RootSession and its snapshots, metadata/cache state, watcher coverage, and provisional work naturally through strong-reference ownership; scan limits bound each requested generation but do not promise a cumulative memory bound for a still-live RootSession.
23. Let one instance own one editable buffer while displaying that buffer in multiple windows and sharing RootSession data by reference.
24. Let users create named or unnamed instances, inherit configuration, choose action functions per keymap mode, and reveal files through normal flat expansion.
25. Destroy Hidden instances through exactly two Lua-owned, OR-combined triggers: their per-instance hidden delay and their setup-defined cleanup-group LRU capacity, while discarding only unapplied buffer state and never mutating the filesystem.
26. Keep the Rust/Lua data path projection-neutral: Rust publishes semantic nodes and parent/child membership, Lua produces one common depth-first `ProjectedRow` sequence, and only the internal Lua codec determines editable line syntax.

## 3. Non-Goals For Version One

Version one will not:

- depend on or interoperate with Oil;
- ship an indented-tree renderer, flat/tree toggle, connector-glyph hierarchy, permanent sidebar layout, placeholder projection-mode option, or public custom projection codec;
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
- provide a public apply/save method, command, action, or mapping outside ordinary current-buffer write handling;
- treat sorting, hidden-file display, ignoring, expansion, collapse, layout, reveal, or navigation changes as filesystem operations.

These constraints keep version one centered on paths, identity, immutable snapshots, views, desired-state planning, and a small fail-stop executor.

## 4. Why FRED Is Independent

Oil's current model binds one buffer to one directory URL and treats that buffer's lines as the direct children of one directory. Listing, cache ownership, rendering, parsing, mutation destinations, and watching derive from that single parent URL.

A flat editable view represents entries owned by many directories and may materialize different subdirectories to different depths. It requires per-row root-relative paths, shared root snapshots, multi-directory collision checks, subtree normalization, selective watcher coverage, and multi-view refresh semantics.

Oil's maintainer described recursive display as incompatible with Oil's current architecture in [issue #26](https://github.com/stevearc/oil.nvim/issues/26#issuecomment-1379844295). [PR #305](https://github.com/stevearc/oil.nvim/pull/305#pullrequestreview-1941933152) also demonstrated that merely allowing path separators in existing entries can miss unseen destination conflicts and risk data loss.

FRED therefore borrows concepts, not runtime modules or private interfaces. If any Oil source code is copied rather than independently reimplemented, its MIT license and copyright notice must be preserved.

## 5. Core Invariants

The implementation and tests must preserve these invariants.

### 5.1 Buffer Intent Mutates Only Through The Two-Stage Current-Buffer Write Pipeline

Editing, deleting, putting, expanding, collapsing, sorting, changing hidden-file visibility, refreshing, revealing, navigating, hiding, or destroying a FRED buffer does not mutate the filesystem by itself.

The only buffer-planned filesystem mutation entrypoint is the user's ordinary write of the current FRED buffer through that buffer's Lua `BufWriteCmd` handler. Standard full-buffer write variants such as `:write`, `:update`, and `:wq` may share the handler. The `BufWriteCmd` handler rejects a target filename different from the FRED buffer's private URI. Buffer-local `FileWriteCmd` and `FileAppendCmd` handlers reject ranged and append writes before ordinary Neovim file I/O; they never capture intent or enter preparation, confirmation, locking, or execution. One write processes only the current Instance, never all loaded instances. The handler first validates that the current buffer is exactly that Instance's buffer and that the Instance is Open and displayed; Created, Hidden, Destroyed, non-current, alternate-target, ranged, and append-write cases cannot enter the pipeline.

The internal write pipeline has exactly two native stages:

1. Lua captures the current semantic rows and calls `apply_prepare(view_handle, CaptureRequest)`. Rust validates and records normalized ViewIntent, performs the initial immutable direct namespace probes, and calls the pure Planner.
2. A rejected capture or plan returns non-confirmable diagnostics. A valid plan returns one sealed, opaque, one-shot `PreparedApply` capability containing the immutable `ValidPlan`, including Planner-owned affected paths and final-probe expectations, plus its authority stamps. A non-empty plan also returns a sanitized logical preview; an empty plan returns no preview.
3. Lua displays the complete non-empty preview and asks for one all-or-nothing confirmation. Preview cancellation calls `apply_finish(prepared, Cancel)` or drops the capability, acquires no mutation lock, performs no filesystem action, and preserves the dirty buffer and ViewIntent.
4. For a confirmed non-empty plan, or immediately for an empty plan, Lua establishes its `ApplyBufferGuard`, prevents reentrant writes and new ordinary edits, reads the observed changedtick and write epoch, and calls `apply_finish(prepared, Execute { observed_changedtick, write_epoch })`.
5. `apply_finish` consumes the capability exactly once. Replay, duplicate finish, a foreign capability, or a previously dropped capability is rejected. Before an already-requested execution cancellation may abandon intent, Apply upgrades the initiating handles and validates current View ownership plus the sealed View/root/projection/edit-base/planned-root/intent/changedtick authority and execute witness. A stale old capability returns a pre-execution stale error and preserves newer intent.
6. For a current non-empty Execute without an already-observed cancellation, Apply acquires the process-global mutation lock, registers the `ValidPlan`'s affected paths in the process-wide active mutation fence, invalidates existing overlapping scan generations, repeats direct namespace probes under the lock, and invokes the serial Executor only when every check succeeds. A current cancellation observed after authority validation may instead take the clean path before the gate or first syscall. Existing and newly created RootSessions consult the active fence at scan start and publication. An empty plan performs the same authority/witness checks without acquiring the mutation gate, registering a fence, or invoking Executor.
7. Every empty-plan, successful, execution-failed, or accepted execution-cancelled outcome that produces a clean semantic frame is synchronously rendered and committed by Lua with `'modified' = false` before `BufWriteCmd` returns. Success and empty plans then return write success; execution failure and cancellation return write failure so `:wq` remains open. Only best-effort refresh and ancillary UI work are deferred. Every terminal or abort path releases `ApplyBufferGuard`.

No public Instance apply/save method, apply action, apply command, or default apply mapping exists. The sealed capability, execution task handle, cancellation token, and Apply pipeline are internal implementation details and do not create another mutation entrypoint. `:FredLink` is the sole immediate mutation helper outside buffer write and uses the same process-global mutation runner and finalization path.

### 5.2 Projection Absence Is Not Deletion

An entry absent from the current buffer because of ignore rules, Lua hidden-file filtering, Lua sorting, baseline depth, local collapse, scan limits, scan cancellation, scan failure, or missing watcher coverage remains part of filesystem state.

After Lua synchronously renders a `ViewSemanticFrame`, it commits the frame token, ordered visible `NodeId` sequence, and resulting changedtick. Rust validates frame currency, membership, and uniqueness and records a `CommittedProjection`. A node omitted from the current frame or projection because of ignore rules, hidden-file filtering, materialization, collapse, scan status, or presentation policy remains projection absence. Only an existing NodeId that belonged to the initiating `CommittedProjection` and is absent from Lua's later semantic capture can generate removal intent. Dirty, pending, conflict, and validation state is already applied by Rust when it builds the next `ViewSemanticFrame`; Lua does not infer filesystem deletion from catalog omission. Metadata columns are projection-only: a missing, stale, or failed metadata value cannot create, remove, rename, or delete intent.

### 5.3 Path Is Not Identity

Every namespace node receives one opaque `NodeId` scoped to its RootSession. Existing files, directories, symlinks, pending creates, and pending copy destinations all use the same identity type; lifecycle state is separate from identity. Rust may store NodeId as a compact strongly typed value, but its Lua representation is an opaque namespaced string such as `n:<root-session-namespace>:<allocator-value>`. The prefix separates the identity transport domain from editable path text; it does not encode Existing, PendingCreate, or PendingCopy lifecycle. Lua compares and stores the whole token without parsing or constructing it. A filename such as `99` or `n:rs7:99` remains an unrelated path value and cannot collide with a NodeId because FRED never derives identity from editable text. Editing or FRED-moving a path changes location without changing NodeId. A pending node that is successfully created becomes an existing snapshot node without changing NodeId. External rename remains conservative delete-plus-create because FRED does not infer identity from inode or platform file keys.

### 5.4 Direct Namespace Checks Authorize Operations

A partial or degraded browse snapshot does not globally disable apply. Before planning, the runtime captures immutable direct namespace-probe results for every source, destination, existing required parent, and planned-parent chain required by the current intent. For a parent created by the same plan, probes establish that the planned path is either absent or occupied by a base identity that the plan explicitly removes or moves away before creating the directory; an occupied path must still match the base identity's expected path and kind under FRED's ordinary path/kind checks. Probes also establish that the chain's nearest existing ancestor is a directory reached without traversing a symlink. The pure planner consumes those probe results without performing I/O. Final preflight repeats the same probes under the process-global mutation lock.

An operation is blocked when its required namespace state cannot be established, including a missing source, wrong source kind, required parent that neither exists nor is created by the plan, non-directory parent or nearest existing ancestor, parent traversal through a symlink, invalid path, or occupied destination. A parent that the same valid plan creates is not rejected merely because it is currently absent.

Directory COPY_TREE and DELETE_TREE do not require advance enumeration of every descendant.

### 5.5 Watchers Are Advisory But Active

Watchers actively trigger coalesced rescans and view updates. Watcher events can be missed, merged, delayed, or unavailable, so watcher health is not a correctness prerequisite for apply.

Watcher failure produces a visible degraded state. Direct final preflight remains authoritative for the current plan.

### 5.6 Only One FRED Mutation Runs At A Time

One process-global mutation lock covers all RootSessions and every mutation entered through a FRED buffer's `BufWriteCmd` handler or `:FredLink`.

The lock is acquired after preview confirmation and current authority validation, before final preflight. A non-empty mutation registers one process-wide active mutation affected-path fence while holding the gate and before enumerating RootSessions or issuing the first filesystem syscall. The fence remains registered through serial execution, deterministic publication, overlap invalidation, and common finalization; only the finalizer removes it and releases the gate. Preview and empty plans hold neither.

The global gate prevents concurrent mutation between overlapping roots such as `/project` and `/project/src`. The active affected-path fence additionally prevents a RootSession created or started after initial session enumeration from scanning or publishing across the in-flight mutation.

### 5.7 Directory Operations Are Subtree Transformations

A directory rename or move generates one primary directory MOVE. Unedited descendants are rebased in the desired model and do not generate redundant actions.

A directory deletion generates one DELETE_TREE. A directory copy generates one COPY_TREE. The planner does not expand either operation into per-descendant logical actions.

### 5.8 The Final Desired State Determines COPY Versus MOVE

FRED plans from the final captured buffer state, not from the sequence of editing commands:

- source remains plus provenance destination: COPY or COPY_TREE;
- source removed plus one provenance destination: MOVE;
- source removed plus multiple provenance destinations: COPY each destination, then DELETE the source.

### 5.9 Execution Is Serial, Fail-Stop, And Cooperatively Cancellable

Execution nodes run serially in dependency order. The first failed system call stops the batch immediately. An accepted execution-time cancellation stops at the next safe checkpoint, except that a request observed only after the final execution node has successfully completed is classified as Success. Later nodes are not run, completed effects remain, and FRED does not automatically retry or reverse them. A blocking system call that cannot be interrupted may finish before cancellation is observed.

System API success is trusted. Known successful effects, including successful internal steps of a failed or cancelled compound node, are applied deterministically to the in-memory namespace model before the initiating View is synchronously re-rendered for a clean outcome. Failure reports `COMPLETED / FAILED / UNSTARTED`; cancellation reports `COMPLETED / CANCELLED / UNSTARTED`. Filesystem refresh is best effort, and neither outcome adds a journal, rollback, recovery, unknown-state, or apply-blocking state machine.

### 5.10 Instance Destruction Discards Only Unapplied Buffer State

An instance may lose every displaying window while its buffer remains loaded and modified. `Open -> Hidden` inserts it as the newest member of its configured cleanup-group LRU and, when `delay_ms > 0`, starts its one-shot hidden timer. Neither action applies, discards, confirms, or mutates the filesystem. The user may reopen the buffer and write it before either cleanup trigger destroys it.

Exactly two OR-combined automatic destroy triggers exist for a Hidden instance: expiration of its per-instance hidden delay and overflow of its setup-defined group capacity. `Hidden -> Open` removes the instance from the LRU and cancels its timer; rehiding reinserts it as newest. Capacity enforcement destroys the oldest Hidden instances through the same idempotent terminal destroy path until the group is within capacity. `capacity = 0` retains no Hidden instances, while `delay_ms = 0` disables only the timer. Cleanup is entirely Lua-owned and applies only to Instance lifecycle.

When explicit destruction, external buffer wipeout, timer expiry, or group eviction destroys the instance, FRED discards that buffer text, View intent, undo history, and edit-base reference and detaches its View. Rust knows nothing about cleanup timers, groups, or capacities. The RootSession registry retains only Weak references; attached Views and active tasks retain strong references. Detaching the last View cancels root work that has no consumers. After active tasks release their final references, the RootSession, snapshots, metadata/cache state, and watchers drop naturally. Destroying unapplied text is not a filesystem operation and never enters planner or executor flow.

One instance owns one buffer and View. Multiple windows may display that buffer, and a displayed/Open instance is never a cleanup-group LRU member and has no running hidden timer.

## 6. Platform, Build, And Runtime Architecture

### 6.1 Supported Neovim And Platform Combinations

FRED supports only exact Neovim 0.12 compatibility pairs listed in the build configuration. Each pair contains:

- one exact Neovim 0.12 patch version;
- one pinned `nvim-oxi` revision using its `neovim-0-12` feature.

The build helper accepts only a listed pair and rejects every unlisted patch/revision combination rather than guessing a feature selection. Neovim 0.11 and earlier, nightly builds, and unlisted 0.12 patches are unsupported. The first candidate is Neovim 0.12.4, but it becomes a supported pair only after the FRED compatibility harness validates every direct native integration call used for module registration, argument/result conversion, `AsyncHandle` wakeups, and main-thread event delivery. Buffer access, range extmarks, virtual text, changedtick, cursor, and undo operations are Lua responsibilities and are tested through the headless Neovim harness rather than Rust native-bridge calls.

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

### 6.3 Projection-Neutral Rust Core And Lua Buffer Runtime

Rust owns:

- canonical native and encoded paths, the RootSession-scoped `NodeId` allocator, immutable `RootSnapshot` values, normalized parent/child membership, metadata observations, watcher state, and coverage accounting;
- Rust Views identified by `ViewId`, each with a movable `root_node_id`, immutable edit base, semantic `ViewIntent`, conflict/validation state, and `CommittedProjection`;
- application of `ViewIntent` to a snapshot to produce a projection-neutral, tree-shaped `ViewSemanticFrame` containing clean, dirty, pending, moved, conflict, and validation nodes;
- scanning, metadata collection, watching, direct namespace probes, the pure Planner, the serial Executor, deterministic known-effect publication, overlap invalidation, and the process-global mutation gate;
- the deep internal `apply` module, which owns `apply_prepare`/`apply_finish`, sealed `PreparedApply` capabilities, authority checks, execution cancellation coordination, the process-wide active mutation affected-path fence, common post-lock finalization, and refresh scheduling;
- bounded frame events, generic worker-task cancellation/transport, generation checks, and main-thread event notification.

Lua owns:

- build integration and native-library discovery/loading;
- `setup()` configuration capture and direct/child instance inheritance;
- the public Instance object and live-instance registry lookups;
- action functions, mode-grouped buffer-local keymaps, FileType metadata, and `on_attach`;
- native Neovim layout construction, window ownership, open/hide/toggle/focus behavior, highlight definitions, lifecycle autocmds, and all Instance cleanup timers/group LRUs;
- per-Instance materialization and expansion state, per-View catalog mirrors, hidden-file classification, sibling-local sorting, depth-first traversal, columns, and the common projector;
- the internal `ProjectionCodec` boundary, with only `FlatProjectionCodec` implemented in version one;
- every FRED buffer projection operation: line rendering and capture, full-line range extmarks, virtual text, cursor reconciliation, changedtick reads, `'modified'`, undo-baseline resets, internal-sync suppression, `ApplyBufferGuard`, preview/confirmation UI, execution-progress cancellation requests, and synchronous clean outcome commits; and register-qualified provenance integration through the separate `ProvenanceStore`: `TextYankPost`, `p`/`P` wrappers, register snapshots, and inserted-range qualification.

The forward boundary passes opaque RootSession/View/Node handles and bounded generation-tagged `ViewSemanticFrame` events. Lua builds one ordered `ProjectedRow` sequence, synchronously renders it, and submits a `ProjectionCommitRequest { frame_token, ordered_node_ids, changedtick }`. The reverse boundary passes semantic capture through `apply_prepare` and returns either diagnostics or a sealed `PreparedApply` plus an optional sanitized preview. Lua cannot inspect or modify the plan. After confirmation, `apply_finish` consumes that same capability with an observed changedtick/write-epoch witness; the running task exposes only generic internal progress and cancellation plumbing. Rust never parses line syntax or receives extmark positions, indentation, connectors, or cursor positions. Lua never creates filesystem operations, and Rust never executes user callbacks or directly reads or writes a FRED View buffer.

### 6.4 Background Work And Task State

Version one does not use Tokio or a general-purpose asynchronous runtime.

Each RootSession is in one of three conceptual states:

```text
Idle
Scan
Apply
```

`Idle` means no RootSession filesystem task is running. `Scan` covers ordinary browse, expansion, and refresh scans. `Apply` begins only when a current non-empty Execute path acquires the process-global mutation lock; preparation, preview, pre-gate cancellation, and empty-plan synchronization do not enter Apply.

Rules:

- a newer ordinary scan request may cancel and replace an older scan generation;
- `apply_prepare` performs capture validation, initial probes, and pure planning without acquiring the mutation lock or changing RootSession task state;
- after current Execute authority is validated and the global gate is acquired, Apply registers the immutable `ValidPlan` affected paths in one process-wide active mutation fence before enumerating RootSessions or issuing the first syscall;
- every existing overlapping RootSession generation is invalidated, and any RootSession created later must consult the active fence at scan start and publication; an overlapping scan request is queued or cancelled, and no overlapping candidate may publish until the fence is removed;
- scan events from obsolete or fenced generations are ignored, and no candidate begun before or during the fence may publish across the mutation;
- watcher events received during Apply are coalesced for the post-apply refresh;
- the execute task exposes its existing internal cancellation handle to Lua progress UI without creating a public apply/save entrypoint;
- stale authority wins over an already-requested execution cancellation: only a current accepted Execute may abandon intent and clean-sync before the gate or first syscall;
- execution cancellation is cooperative and checked before nodes and at safe checkpoints in long tree operations; a request observed only after the final node succeeds is Success;
- every post-lock outcome, including final-preflight failure, execution error, and execution cancellation, leaves Apply through one common finalization path that coalesces and schedules queued refresh demand before removing the active affected-path fence and releasing the global mutation lock;
- another FRED buffer write or `:FredLink` receives a Busy error while the global mutation lock is held.

Long work uses:

```text
std::thread
std::sync::mpsc::sync_channel
AtomicBool cancellation
nvim_oxi::libuv::AsyncHandle
```

Worker threads never call Neovim functions directly. They enqueue bounded events and use `AsyncHandle::send()` only to wake the main loop. Because wakeups may coalesce, each main-thread callback drains only one entry-count-limited or time-limited batch. If events remain, it calls `AsyncHandle::send()` again before returning. A producer that continues filling the bounded channel therefore cannot turn one callback into an unbounded main-thread drain. Cancellation is a request: an uninterruptible blocking system call may return before the worker observes it, while COPY_TREE and other long internal loops check it at documented safe checkpoints.

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

The current presentation values become part of the child's immutable construction snapshot; later parent changes do not update the child. Explicit child options override the copied values. SortSpec is an explicit exception to every deep-merge rule: a setup SortSpec, direct-instance override, copied parent-current SortSpec, or explicit child override is independently validated as a table containing exactly `by`, `direction`, and `case_sensitive`, and the selected table replaces the previous SortSpec wholesale. No SortSpec field is inherited from another table. This inheritance keeps columns, keymaps, layout, resolved `cleanup.instance` values, presentation behavior, and callbacks consistent while navigating through newly created instances. Setup-only `cleanup.groups` is not copied into direct or child Instance options and is immutable after setup. The inherited `on_attach` list already contains setup callbacks and is not prefixed with setup callbacks a second time.

`name` is optional. Every instance has an opaque unique InstanceId; a non-empty name must be unique among live instances. Creating a second live instance with the same name is an explicit error. Destroying the old instance releases the name for reuse. Automatically created child instances are unnamed unless the action supplies a name.

The resolved configuration is immutable after `new()`. `instance:config()` returns a read-only construction snapshot; `instance:hidden_files_visible()` and `instance:sort()` return read-only copies of committed runtime presentation state. Runtime changes use explicit methods such as `set_columns`, `set_hidden_files_visible`, `set_sort`, `navigate`, and per-call `open({ layout = ... })`; FRED does not expose a generic `set_config()` or `update_config()`.

In Created state, presentation setters only validate and update Lua PresentationState. There is no View or semantic frame yet, so they perform no intent capture, metadata request, projection, buffer synchronization, or history reset. `open()` creates the View from those values. Open and lifecycle-Hidden instances use the transaction rules in Section 13. Destroyed instances reject presentation operations through the ordinary lifecycle guard. `instance:refresh()` is separate: it accepts any Open Instance with at least one valid display, even when its buffer is not current. `:FredRefresh` is current-buffer-only and validates that the current buffer belongs to a visible Open FRED Instance.

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
  hidden cleanup timer
  cleanup group and optional Hidden-LRU membership
```

Multiple instances may share a RootSession in two cases: exact canonical-root reuse and explicit parent/child/navigate lineage within that session's namespace. They own distinct buffers, Views, intent, undo history, columns, expansion state, and Lua presentation state while sharing immutable snapshots, NodeIds, scan cache, metadata cache, and watcher coverage. An independent direct `fred.new()` reuses only an exact-root RootSession; it does not automatically adopt an existing ancestor session or dynamically merge overlapping sessions.

The public lifecycle states are:

```text
Created
  instance exists; root and configuration are resolved; buffer does not exist

Open
  buffer exists and at least one window displays it

Hidden
  buffer exists and no window displays it; instance is in its group's LRU and its hidden timer may run

Destroyed
  terminal state; buffer, View, windows, timers, group membership, name registration, and RootSession references are released
```

Transitions are driven by observed Neovim state as well as instance methods:

```text
new()                         -> Created
open()                        -> Open
last displaying window gone  -> Hidden
open() while Hidden           -> Open
group capacity overflow       -> Destroyed
cleanup delay expires         -> Destroyed
buffer delete or wipeout      -> Destroyed
destroy()                     -> Destroyed
```

FRED uses `WinNew`, `BufWinEnter`, `WinEnter`, `WinClosed`, and `BufHidden` as display-reconciliation triggers; event-local counters are not authoritative. Reconciliation derives displays from valid windows that actually contain the instance buffer. Because `WinClosed` fires before the closing window disappears from window queries, its reconciliation is deferred to the next main-loop turn. A validated `BufDelete` or `BufWipeout` for the instance's current buffer bypasses display reconciliation and invokes the terminal destroy path directly.

Instance cleanup is configured with inherited per-instance values and setup-only groups:

```lua
cleanup = {
  instance = {
    delay_ms = 300000,
    group = "default",
  },
  groups = {
    default = {
      capacity = 10,
    },
  },
}
```

`cleanup.instance` is resolved into each direct or child Instance. `delay_ms` is not keyboard-idle time; it is a one-shot delay beginning on `Open -> Hidden`. `group` must name a setup-defined group, and an unknown group fails during `new()` validation. `cleanup.groups` is setup-only, process-wide, and immutable after setup; direct and child options cannot override or define it.

On `Open -> Hidden`, Lua inserts the Instance as newest in its group's LRU, starts its timer only when `delay_ms > 0`, and immediately enforces capacity by destroying oldest Hidden members until within the group's limit. `capacity = 0` therefore retains no Hidden instances. On `Hidden -> Open`, Lua removes the Instance from the LRU and cancels its timer. Rehiding starts a fresh one-shot timer and reinserts it as newest. Timer expiry and LRU eviction both call one idempotent terminal destroy path, so races between them are harmless. No other automatic destroy trigger exists.

Unapplied buffer edits and intent are discarded without mutating the filesystem. FRED does not prompt, auto-write, or block child-instance window transfer or hiding merely because the old instance buffer is modified: the user may return to that buffer and write it before destruction, or allow cleanup to discard the edits. Same-instance `navigate()` still requires a clean View as specified in Section 7.8. A forced external buffer deletion has the same discard semantics. This is not filesystem undo or recovery: text that was never successfully written simply ceases to exist with the destroyed buffer.

Destroy detaches the View from its RootSession. The registry's Weak reference does not keep the RootSession alive; other Views and active tasks do. Last-View detach cancels work with no consumers, and final task release naturally drops the RootSession and all Rust-owned snapshots, metadata/cache state, and watchers.

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

The default directory-selection action creates a child instance, creates its buffer, and transfers the triggering FRED window to that child buffer instead of stacking another float or split. When the selected directory belongs to the parent's RootSession, the child shares that RootSession and its NodeIds, uses the selected directory NodeId as `root_node_id`, and snapshots only the parent's expansion entries inside that subtree; later parent and child expansion changes are independent. The parent loses that window, becomes Hidden if no other window displays it, enters its cleanup-group LRU as newest, and starts its timer when enabled. Window buffer replacement uses normal Neovim behavior; FRED does not maintain a separate back/forward history or override native buffer, alternate-buffer, or jumplist behavior.

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

Version one's internal `FlatProjectionCodec` encodes each editable line as exactly one canonical path relative to the current View root. Rust never parses this syntax: Lua's buffer projection engine decodes edited lines into semantic rows containing NodeId, desired path candidate, kind, and optional provenance, and Rust repeats canonical path validation before updating View intent.

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

The editable buffer line remains exactly one encoded View-relative path. Lua renders columns as Neovim extmark virtual text at byte column zero with `virt_text_pos = "inline"`; they are not inserted into the buffer line, are not returned by the codec as path text, and cannot be mistaken for a path component. The path is always the final visible field. NodeId remains in a Lua-owned full-line range-extmark binding and is never rendered as editable text.

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

FRED displays neither Oil's concealed `/NNN` internal-ID prefix nor a `../` parent row. NodeId stays in a Lua-owned extmark binding, and the View root NodeId is intentionally not represented by a row.

A string selects defaults. A table uses the first element as the column name and may set `width`, `align` (`"left"`, `"center"`, or `"right"`), `format` for time columns, `highlight`, `separator`, and column-specific icon options. The path column is implicit and cannot be removed.

The default separator is one display cell. Default minimum display widths are two cells for `icon`, nine for `permissions`, eight for `size`, nine for `type`, and the formatted display width for each timestamp. Missing values use `-` and the same padding/alignment as present values. `width` is a minimum measured with Neovim display width, not byte length; a wider value expands that column for the whole projection rather than being truncated. All rows in one projection therefore use the same stable boundaries. This follows Oil's projection-wide width/alignment pass in [`view.lua`](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/lua/oil/view.lua#L650-L818), adapted to virtual text rather than editable prefix text.

The internal Lua definition is intentionally small:

```text
ColumnDefinition {
  name
  required_metadata
  default_width
  default_alignment
  render(view_node, metadata, column_options) -> highlighted text chunks
}
```

Version one registers only built-in Lua columns; it does not expose a public custom-column, renderer, projection-codec, or backend registration API. Column definitions do not parse mutable path text. Rust only collects the metadata fields requested by attached Views and returns normalized values in semantic frame nodes.

Each column definition declares the metadata it needs. `icon` and `type` use already-known entry kind; `permissions`, `size`, and time columns request stat metadata. Lua's current SortSpec separately declares the one sort field it needs. A RootSession requests the union of metadata fields required by attached View columns and Lua presentation states, just as coverage is the union of their directory scopes. This follows Oil's on-demand `require_stat` approach in [`files.lua`](https://github.com/stevearc/oil.nvim/blob/b73018b75affd13fa38e2fc94ef753b465f770d7/lua/oil/adapters/files.lua#L50-L225). Metadata failures render a placeholder and a non-blocking status rather than making a namespace scan authoritative or failed. Refreshing metadata may re-render clean and dirty Views, but changes to permissions, size, or timestamps never create FRED namespace conflicts.

Changing columns only reprojects the View and requests any missing metadata. It never changes dirty state, edit-base revisions, or filesystem intent. Lua sorting may use `path`, `type`, `size`, `mtime`, `ctime`, `atime`, or `birthtime`; unavailable metadata always sorts after available values and does not block apply.

A metadata revision may cause Lua to rebuild and commit a new ProjectedRow order only for a clean View. A dirty View updates cached metadata values and virtual columns but retains its frozen sibling-local traversal order. When it becomes clean after apply, successful empty-plan synchronization, or ordinary undo removes its intent, Lua recomputes from current metadata and the already-current SortSpec; no deferred requested sort exists. A SortSpec or metadata change that actually changes real-row order clears Neovim undo history and establishes a new baseline. If real-row order is unchanged, presentation state may still advance but history is not cleared.

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

Commands remain convenience entry points over instances and actions. There is no apply/save command or action; buffer-planned mutation is entered only by writing the current Open FRED buffer:

```text
:Fred [root]                 create an unnamed instance and open it
:FredRefresh                 refresh the current visible Open FRED instance
:FredDepth {n|all}           change baseline recursion depth
:FredExpand [depth]          expand the current directory
:FredCollapse                collapse the current directory
:FredPasteInto               paste a FRED-aware yank into the current directory
:FredNew {file|dir}          insert an explicitly typed pending entry
:FredLink {from} {to}        immediately create a symlink outside the planner
```

There is no general temporary filter command. Hidden-file visibility is handled by `view.hidden_files_visible`, `is_hidden_file`, `instance:set_hidden_files_visible()`, and `actions.toggle_hidden_files`. There is no `:FredSort`, sort picker, or built-in sort action; mappings may call `ctx.instance:set_sort({...})` directly.

### 7.8 Expansion, Reveal, And Navigation

A View has a configurable baseline depth plus ordinary directory expansions. Lua stores expansion depth by RootSession-scoped directory NodeId, not by buffer path or current-root-relative identifier. The version-one projection remains a flat list of complete paths relative to the instance's `current_root`, while the shared NodeId expansion state is independent of the codec.

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
3. materialize or request semantic node facts for each required ancestor and the target without first committing the target into the visible projection;
4. classify the ancestor chain and target as those facts arrive;
5. if any required fact is hidden, commit the same persistent `hidden_files_visible = true` transition as the public setter;
6. wait for the target NodeId to appear in the committed visible projection;
7. move the cursor in the selected instance window.

This order lets reveal traverse ignored or previously uncovered ancestors before hidden classification is possible. A real hidden-file visibility change first captures semantic rows into Rust ViewIntent, obtains a new ViewSemanticFrame containing all dirty/pending/conflict/validation nodes, reprojects it in Lua, clears Neovim undo history, and establishes a new baseline. Existing unapplied intent remains and keeps `'modified'` set. Reveal does not create a pinned row, temporary reveal mode, or `clear_reveal()` API. Its expansions remain normal NodeId-keyed View state and users collapse them normally. Enabling hidden-file visibility remains global to the instance and persists until the user toggles it again.

`navigate(relative_directory)` is an explicit alternative to creating a child instance. The target must resolve to a real directory NodeId inside the current RootSession; a directory symlink is a leaf and cannot be expanded or used as same-session inline navigation. Navigation changes the View's `root_node_id` while preserving InstanceId, RootSession, NodeIds, buffer, windows, resolved config, keymaps, attach effects, columns, current hidden-file visibility, current SortSpec, and NodeId-keyed expansion state. It invalidates the old View generation, updates `current_root` and `vim.b[bufnr].fred.root` from the selected node's canonical location, re-bases View-relative paths, commits a new projection, and establishes a new undo baseline. FileType and `on_attach` do not run again.

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

`navigate` is not another expansion operation. Because one instance owns one buffer, direct same-instance navigation requires a clean View. Expansion NodeIds that still occur under the new root remain active automatically; expansion entries outside it remain harmless instance-local state and become active again if a later same-session root includes them. The default directory action creates a child instance instead, snapshots the parent's expansion entries inside the selected subtree, and leaves the old dirty parent buffer available for a later current-buffer write until either Hidden cleanup trigger destroys it. Opening a directory outside the current session, including a resolved directory-symlink target, creates or reuses an exact-root RootSession rather than aliasing target children beneath the symlink node.

### 7.9 Write Semantics

The only buffer-planned filesystem mutation entry is a user's ordinary full-buffer write of the current visible Open FRED buffer. `:write`, `:update`, and `:wq` may invoke the same buffer-local Lua `BufWriteCmd` handler. One invocation processes only the Instance that owns that current buffer; it never discovers, batches, or writes other loaded FRED instances. FRED exposes no public apply/save method, command, action, mapping, or replacement entrypoint.

Every FRED buffer installs three buffer-local write-command handlers. `BufWriteCmd` is the sole preparation handler. `FileWriteCmd` always rejects ranged writes, and `FileAppendCmd` always rejects append writes; because these command autocmds replace Neovim's ordinary operation, both errors occur before any ordinary file I/O. The two rejection handlers do not capture intent, prepare a plan, preview, acquire the mutation lock, or mutate the filesystem.

Before preparation, the `BufWriteCmd` handler validates all of the following:

- the event buffer is the current buffer;
- the buffer is the initiating Instance's registered buffer;
- the Instance is Open and at least one valid window displays that buffer;
- the command's target filename is exactly the FRED buffer's private URI after the same normalization used for buffer identity.

An alternate target filename is rejected rather than treated as an export. A Hidden Instance cannot enter the write/Apply pipeline or refresh. The user must reopen it, making it Open and displayed, before `:write`, `instance:refresh()`, or `:FredRefresh` is accepted; only the write and `:FredRefresh` additionally require that buffer to be current.

The Lua handler captures semantic rows through the active `ProjectionCodec` and `BufferProjectionEngine`, then calls `apply_prepare(view_handle, CaptureRequest)`. Rust validates and records normalized ViewIntent, performs the initial immutable probes, and calls the pure Planner. A rejected result returns diagnostics. A valid result returns a sealed one-shot `PreparedApply`; a non-empty plan also returns its complete sanitized logical preview, while an empty plan has no preview.

Lua displays and confirms only the preview. Preview cancellation calls `apply_finish(prepared, Cancel)` or drops the capability, acquires no global lock, creates no `ApplyBufferGuard`, records no clean projection, and preserves the dirty text, ViewIntent, and undo history. For confirmation, or immediately for an empty plan, Lua establishes one `ApplyBufferGuard` before reading the execute witness. The guard rejects reentrant writes and prevents new ordinary edits until terminal handling finishes. Lua then calls `apply_finish(prepared, Execute { observed_changedtick, write_epoch })`. The sealed capability is consumed exactly once; changing the preview cannot change the stored `ValidPlan`.

The `BufWriteCmd` handler waits for a terminal Apply outcome while pumping bounded Neovim events. The running execute task may expose internal progress and a cancellation control. Before a cancellation already requested on that task may abandon intent, Rust validates the sealed capability's current initiating View ownership and View/root/projection/edit-base/planned-root/intent/changedtick authority against the execute witness. Stale authority returns a pre-execution stale error and preserves newer intent. Once a current Execute is accepted, cooperative cancellation may produce the clean execution-cancellation outcome even before gate acquisition or the first syscall.

For Neovim 0.12.4 `acwrite` behavior, every outcome that produces a clean frame is synchronized inside the active `BufWriteCmd`: Lua revalidates the same live Instance and View, generations, clean frame, `ApplyBufferGuard.write_epoch`, and expected unchanged changedtick; renders with edit capture suppressed; commits the projection; reconciles the cursor; establishes a new ordinary undo baseline; and sets `'modified' = false` before returning. Success and an empty plan return write success only after that synchronous commit, allowing ordinary `:wq` to exit. Execution failure or accepted execution cancellation returns write failure only after the same synchronous clean commit, so `:wq` remains open with a clean buffer. Best-effort refresh and ancillary progress/diagnostic UI may run later, but the clean projection and modified-state transition are never deferred.

Preview cancellation, validation failure, stale authority, Busy lock contention, final-preflight failure, or another pre-execution abort performs no clean render, preserves buffer text, ViewIntent, `'modified'`, and undo history, and releases `ApplyBufferGuard` if one exists before returning write failure. A clean-outcome synchronization that finds a destroyed Instance, missing View, generation mismatch, frame mismatch, write-epoch mismatch, or unexpected changedtick does not resurrect or overwrite anything; it releases `ApplyBufferGuard` and returns a terminal write failure/no-op for the buffer while preserving the filesystem result. A synchronous render or ProjectionCommit error is fail-fast, releases the guard, and returns write failure. Every successful synchronization also releases the guard before `BufWriteCmd` returns.

Rules:

- every non-empty plan is previewed and confirmed;
- a valid empty plan skips preview but still passes through sealed `PreparedApply` consumption and the final changedtick/currency witness;
- `:wq` closes only after successful execution or a successful empty plan has synchronously committed the clean frame and cleared `'modified'`;
- preview cancellation and every pre-execution error, including stale authority and Busy, preserve `'modified'`;
- successful execution produces a synchronously committed clean View and a new ordinary undo baseline before write success;
- execution failure or execution cancellation synchronously commits a clean View from known successful effects, then returns write failure and leaves `:wq` open;
- hiding an instance or transferring its window does not write, discard, or prompt; the Hidden buffer remains editable state until reopened or destroyed;
- destroying an instance discards unwritten buffer text and intent without filesystem mutation, while destruction after mutation begins does not interrupt required fail-stop finalization; terminal handling always releases `ApplyBufferGuard`;
- FRED does not export its path list through ordinary write syntax.

Hidden-file visibility changes are presentation synchronizations, not writes. A setter that supplies the already-committed boolean is a no-op. Every actual boolean change captures semantic rows into Rust ViewIntent and, after Lua renders and Rust accepts the resulting ProjectionCommitRequest, clears Neovim undo history and establishes a new baseline even when visible real-row membership/order happens to remain unchanged. Existing intent and `'modified'` survive. An identical normalized SortSpec is also a no-op. A different SortSpec updates presentation state, but clears history only when its committed sibling-local traversal actually changes real-row order.

Normal Vim editing remains the primary interface for create, rename, move, copy, replacement, and delete; writing the current Open FRED buffer is the only way those edits enter preparation and execution.

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
- descendants explicitly moved outside the deleted subtree remain explicit ViewSemanticFrame nodes and are evacuated before DELETE_TREE;
- undoing the directory-row removal restores its descendant projection;
- filesystem mutation still waits for confirmed write.

DELETE_TREE does not require the directory to be expanded or fully scanned. Preview shows the directory path and indicates that all contents are included. Cached counts may be displayed, but FRED does not calculate counts as a prerequisite.

### 8.4 Copy Provenance From Normal Editing

Extmarks track positions through supported buffer edits but do not travel through yank or put. FRED therefore records provenance separately on every supported Neovim 0.12 pair.

- FRED installs buffer-local `p` and `P` wrappers because Neovim 0.12 does not expose `TextPutPre` or `TextPutPost`;
- the wrappers execute normal put behavior, preserving the chosen register, count, direction, undo behavior, and cursor semantics, then record the inserted range;
- Lua `TextYankPost` stores provenance for the affected register as its register name, text and register type snapshot, `RootSessionId`, and source NodeIds;
- a wrapper attaches provenance only when the register actually used still has the recorded text and type and belongs to the same RootSession; a mismatch invalidates that register's provenance;
- provenance is valid only in the same RootSession; wrappers do not attach foreign-session provenance;
- insertions not observed with valid same-session provenance are ordinary unbound CREATE declarations, even if their text matches an existing row.

FRED does not guess provenance from matching text or from another RootSession.

Typical behavior:

```text
yy + p + edit destination
```

If the source remains in the final desired state, the result is COPY or COPY_TREE. If the source is removed and one provenance-linked destination remains, the result is MOVE.

A `dd` followed by an observed `p` gives the Planner a removed source plus one provenance-linked destination and may therefore normalize to MOVE after the destination path is edited. Capture and provenance do not perform the rebinding; the provisional destination remains distinct until Snapshot, under Apply orchestration, derives the authoritative terminal identity from actual execution effects.

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

`:FredPasteInto`, mapped to `gp`, copies the most recent FRED-aware yank into the directory under the cursor. The yank must belong to the current View's RootSession. Parent/child Views that explicitly share that RootSession may exchange provenance; an independent exact-root or overlapping RootSession produces an explicit error. Users can expand directories in the same flat View when source and destination must share provenance.

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

COPY_TREE writes directly to the final desired destination. It does not use an intermediate copy location. If copying fails or is cancelled partway, a partial destination may remain. Executor stops and reports the node outcome and known successful internal effects; Apply owns affected-path invalidation and best-effort View refresh.

A directory cannot be copied to itself or to any descendant of itself after canonical path comparison.

### 8.9 Cross-Filesystem Move

A same-filesystem MOVE uses the platform rename operation when possible.

When rename reports a cross-filesystem condition, the logical MOVE expands internally into:

```text
COPY or COPY_TREE source -> final destination
DELETE or DELETE_TREE source
```

Source deletion runs only after the copy operation reports success. If both copy and source deletion succeed, the source NodeId moves to the final destination. If copy succeeds but source deletion fails, or cancellation is observed after copy success and before source deletion starts or at a cancellable checkpoint before deletion completes, both namespace entries remain: the source retains its NodeId and the destination receives a distinct NodeId because one identity cannot represent two actual entries. When the logical MOVE originated from a PendingCopy destination, that destination keeps its already allocated provisional NodeId; a direct Existing-path MOVE without such a destination receives a newly allocated destination NodeId. If deletion completes successfully and cancellation is first observed afterward, the MOVE is Success instead. If COPY_TREE fails or is cancelled before its copy step completes and leaves a partial destination, later refresh discovers unrepresented partial nodes with new destination NodeIds rather than transferring source identities. FRED stops and reports the partial result.

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

### 10.1 Identity And Path Types

The native model uses three opaque identity domains:

```text
RootSessionId  identifies one namespace session
ViewId         identifies one semantic/edit View
NodeId         identifies one existing or pending namespace node inside a RootSession
```

`NodeId` is the only node-identity type. Files, directories, symlinks, pending creates, pending copy destinations, dirty moved nodes, and conflicted nodes do not use parallel identity types. Lifecycle state is separate from identity. The complete scope is conceptually `(RootSessionId, NodeId)`, although the serialized Lua value carries a transport namespace so Lua handles one opaque string, for example `n:rs7:99`. Rust may implement that value as a newtype over allocator fields rather than a string. Lua must not parse the prefix or allocator value, and prefixes such as `existing:` or `pending:` are forbidden because a pending identity can become durable without changing NodeId.

Revision and generation types remain distinct:

```text
RootRevision
MetadataRevision
ScanGeneration
CoverageGeneration
IntentGeneration
FrameSequence
ProjectionRevision
ChangedTick
```

Rust distinguishes:

```text
SessionRelativePath  relative to RootSession.session_root
ViewRelativePath     relative to View.root_node_id
RelativePathCandidate submitted by Lua before Rust validation
EntryName            one canonical encoded path component
```

Rust validates and converts every Lua path candidate. Lua never authorizes native path joining or root escape.

### 10.2 Immutable RootSnapshot And Node Lifetime

A published filesystem snapshot is immutable and carries a monotonically increasing revision. The RootSession root is itself a directory NodeId but is never rendered as an ordinary row.

```text
NodeObservation = Confirmed
                | RetainedUnverified { source_revision, reason }

SnapshotNode {
  id: NodeId
  parent_id: optional NodeId   -- none only for the session root
  name: EntryName
  session_relative_path: SessionRelativePath
  kind: file | directory | symlink | other
  symlink_info: optional SymlinkInfo
  observation: NodeObservation
}

RootSnapshot {
  root_session_id: RootSessionId
  revision: RootRevision
  root_node_id: NodeId
  nodes_by_id: Map<NodeId, SnapshotNode>
  children_by_parent: Map<NodeId, Arc<[NodeId]>>
  node_id_by_session_path: Map<SessionRelativePath, NodeId>
  directory_status: Map<NodeId, Complete | Partial | Failed>
}
```

`children_by_parent` is normalized namespace membership, not a View's sort order. A Snapshot builder creates the node table, parent index, child membership, and path index together and publishes only a self-consistent immutable value. Every non-root node has exactly one directory parent, every child list agrees with `parent_id`, each path agrees with its parent and name, and symlinks have no child membership even when their targets are directories.

Directory membership is authoritative only when that directory's status is `Complete`. A Complete result replaces the prior direct-child set and may prove that an old child disappeared. A Partial or Failed result merges newly observed children into the prior snapshot but carries forward every previously known unobserved child as `RetainedUnverified`; it never converts non-observation into deletion. Retained nodes remain in path and parent/child indexes and may be displayed with degraded status. A later Complete result, successful execution model update, or authoritative direct probe may confirm or remove them. Initial partial scans cannot invent unknown entries, but entries never committed to a projection cannot be interpreted as deletions.

A scan generation builds one candidate snapshot through bounded internal worker events without mutating `current_snapshot`. Scan progress may be reported while work runs, but candidate nodes do not enter Lua's editable projection. A generation publishes exactly one new immutable `current_snapshot` only after every requested scope reaches a terminal complete, partial, or failed result; only then does Rust build the next ViewSemanticFrame. Late or cancelled generations cannot publish, and candidate data never advances a dirty View's edit base.

NodeId lifetime rules are:

- an unchanged same-path, same-kind node in one snapshot lineage ordinarily retains NodeId;
- a FRED rename or MOVE retains NodeId while changing path and parent membership;
- a pending create or provenance destination receives a NodeId before planning; a destination that remains CREATE, COPY, or COPY_TREE after final-state normalization keeps that NodeId after successful creation;
- when one provenance destination plus source removal normalizes to MOVE, Planner records a planned identity disposition from the provisional destination to the source NodeId, but capture, ViewIntent construction, preview, and preparation do not apply that transition;
- external rename is represented conservatively as deletion of the old NodeId plus creation of a new NodeId;
- same-path, same-kind external replacement may be treated as the previous node because FRED intentionally does not use inode or platform file keys;
- successful same-filesystem MOVE and fully successful cross-filesystem MOVE retain the source NodeId at the destination; only terminal Snapshot application then retires a PendingCopy destination's provisional NodeId and emits its authoritative transition to the source NodeId;
- when a cross-filesystem MOVE copies successfully but source deletion fails, or cancellation is observed after copy success and before deletion starts or before a cancellable deletion completes, the source retains its NodeId and the now-separate destination receives a distinct NodeId; a PendingCopy-derived destination retains its provisional NodeId, while a direct Existing-path MOVE allocates a new destination NodeId; successful deletion followed only by a later cancellation observation remains a fully successful MOVE;
- a partial failed or cancelled COPY_TREE destination discovered by refresh receives new destination NodeIds for nodes not already represented by pending destination identities rather than inheriting source identities.

Published snapshots remain alive while referenced by Views or active tasks. Optional permission, size, and timestamp observations live in a separate metadata snapshot keyed by NodeId plus observed path. Metadata enrichment may re-render Lua columns without advancing RootRevision or a View edit base.

### 10.3 RootSession, Sharing, And Movable View Roots

```text
RootSession {
  id: RootSessionId
  session_root: CanonicalRoot
  current_snapshot: Arc<RootSnapshot>
  root_revision
  metadata_snapshot
  metadata_revision
  node_id_allocator
  scan_cache
  scan_generation
  metadata_generation
  watcher_state
  coverage_by_view
  metadata_reference_counts
  task_state: Idle | Scan | Apply
}
```

The RootSession does not own the process-global mutation lock. The runtime registry stores Weak references for exact canonical session roots; attached Views and active tasks hold strong references.

Sharing rules are explicit:

- direct `fred.new({ root = ... })` reuses only an exact-root RootSession;
- same-instance navigation to a descendant keeps the current RootSession and changes only the View's `root_node_id`;
- a child Instance opened on a directory inside the parent's session shares the parent RootSession and NodeIds;
- independent overlapping direct roots are not automatically merged or reparented;
- a directory outside the current session, including a resolved directory-symlink target, creates or reuses its own exact-root RootSession.

This avoids creation-order-dependent ancestor-session adoption while allowing ordinary descendant navigation and child Instances to preserve NodeIds and expansion state. Separate overlapping RootSessions remain consistent because every mutation first registers a process-wide affected-path fence consulted by current and future sessions, then invalidates and refreshes overlaps after the terminal result.

Last-View detach cancels work with no consumers. After active tasks release the final strong reference, the RootSession, snapshots, metadata/cache state, coverage bookkeeping, and watchers drop naturally. Rust implements no cleanup timer, capacity, LRU, sweeper, or object-level eviction policy.

### 10.4 Rust View, Intent, And Committed Projection

Rust View state contains no buffer or presentation objects:

```text
View {
  id: ViewId
  root_session: Arc<RootSession>
  root_node_id: NodeId
  root_status: Present | Missing { last_known_session_path }
  view_generation
  edit_base: Arc<RootSnapshot>
  edit_base_revision: RootRevision
  intent: ViewIntent
  committed_projection: optional CommittedProjection
  frame_sequence
}
```

`root_node_id` is the current namespace root for that View. The FlatProjectionCodec renders paths relative to this node. Changing `root_node_id` inside the same session rebases View-relative paths without reallocating descendant NodeIds.

Lua captures the editable buffer into projection-neutral rows:

```text
SemanticCaptureRow {
  node_id: NodeId
  desired_path: RelativePathCandidate
  desired_kind
  copy_from: optional { root_session_id, node_id }
}

CaptureRequest {
  view_id
  committed_projection_revision
  changedtick
  rows: [SemanticCaptureRow]
}
```

When Lua discovers an unbound row, it requests one or more new NodeIds from the Rust View, attaches them to the rows, and then submits the capture. Rust validates projection revision, NodeId scope and uniqueness, existing-kind preservation, pending-node ownership, same-session provenance, canonical path legality, and duplicate destinations. Capture records only desired rows and normalized provenance facts. It does not classify COPY versus MOVE and never returns or applies a NodeId rebinding. Rust derives `removed_existing` only by comparing the accepted capture with the initiating `CommittedProjection`; projection absence outside that baseline remains non-deletion.

Normalized semantic intent is stored as:

```text
NodeOrigin = Existing
           | PendingCreate
           | PendingCopy { copy_from: NodeId }

ViewIntent {
  generation: IntentGeneration
  root_node_id: NodeId
  existing_changes: Map<NodeId, ExistingDesiredChange>
  removed_existing: Set<NodeId>
  pending_nodes: Map<NodeId, PendingDesiredNode>
  validation: Map<NodeId, [ValidationError]>
  conflicts: Map<NodeId, [NamespaceConflict]>
}
```

Rust never stores FlatProjectionCodec text as intent. It stores validated desired View-relative paths anchored to the stable `root_node_id`, kinds, identity, and provenance, and derives canonical session paths from the root node's current location whenever it builds a semantic frame, probes, or plans. Therefore a dirty View's pending creates, copies, and path edits follow a same-session MOVE of its root NodeId while preserving their relative locations. Planner may mark a unique PendingCopy destination as the logical destination of a removed source, but ViewIntent retains both the provisional destination NodeId and `copy_from` until a terminal Apply outcome. CREATE/COPY destinations retain their allocated NodeIds; successful MOVE identity retirement is committed only by Snapshot application of known successful effects.

After Lua renders a frame, it submits:

```text
ProjectionCommitRequest {
  view_id
  frame_token
  ordered_node_ids: [NodeId]
  changedtick
}

CommittedProjection {
  revision: ProjectionRevision
  source_frame
  ordered_node_ids
  node_id_set
  changedtick
}
```

The call is synchronous and fail-fast. There is no projection lease, rollback protocol, recovery state, or automatic renderer retry. `CommittedProjection` is the authoritative baseline for distinguishing a user-deleted bound node from a node that never entered the editable projection.

### 10.5 ViewSemanticFrame And Bounded Transport

Rust applies the View's semantic intent to the current snapshot before Lua projection:

```text
ViewNodeState {
  origin: Existing | PendingCreate | PendingCopy
  dirty: boolean
  validation_errors
  conflicts
}

ViewNode {
  id: NodeId
  parent_id: optional NodeId
  view_relative_path: ViewRelativePath
  name: EntryName
  kind
  observation: Confirmed | RetainedUnverified
  state: ViewNodeState
  metadata: optional normalized metadata
}

UnplacedViewNode {
  id: NodeId
  path_candidate
  desired_kind
  state
}

ViewFrameToken {
  root_session_id
  view_id
  view_generation
  root_revision
  metadata_revision
  intent_generation
  frame_sequence
  sealed_identity
}

ViewSemanticFrame {
  token: ViewFrameToken
  root_node_id: NodeId
  root_status: Present | Missing { last_known_session_path }
  nodes_by_id: Map<NodeId, ViewNode>
  children_by_parent: Map<NodeId, Arc<[NodeId]>>
  unplaced_nodes
  directory_status
}
```

The frame already reflects existing renames/moves, pending creates and copies, removed nodes, directory-delete descendant suppression, explicitly evacuated descendants, conflict state, and validation state. Lua does not merge a second protected-row model into ordinary browse rows.

Large frames cross the native boundary through bounded events:

```text
FrameBegin { token, root_node_id, root_status }
FrameNodeBatch { token, nodes }
FrameDirectoryBatch { token, parent_id, child_ids, status }
FrameUnplacedBatch { token, nodes }
FrameComplete { token }
FrameCancelled { token }
```

Lua rejects mixed or stale tokens. Rust child lists represent membership only; each Lua View applies its own sort.

### 10.6 Lua Materialization, Projector, Codec, And Buffer Engine

Each Instance/View owns Lua state:

```text
ExpansionState {
  by_node: Map<NodeId, DepthLimit>
}

MaterializationState {
  baseline_depth
  expansions: ExpansionState
  explicit_includes
  generation
}

PresentationState {
  hidden_files_visible
  sort
  columns
  materialization
  presentation_generation
  catalog_mirror
  hidden_classification_cache
  frozen_dirty_sibling_orders: optional
}
```

Expansion is keyed by NodeId. Same-session navigation therefore preserves descendant expansion automatically. A child Instance snapshots the parent's expansion entries inside the selected directory subtree and then evolves independently. A FRED directory MOVE retains expansion because the source NodeId is stable. Planner normalization alone does not change Lua state. When terminal Apply handling returns an authoritative `NodeIdTransition` for a successfully realized PendingCopy-derived MOVE, Lua atomically remaps expansion, cursor identity, and row bindings while synchronously rendering the clean frame. Preview cancellation and pre-execution errors apply no transition; COPY and cross-filesystem identity-split destinations retain their own NodeIds and do not inherit source expansion by default.

The Lua catalog mirror contains the current frame token, root NodeId/status, nodes by NodeId, child membership, unplaced nodes, directory statuses, and completion state. A Missing root status is a View-level diagnostic and does not cause Lua to replace the existing editable buffer with an empty projection. The common projector:

1. applies hidden-file classification with hidden-directory subtree propagation;
2. applies the current SortSpec only within each direct sibling list;
3. applies baseline depth, NodeId-keyed expansions, and explicit includes;
4. performs depth-first traversal so every materialized directory subtree remains contiguous;
5. produces one ordered `ProjectedRow` sequence containing NodeId, parent NodeId, depth, View-relative path, name, kind, state, and metadata.

Flat and future tree rendering consume the same ProjectedRow order. Version one implements only an internal sealed codec:

```text
ProjectionCodec {
  encode(ProjectedRows) -> lines, binding specs, decorations
  decode(buffer lines, NodeId bindings) -> SemanticCaptureRows
}

FlatProjectionCodec.encode(row) = row.view_relative_path
FlatProjectionCodec.decode(line) = desired View-relative path candidate
```

Version one exposes no projection-mode option, tree codec, toggle, custom renderer, or public codec registration seam.

The Lua `BufferProjectionEngine` owns the buffer number, extmark namespace, current frame token, projection revision, NodeId bindings, internal-sync depth, changedtick, line rendering and capture, full-line range extmarks, virtual text, cursor reconciliation, `'modified'`, and undo-baseline resets. During semantic capture it consumes optional normalized `copy_from` facts from `ProvenanceStore`, but it does not own register snapshots, `TextYankPost`, `p`/`P` wrappers, or inserted-range provenance state. Rust does not read or write those objects.

When a clean View becomes dirty, Lua freezes each currently materialized sibling order rather than one catalog-global order. Metadata may update virtual columns, but no comparator result moves real rows until the View becomes clean. Hidden-file membership changes take filtered subsets or supersets of those frozen sibling orders. Newly discovered nodes use deterministic sibling-local fallback order until clean recomputation.

### 10.7 Revision, Rebase, Lifecycle, And Overlap Behavior

When a new snapshot is published:

- clean Views receive a new ViewSemanticFrame derived from it;
- dirty Views compare their immutable edit base, the new snapshot, and their desired View-relative namespace anchored to `root_node_id`;
- a same-session MOVE of that root NodeId changes the derived session paths but preserves every desired View-relative path;
- nodes carried as RetainedUnverified because their parent scope is Partial or Failed are treated as Unknown, not external deletion or kind conflict;
- non-overlapping authoritative namespace changes merge automatically;
- conflicting confirmed paths, kinds, identities, or destinations remain frame nodes with conflict state and block the affected plan;
- the dirty View advances its edit base only after a successful semantic rebase.

File contents, permissions, size, and timestamps are not part of this conflict model.

Window entry/exit updates Lua Instance display handles and cleanup state without changing Rust View intent. Buffer wipeout, explicit destroy, timer expiration, or LRU eviction enters one idempotent terminal path that invalidates Instance/View generations, discards unwritten buffer text and semantic intent, releases coverage and edit-base references, and detaches the View.

Same-instance `navigate` requires a clean View and a real directory NodeId inside the current RootSession. It keeps InstanceId, ViewId, RootSession, NodeIds, buffer, windows, config, presentation values, and expansion state; changes `root_node_id`; invalidates the old frame/projection generation; updates buffer metadata; re-renders rebased View-relative paths; and establishes a new undo baseline. A directory symlink remains a leaf and cannot be same-session expanded or navigated as though it owned target children.

If another FRED View moves this View's root NodeId inside the same RootSession, NodeId preservation keeps `root_status = Present`; `current_root` follows the node's new canonical path, dirty View-relative intent follows that root, and the next frame re-bases descendants. Rust records `root_status = Missing { last_known_session_path }` only when a successful model update, Complete authoritative parent membership, or direct probe confirms that the root NodeId disappeared; Partial/Failed non-observation retains it as unverified and cannot make the View Missing. Lua preserves the current buffer and intent, shows a non-confirmable View-level root-missing diagnostic, permits ordinary refresh and destruction, and rejects write planning and same-instance navigation. FRED does not silently rebind the View to a newly created node at the same path because external identity is conservative.

Creating a child Instance on a directory inside the parent session creates a new View over the same RootSession and selected `root_node_id`, snapshots expansion entries in that subtree, and leaves the parent View and dirty buffer intact. An independent direct nested root may still use another RootSession. Before any mutation syscall, the process-wide active affected-path fence blocks overlapping current or future scan publication; after a successful or partially successful mutation, Apply invalidates every attached RootSession whose session root overlaps an affected path and queues best-effort refresh.

## 11. Identity And Provenance

Lua's shared `BufferProjectionEngine`, not an individual codec and not Rust, owns NodeId-to-row bindings. Every rendered semantic row receives a full-line range extmark mapped to its opaque RootSession-scoped NodeId. The range spans `[row, 0]` through `[row + 1, 0]` with `right_gravity = false`, `end_right_gravity = true`, `invalidate = true`, and `undo_restore = true`. Internal projection rendering runs under a Lua sync guard that suppresses edit capture and recreates bindings without generating filesystem intent.

Identity rules are:

- editing text preserves NodeId while one valid range extmark remains associated with that row;
- moving text preserves NodeId only when Neovim moves that exact range with it;
- deleting the complete row outside the internal-sync guard invalidates its binding;
- one row cannot carry multiple NodeIds, and two captured rows cannot carry the same NodeId;
- a new unbound row receives a Rust-allocated NodeId before semantic capture;
- an Existing node may change desired path but may not directly change kind;
- a PendingCreate or PendingCopy node uses the same NodeId type as an Existing node; CREATE/COPY keeps that NodeId after success, while a unique removed-source destination classified as MOVE remains provisional through planning and is rebound to the source NodeId only when terminal Snapshot application proves the logical MOVE succeeded;
- if a binding is lost for a reason other than range invalidation, Lua may request rebind only through a unique, unconsumed, unchanged base path; ambiguous identity is never guessed;
- external rename is represented conservatively as old-node deletion plus new-node creation.

Extmarks do not travel through yank or put. Lua's `ProvenanceStore` therefore owns register-qualified provenance:

- `TextYankPost` records register name, text, register type, RootSessionId, and source NodeIds;
- buffer-local `p` and `P` wrappers preserve native register, count, direction, undo, and cursor behavior, then inspect the inserted range;
- provenance attaches only when the actual register still matches the recorded text and type and belongs to the same RootSession;
- a valid provenance-linked destination initially receives a newly allocated NodeId with `origin = PendingCopy` and `copy_from = { root_session_id, node_id }`; provenance records no COPY/MOVE decision, and the provisional destination is retired only by an authoritative terminal identity outcome after a successful logical MOVE;
- parent and child Views sharing one RootSession may exchange provenance; an independent overlapping RootSession may not;
- insertions without valid provenance are PendingCreate declarations even when their text matches an existing row.

`BufferProjectionEngine` never stores register provenance. During semantic capture it asks `ProvenanceStore` to qualify observed inserted rows and receives only the normalized optional `copy_from` fact for each row.

Lua submits only the normalized `copy_from` reference. Rust validates same-session source identity and stores it in ViewIntent without classifying the operation. Planner later groups final destinations by source identity and normalizes the final desired state:

```text
source retained + destination
  -> COPY or COPY_TREE

source removed + one destination
  -> MOVE

source removed + multiple destinations
  -> COPY each destination, then DELETE or DELETE_TREE source
```

Hard links are separate namespace nodes and receive separate NodeIds even when the operating system reports shared underlying storage.

## 12. Scanner, Coverage, And Watchers

### 12.1 Scanner Events, Snapshot Publication, And Semantic Frames

The scanner is a cancellable Rust worker task. Lua converts baseline depth, NodeId-keyed expansion, and explicit includes into projection-neutral directory scopes before requesting coverage:

```text
CoverageRequest {
  view_id
  coverage_generation
  scopes: [
    { directory_node_id, depth }
  ]
  ignore matcher
  required metadata fields
  max_entries
  max_directories
}
```

The worker emits bounded events such as:

```text
ScanNodeBatch
MetadataBatch
ScanProgress
DirectoryComplete
DirectoryFailed
ScanComplete
ScanCancelled
```

After a candidate snapshot or semantic intent changes, Rust emits bounded `ViewSemanticFrame` events as defined in Section 10.5. Requirements are:

- allocate or reuse RootSession-scoped NodeIds and construct `nodes_by_id`, `children_by_parent`, path index, directory status, and Confirmed/RetainedUnverified observation state as one self-consistent candidate snapshot;
- emit at most `render_batch_size` node facts per batch and expose every node in each requested scope regardless of Lua hidden-file classification;
- attach `scan_generation` to namespace events and `metadata_generation` plus `source_root_revision` to metadata events;
- discard obsolete scan, metadata, coverage, and View-frame generations;
- consult the process-wide active mutation affected-path fence both before starting a scan generation and immediately before publication; overlapping requests are queued or cancelled, and neither existing nor newly created RootSessions may publish across an active mutation;
- record each requested directory NodeId as Complete, Partial, or Failed; Complete membership may remove absent prior children, while Partial/Failed membership must carry prior unobserved children forward as RetainedUnverified;
- never create children for a symlink or recursively follow a directory-symlink target;
- stop at configured limits and report partial coverage;
- publish exactly one immutable RootSnapshot only after every requested scope in the generation reaches a terminal result;
- retain partial and failed scopes as non-authoritative status inside that published result;
- build each ViewSemanticFrame from the current RootSnapshot plus that View's semantic intent, then stream bounded node and directory-membership batches with one sealed ViewFrameToken;
- keep child membership free of View-specific sort order;
- accept a post-render `ProjectionCommitRequest` only when its frame token is current and every submitted NodeId belongs to that frame exactly once;
- accept a semantic `CaptureRequest` only when its projection revision and changedtick match the initiating Lua buffer state and all NodeIds are current, unique, and valid for that View;
- accept a MetadataBatch only when its metadata generation and source RootRevision are current, every field remains required, and NodeId plus observed encoded path still match the current snapshot;
- attach metadata observations to NodeId plus observed path so a late value cannot bind to a moved or replaced node;
- fetch stat metadata only when a Lua column or SortSpec requires it;
- allow metadata-only enrichment without publishing a namespace snapshot or advancing RootRevision;
- publish accepted metadata by advancing only MetadataRevision and emitting updated semantic frame facts;
- treat metadata collection failure as a per-node placeholder, not a namespace scan failure;
- perform no direct FRED buffer, extmark, virtual-text, cursor, changedtick, or undo operation in Rust.

Default limits remain:

```lua
max_entries = 50000
max_directories = 10000
render_batch_size = 500
```

Reaching a limit or failing an unrelated directory produces a visible partial state; it does not globally disable apply.

### 12.2 Coverage Union

A RootSession scans and watches the union of directories required by attached Views.

Lua derives coverage from:

- each View's baseline depth;
- directory NodeIds in that Instance's ExpansionState;
- precise explicit includes and reveal ancestor chains;
- directories required by Rust ViewIntent, conflict, validation, and unplaced-node context.

The Rust RootSession receives only directory NodeId plus depth scopes and reference-counts their union; it does not know whether a scope originated from a flat expansion, a future tree expansion, reveal, or semantic intent.

Hidden-file visibility and Lua filtering do not reduce or expand coverage and never trigger a whole-root recursive scan. Rust still includes every node from already requested scopes in semantic frame facts. Explicit reveal may add a precise ancestor chain and ignore override; collapsing that chain releases the corresponding coverage normally.

Coverage is reference-counted. When no attached View needs a directory, its watcher may be released and its cached state may cease to be authoritative. COPY_TREE and DELETE_TREE do not add descendant enumeration coverage.

Metadata requirements are also reference-counted as the union of attached Lua View columns and SortSpecs. Removing a column or changing sort reduces future metadata requirements without reducing directory coverage, but it does not promise object-level pruning of metadata already retained by a still-live RootSession. Namespace NodeId, path, kind, parent/child membership, coverage authority, and previously cached metadata may remain until the whole RootSession is released.

### 12.3 Watcher Behavior

FRED automatically attempts to establish watchers for covered directories; version one has no watcher toggle. Established watchers monitor covered directories. Events are debounced and coalesced by affected directory, then trigger incremental scan requests.

FRED-generated mutations also produce watcher events. Planner seals affected paths in `ValidPlan`; Apply uses them to merge watcher demand with explicit overlap invalidation and post-terminal refresh. Executor neither owns affected paths nor refreshes Views.

If watcher coverage cannot be established, FRED displays a degraded warning and leaves manual refresh available. Missing watcher coverage alone does not prevent apply.

### 12.4 RootSession Ownership And Natural Release

FRED has no persistent disk cache in version one and no Rust retention or eviction policy. Rust knows nothing about Instance timers, cleanup groups, LRU capacity, idle age, estimated memory, or cache entry-count thresholds.

The RootSession registry retains Weak references so registry membership alone never extends a session's lifetime. Attached Views and active tasks retain strong references to the RootSession and thereby to the snapshots, metadata/cache state, and watcher resources they require. When a View detaches, it releases its coverage, edit-base, intent, and diagnostic references. If it was the last View, the runtime cancels root work that has no remaining consumers. Ordinary cancellation and generation replacement release provisional scan and metadata state promptly through ownership.

Once active tasks release the final strong RootSession reference, the RootSession, current and obsolete snapshots, metadata/cache state, coverage bookkeeping, and watchers drop naturally. There is no object-level cleanup or pruning, periodic sweeper, LRU, estimated-memory policy, idle-age policy, cached-entry-count policy, or other Rust retention policy.

`max_entries` and `max_directories` bound each requested scan generation, and bounded channels/batches bound provisional task-owned work. They do not bound cumulative cache retained by a still-live RootSession: a long-lived session may retain namespace and metadata state for scopes visited by earlier generations even after current View coverage moves elsewhere. Cancelled or superseded provisional task-owned batches are still released promptly through ordinary ownership, while the only guaranteed release of all retained cache is final whole-RootSession release.

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

`instance:set_hidden_files_visible(value)` validates an actual boolean. `instance:toggle_hidden_files()` inverts the committed value. A same-value call is a no-op. For an actual change on an Open or lifecycle-Hidden instance, Lua first captures semantic rows into Rust ViewIntent, creates a proposed presentation generation, and schedules bounded/cancellable classification over the current ViewSemanticFrame. The public method returns after immediate validation, capture, and scheduling; it does not return a promise or async result object. The committed getter value changes only after Lua renders and Rust accepts the resulting ProjectionCommitRequest. Dirty, pending, conflict, and validation nodes already belong to the ViewSemanticFrame. The Lua synchronizer clears undo history and establishes a new baseline while retaining unapplied intent and `'modified'`, even if visible membership happened not to change. `reveal()` persistently enables visibility when needed and waits for that transition before cursor movement.

Sort uses one complete, fixed-shape specification:

```lua
sort = {
  by = "path", -- path|type|size|mtime|ctime|atime|birthtime
  direction = "asc", -- asc|desc
  case_sensitive = false,
}
```

All three fields are required; unknown or missing fields and values are invalid. SortSpec is never deep-merged: every explicit table at setup, direct construction, parent-current snapshot selection, child override, or runtime setter independently contains exactly these three fields and replaces the prior SortSpec wholesale. After hidden-file filtering, Lua applies the comparator only to each directory's direct children, then performs depth-first traversal. A materialized directory row and its visible descendants therefore remain one contiguous subtree in both flat and future tree renderers. Missing metadata always follows available metadata regardless of direction. Ascending `type` order is directory, file, symlink, other. `direction` reverses only the primary `by` comparison. Normalized name/path, exact name/path, and NodeId tie-breakers remain ascending in both directions.

`instance:set_sort(spec)` first normalizes the complete spec. An identical spec is an idempotent no-op even while dirty. A different spec on a dirty View throws a Lua exception and leaves state unchanged; there is no deferred request. On a clean Open or lifecycle-Hidden instance it creates a proposed presentation generation, requests only missing metadata required by its key, and schedules bounded/cancellable sibling ordering plus depth-first traversal. The method returns after immediate validation and scheduling, without an async result object. The getter changes only after the rendered projection commits successfully. Missing values may initially sort last; later metadata may reorder only a clean View. When a View becomes clean after being dirty, Lua reapplies the already-current sort. A changed SortSpec whose committed result leaves real-row order unchanged does not clear history; an actual real-row reorder does.

Baseline depth semantics:

- `0`: entries directly under root;
- `1`: direct entries plus children of direct directories;
- `all`: no depth limit other than configured scan limits.

Expansion depth is stored by directory NodeId and is relative to that selected directory. `actions.expand` records one level; `actions.expand_recursive` records recursive materialization subject to scan limits. Same-session navigation preserves these NodeIds. A child Instance snapshots expansion entries inside its selected root subtree and then evolves independently.

Changing ignore rules requests a new browse scan. Changing hidden-file visibility only reprojects existing requested coverage. Changing sort changes only Lua presentation state and metadata requirements. Changing baseline depth, NodeId-keyed expansion, or precise reveal coverage updates coverage. None of these changes generates filesystem intent.

Default ordering is ascending, case-insensitive stable lexical ordering within each direct sibling group, followed by depth-first traversal. Flat rendering still displays complete View-relative paths, but a directory's materialized descendants remain adjacent to that directory rather than being globally interleaved with another subtree. Destination collision checks use actual platform/filesystem behavior, while displayed spelling remains unchanged.

## 14. Planner

The Planner is a pure Rust module and one of the two domain dependencies of the deep `apply` module. It receives immutable inputs and returns either a `ValidPlan` or non-confirmable diagnostics. It never performs filesystem I/O, owns no global lock, retains no `PreparedApply`, and mutates neither the filesystem nor a RootSession.

Inputs:

```text
View edit-base snapshot and edit_base_revision
current published snapshot and planned_root_revision
current View root_node_id
captured ViewIntent with desired View-relative paths anchored to that root
current changedtick and intent generation
immutable initial direct namespace-probe results
path and platform rules
execution policy
```

Outputs:

```text
ValidPlan {
  edit_base_revision
  planned_root_revision
  projection_revision
  changedtick
  intent_generation
  planned_identity_dispositions
  ordered execution DAG
  logical preview groups
  affected paths
  final_probe_expectations
}

or

Diagnostics {
  validation errors
  conflicts
}
```

The Planner owns final-state validation and normalization, COPY/MOVE/DELETE classification, planned identity dispositions, logical preview construction, operation dependency ordering, affected-path calculation, and the expectations that final preflight must repeat. It does not own the preview UI, execution report, or application of terminal NodeId transitions. Logical operations shown to users include:

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

`create_symlink` is used only by the immediate `:FredLink` path, which constructs a one-node internal plan and enters the same private mutation runner without creating a buffer `PreparedApply`.

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

The Planner does not reject ordinary file-content, permission, size, or timestamp changes. A destination occupied by another planned identity is allowed only when that identity is moved away or explicitly removed earlier in the same plan. Final execution still uses exclusive/no-replace operations.

`apply_prepare` is the sole orchestration caller for buffer plans. It validates and records the CaptureRequest as ViewIntent, performs the initial probes required by that intent, calls the Planner, and on success seals the immutable `ValidPlan`—including Planner-owned affected paths and final-probe expectations—plus View/root/projection/edit-base/planned-root/intent/changedtick authority and weak initiating handles into one opaque `PreparedApply` userdata. Lua receives the capability plus only the sanitized logical preview. The capability is not stored in a process-wide prepared-plan registry, cannot be cloned or inspected as plan data, and contains only an internal `Fresh -> Consumed` slot. Every accepted semantic capture advances intent generation, so a later prepare makes an earlier unused capability stale even without a registry. Explicit preview cancellation and handle drop release its owned plan memory.

### 14.1 Final-State Provenance Normalization

The Planner normalizes provenance from the final desired state:

```text
source retained + destination added
  -> COPY or COPY_TREE

source removed + one destination added
  -> MOVE

source removed + multiple destinations added
  -> COPY each destination, then DELETE or DELETE_TREE source
```

The normalization chooses operations that reach the final namespace without treating the user's edit sequence as an execution script. A retained source produces COPY/COPY_TREE dispositions that keep each destination's allocated NodeId. One destination plus a removed source produces a planned MOVE disposition that names the source NodeId and any provisional PendingCopy destination, but it does not mutate ViewIntent, Lua bindings, cursor state, expansion state, or RootSnapshot. Multiple destinations plus a removed source retain their destination NodeIds, copy each destination, and delete the source rather than arbitrarily selecting one destination to inherit source identity.

### 14.2 End-To-End Planning And Identity Data Flow

The canonical data flow is:

```text
RootSnapshot
  -> ViewSemanticFrame
  -> Lua projection and ProjectionCommit
  -> user editing plus optional provenance
  -> CaptureRequest
  -> ViewIntent
  -> pure Planner
  -> PreparedApply plus optional immutable preview
  -> serial Executor
  -> known-successful-effect report
  -> terminal Snapshot identity resolution
  -> clean ViewSemanticFrame plus authoritative NodeIdTransition values
  -> synchronous Lua projection commit
```

The stages are:

1. Rust publishes an immutable RootSnapshot and derives a projection-neutral ViewSemanticFrame. Lua projects that frame, renders editable path text, binds each row to its opaque NodeId with a full-line extmark, and commits the frame token, unique ordered NodeIds, and changedtick as CommittedProjection.
2. Editing an Existing row preserves its bound NodeId and changes only its desired path. An unbound row receives a Rust-allocated NodeId before capture. A qualified same-session put receives its own new NodeId plus `copy_from = source`; two rows never share one NodeId.
3. CaptureRequest contains only NodeId, desired path candidate, kind, optional normalized `copy_from`, projection revision, and changedtick. It contains no CREATE, COPY, MOVE, or DELETE instruction. Rust derives removal only for an Existing NodeId that belonged to the initiating CommittedProjection and is absent from the accepted capture.
4. Rust stores validated Existing, PendingCreate, and PendingCopy desired state as ViewIntent. `apply_prepare` performs immutable initial probes and gives the edit base, current snapshot, ViewIntent, authority values, and probe results to Planner.
5. Planner compares source presence with all provenance-linked destinations. Source retained produces COPY/COPY_TREE; one destination with source removed produces MOVE; multiple destinations with source removed produce copies followed by source deletion; a removed source without a provenance destination produces DELETE/DELETE_TREE. Directly editing an Existing NodeId to a new path also produces MOVE without a provisional destination.
6. ValidPlan contains the immutable logical operations, execution DAG, planned identity dispositions, preview groups, affected paths, final-probe expectations, and authority. Planned identity dispositions are declarative plan data only. `apply_prepare` seals them into PreparedApply and does not rebind any NodeId.
7. Preview Cancel consumes or drops PreparedApply, performs no filesystem action, preserves dirty ViewIntent, and applies no identity transition. Validation, stale, Busy, and final-preflight errors likewise apply no identity transition and preserve dirty state because no filesystem execution has begun. Accepted execution cancellation is the separate clean-sync outcome described below.
8. After confirmed Execute passes authority and final preflight, Executor runs only the already planned nodes. Executor neither classifies operations nor assigns semantic identities; it reports COMPLETED, FAILED or CANCELLED, UNSTARTED, and every known successful internal effect.
9. Under Apply orchestration, Snapshot combines ValidPlan's planned identity dispositions with the execution report's known successful effects. Successful CREATE/COPY retains the pending destination NodeId. A fully successful logical MOVE places the source NodeId at the destination and retires any provisional PendingCopy destination. Cross-filesystem copy success followed by source-delete failure or accepted cancellation keeps the source NodeId and a distinct destination NodeId, using the PendingCopy destination's provisional NodeId when one exists and allocating one otherwise. A move with no known successful effect leaves the source identity in place and abandons the provisional destination. Unrepresented partial COPY_TREE discoveries receive new NodeIds.
10. Snapshot publishes one self-consistent immutable RootSnapshot. Success and empty plans clear ViewIntent; execution failure or accepted execution cancellation publishes known effects, abandons failed/cancelled and unstarted initiating intent, and also returns a clean initiating frame. Pre-execution aborts publish no clean replacement and preserve dirty intent.
11. For every successfully realized PendingCopy-derived logical MOVE, Snapshot returns exactly one authoritative `NodeIdTransition { from: provisional_destination, to: source }` in the same terminal outcome as the clean frame. Lua never derives this mapping. During synchronous terminal handling, Lua `buffer` coordinates `presentation` expansion remapping and `BufferProjectionEngine` binding/cursor reconciliation and consumes the transition before committing that clean frame, clearing `'modified'`, and establishing the undo baseline before `BufWriteCmd` returns.

The ownership rule is therefore: capture describes facts, Planner decides semantic operations and planned identity dispositions, Executor reports effects, Snapshot decides terminal identity, and Lua only projects the authoritative result.

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

Validation errors and conflicts produce a non-confirmable diagnostic report and location list. The initiating View remains dirty, and its edits and undo history are preserved. Diagnostics never produce a `PreparedApply` or enter confirmation.

A valid plan returns a sealed one-shot `PreparedApply`. Only a valid non-empty plan also returns one sanitized preview grouped by logical operation:

```text
CREATE
COPY
COPY_TREE
MOVE
DELETE
DELETE_TREE
REPLACE
```

Each row shows the source, destination, and kind. Tree operations are displayed as one logical row. Cached informational counts may be displayed, but FRED does not enumerate a tree merely to populate preview metadata. The preview contains no execution DAG, temporary rename names, authority stamps, probe facts, or mutable reference to the plan; modifying preview data cannot change what Executor later receives.

Confirmation is all-or-nothing. The user cannot deselect individual operations because doing so could invalidate dependencies or produce a namespace different from the edited buffer. Preview `Cancel` consumes or drops the prepared capability, never creates `ApplyBufferGuard`, never acquires the global gate, preserves dirty intent, and requires a new write to prepare another plan. A valid empty plan opens no preview, but Lua immediately establishes `ApplyBufferGuard` and consumes the same kind of capability through `apply_finish(... Execute ...)`, so empty and non-empty plans share the final changedtick and authority checks.

`PreparedApply` is not public API, is not a general ApplyAttempt state machine, and is not retained in a process-wide registry. Its only lifecycle is an internal one-shot `Fresh -> Consumed` transition. Duplicate finish, replay, foreign userdata, or finish after cancellation/drop is rejected. View destruction before Execute makes the weak initiating handle fail authority validation; destruction after mutation begins does not stop required fail-stop finalization. Synchronous clean handling that observes destruction or a generation mismatch performs no buffer resurrection or overwrite and always releases `ApplyBufferGuard`.

## 17. Apply Pipeline, Executor, And Failure Semantics

The deep Rust `apply` module owns the complete prepare-to-terminal data flow. Its two domain dependencies are the pure Planner and serial Executor. Lua remains the only owner of preview UI and FRED buffer synchronization; Lua never receives the execution DAG.

Before `apply_finish(... Execute ...)`, Lua creates `ApplyBufferGuard`, prevents reentrant writes and new ordinary edits, reads `observed_changedtick` and `write_epoch`, and retains the guard until synchronous terminal handling releases it. Execute atomically consumes the sealed capability before any lock attempt, so even Busy, stale, final-preflight, failure, or cancellation outcomes cannot reuse it.

For a non-empty plan, after Execute is requested:

1. the Apply task exposes only its generic internal progress/cancellation handle to Lua;
2. upgrade the weak initiating View/RootSession handles and validate current View ownership, View generation, root NodeId, projection revision, edit-base revision, planned-root revision, intent generation, and the Lua observed changedtick/write-epoch witness against the sealed authority;
3. if that validation is stale, return a pre-execution stale error, preserve the current/newer ViewIntent and dirty buffer, and release `ApplyBufferGuard`; an already-requested cancellation cannot bypass this check;
4. once current Execute is accepted, if cancellation is already requested before gate acquisition, skip mutation and follow the clean execution-cancellation outcome in Section 17.4;
5. otherwise acquire the process-global mutation lock, or return Busy without mutation, preserve dirty intent, and release `ApplyBufferGuard`;
6. register the immutable `ValidPlan` affected paths in the process-wide active mutation fence before enumerating RootSessions or issuing the first syscall, then create one common finalizer that owns every later exit, including cancellation and panic-to-error conversion;
7. cancel/fence existing initiating and overlapping RootSession scan generations, enter Apply, and require every RootSession created later to consult the active fence at scan start and publication;
8. repeat every required source, destination, existing-parent, planned-parent-state, nearest-existing-ancestor, kind, occupancy, and symlink-traversal probe under the lock; cancellation observed here follows Section 17.4 only because authority was already validated;
9. if every check succeeds, pass the immutable `ValidPlan` and cancellation token to Executor;
10. combine ValidPlan's planned identity dispositions with every known successful effect, derive the authoritative terminal identity outcome through Snapshot construction, and publish the resulting immutable RootSnapshot;
11. clear or abandon initiating ViewIntent according to the terminal outcome, build the clean initiating semantic frame and authoritative NodeIdTransition values when required, invalidate overlapping RootSessions, and queue best-effort refresh;
12. run the common finalizer: coalesce and schedule queued refresh demand, leave Apply, release scan-generation fences, remove the process-wide active mutation fence, and release the global mutation lock before the terminal outcome becomes observable;
13. Lua synchronously consumes the terminal frame and any authoritative NodeIdTransition values, or handles the abort, and releases `ApplyBufferGuard` before `BufWriteCmd` returns.

Authority or final-probe mismatch aborts before mutation, produces no clean projection, and preserves buffer text, ViewIntent, `'modified'`, and undo history. No overlapping scan candidate from an existing or future RootSession can publish across the active mutation fence. Snapshot application of a validated execution report's known successful effects is deterministic and total; it is not a second Apply transaction owner.

### 17.1 Serial Fail-Stop Execution And Cancellation

Executor accepts only a `ValidPlan` plus an internal cancellation token. `ValidPlan`, not Executor, owns affected paths. Executor does not plan, confirm, acquire the global lock, publish snapshots, invalidate overlaps, refresh Views, or operate on Lua buffers.

Executor rules:

1. run one node at a time in DAG order;
2. check cancellation before each node;
3. let COPY_TREE, cross-filesystem MOVE, and other long or compound operations check cancellation between entries or internal steps and at other safe checkpoints;
4. record each system API result and every known successful internal effect;
5. after a successful non-final node, proceed to the next cancellation checkpoint;
6. stop immediately on the first failed system call or cancellation observed before completion;
7. if the final execution node has already reported success when cancellation is first observed, classify the terminal result as Success rather than Cancellation;
8. do not run later nodes after failure or accepted cancellation;
9. do not automatically retry;
10. do not automatically undo known successful effects.

An uninterruptible blocking system call may finish before cancellation is observed. Destination creation and installation always use exclusive/no-replace behavior. A destination that appears after final preflight fails the relevant node instead of being overwritten.

Failure reports:

```text
COMPLETED
FAILED
UNSTARTED
```

Cancellation reports:

```text
COMPLETED
CANCELLED
UNSTARTED
```

At most one node is FAILED or CANCELLED. The execution report also carries known successful internal effects from that node. A cancelled long tree node may have partial side effects; best-effort refresh discovers effects that were not authoritatively reported. COPY_TREE failure or cancellation may leave a partial final destination, a cross-filesystem MOVE cancelled after copy success but before source deletion completes splits source/destination identity exactly like source-delete failure, and an interrupted rename cycle may leave an internal unique name. These are actual filesystem results, not reasons to add rollback or recovery state.

### 17.2 Success And Empty Plans

A valid empty plan is consumed through Execute, passes the final Lua witness and current View authority checks, acquires no mutation gate because it has no filesystem work, clears initiating intent, and returns a clean semantic frame for synchronous Lua commit.

When every non-empty execution node reports success, including cancellation first observed only after the final node succeeded, Apply:

- has Snapshot combine every known successful effect with Planner-owned identity dispositions, preserve the source NodeId for each successful logical MOVE, and retire a provisional destination only after that MOVE's success is known;
- publishes a new immutable RootSnapshot;
- clears initiating ViewIntent;
- returns a clean initiating semantic frame;
- invalidates every RootSession overlapping a Planner-owned affected path and queues best-effort refresh;
- reports write success;
- exits through the common finalization path.

Before `BufWriteCmd` returns write success, Lua synchronously validates and consumes the clean frame plus any authoritative NodeIdTransition values, renders with edit capture suppressed, remaps NodeId-keyed expansion when instructed, commits the projection, reconciles bindings and cursor, clears `'modified'`, establishes the new ordinary undo baseline, and releases `ApplyBufferGuard`. This ordering is required for ordinary Neovim 0.12.4 `:wq` to exit. Refresh and ancillary UI may be deferred; the clean commit may not be.

### 17.3 Execution Failure

On the first execution error, Apply:

1. stops the batch and records completed, failed, and unstarted nodes plus known successful internal effects;
2. has Snapshot apply only known successful effects and derive terminal identity, including a distinct destination NodeId when cross-filesystem copy succeeds but source deletion fails—retaining the PendingCopy provisional destination when available and allocating one otherwise—then publishes the resulting immutable RootSnapshot;
3. abandons failed and unstarted initiating intent;
4. returns a clean initiating semantic frame from the updated model;
5. invalidates overlapping RootSessions and queues best-effort refresh;
6. reports the system error and execution report;
7. exits through the common finalization path.

The failed node is not assumed wholly successful; refresh may reveal additional partial side effects left by that syscall. Before returning write failure, Lua synchronously commits the clean frame, clears `'modified'`, establishes a new undo baseline, and releases `ApplyBufferGuard`; therefore `:wq` remains open with a clean buffer. Other dirty Views retain their own intent and rebase against the next published snapshot. FRED does not offer automatic retry.

### 17.4 Execution Cancellation

Preview `Cancel` is defined in Section 16 and preserves dirty intent without creating `ApplyBufferGuard` or acquiring the gate. This section applies only after `apply_finish(... Execute ...)` has consumed the confirmed or empty `PreparedApply`. The Lua progress UI may request cancellation through the running task's internal cancellation handle; this creates no public apply/save entrypoint.

An already-requested cancellation cannot make an old capability abandon newer intent. Apply first validates current initiating View ownership and the sealed View/root/projection/edit-base/planned-root/intent/changedtick authority plus execute witness. Stale validation returns the pre-execution stale outcome, preserves current intent and `'modified'`, and releases `ApplyBufferGuard`. Only after current Execute is accepted may cancellation take the clean path before gate acquisition or the first syscall.

When accepted cancellation is observed before completion, Apply:

1. stops at the next safe checkpoint and runs no later node;
2. records completed, cancelled, and unstarted nodes plus known successful internal effects;
3. has Snapshot apply known successful effects, derive terminal identity, and publish the resulting immutable RootSnapshot;
4. for cross-filesystem MOVE, Snapshot treats copy success followed by cancellation observed before source deletion starts or before a cancellable deletion completes exactly like source-delete failure: the source retains its NodeId and the destination receives a distinct NodeId, retaining the PendingCopy provisional destination when available and allocating one otherwise; successful deletion followed only by later cancellation observation is Success; a partial cancelled COPY_TREE discovered later receives new NodeIds for unrepresented destination nodes;
5. abandons cancelled and unstarted initiating intent, even when no syscall had yet run;
6. returns a clean initiating semantic frame, invalidates overlapping RootSessions, and queues best-effort refresh to discover additional partial effects;
7. reports execution cancellation and exits through the common terminal path; when the gate was acquired, the post-lock finalizer leaves Apply, releases scan fences, removes the active mutation affected-path fence, and releases the lock.

No known successful effect is rolled back. A blocking syscall may complete before the cancellation request is observed. If cancellation is first observed only after the final execution node successfully completed, the result is Success and Section 17.2 applies. Otherwise, before returning write failure, Lua synchronously commits the latest available clean frame, clears `'modified'`, establishes a new undo baseline, and releases `ApplyBufferGuard`, so `:wq` remains open. Refresh failure is reported without restoring abandoned intent or creating an unknown, recovery, reconciliation-required, or apply-blocking state.

### 17.5 Process Kill Or Power Loss

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
- source path authoritatively disappeared from a Complete scope or direct probe: conflict;
- source was merely unobserved in a Partial or Failed scope: retain it as Unknown rather than conflict;
- source kind changed: conflict;
- desired destination appeared or became occupied: conflict;
- both disk and user changed the same namespace identity incompatibly: conflict;
- ordinary file content, permissions, size, or timestamps changed: not a FRED conflict;
- external rename: represent conservatively as old-path deletion plus new-path creation.

Watcher events trigger incremental refresh automatically. `instance:refresh()` forces reconciliation for any Open Instance with at least one valid display, even when another buffer is current. `:FredRefresh` is a current-buffer convenience command and accepts only the current visible Open FRED buffer. Hidden instances reject both forms and must be reopened first.

A dirty View is never silently overwritten by ordinary refresh. Conflicted entries remain visible regardless of hidden-file settings or collapse.

The initiating View after execution failure or accepted execution cancellation is the intentional exception: its failed/cancelled and unstarted intent is abandoned, known successful effects update the in-memory namespace model, and the View is synchronously clean-rendered inside `BufWriteCmd` while deferred best-effort refresh discovers any additional partial side effects.

## 19. Symlink Policy And Immediate Helper

FRED scanning never recursively follows a directory symlink. Every symlink remains a `kind = symlink` leaf Node even when normalized metadata reports `target_kind = directory`; it never owns a `children_by_parent` entry. Existing symlinks may be listed, opened, renamed, copied, or deleted as links. Inline expand/collapse on a symlink is rejected. An explicit action may resolve a directory-symlink target and open it in a new or child Instance backed by a separate exact-root RootSession, but same-instance `navigate()` never crosses into that session and target children are never mounted beneath the symlink NodeId in the current session.

This keeps RootSnapshot parent/child membership a tree rather than an alias graph, prevents recursive cycles and duplicate traversal, and avoids accidental ownership of a target outside the View root.

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
- the command enters the same private mutation runner used by buffer Apply, acquires the process-global mutation lock, and reports Busy if another mutation is active;
- the command builds one internal symlink execution node, executes immediately, and does not wait for `:write`;
- after acquiring the lock, it registers the destination in the process-wide active mutation affected-path fence before enumerating sessions or issuing the symlink syscall, then uses the same existing-generation invalidation and common finalizer as Apply;
- if the symlink system call succeeds or may have partially mutated the namespace, Apply invalidates overlapping RootSessions and queues best-effort refresh; the finalizer schedules that demand, removes the active fence, and releases the gate after success and every failure;
- operation and refresh errors are reported directly through Neovim without introducing an unknown, degraded, recovery, or apply-blocking state.

## 20. Internal Modules

Suggested structure:

```text
Cargo.toml
build.rs
src/
  lib.rs
  native_bridge.rs
  runtime.rs
  apply.rs
  root_session.rs
  snapshot.rs
  node.rs
  view.rs
  intent.rs
  semantic_frame.rs
  task.rs
  scanner.rs
  metadata.rs
  watcher.rs
  coverage.rs
  path.rs
  planner.rs
  conflict.rs
  executor.rs
  tree_ops.rs
  symlink.rs
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
  fred/buffer_projection.lua
  fred/projection_codec.lua
  fred/flat_projection.lua
  fred/projector.lua
  fred/presentation.lua
  fred/columns.lua
  fred/provenance.lua
  fred/layout.lua
plugin/
  fred.lua
ftplugin/
  fred.lua
```

The modules expose small internal interfaces with substantial behavior behind them:

- Rust `native_bridge` owns only exact-pair-tested native module registration, Lua/native argument and result conversion, opaque sealed userdata conversion, `AsyncHandle` wakeups, and main-thread event delivery; it does not own FRED buffer APIs or Apply policy;
- Rust `path` owns canonical native/session/View path conversion and validation;
- Rust `node` owns the NodeId type, RootSession-scoped allocator, scope validation, and opaque cross-language serialization; it exposes no lifecycle-encoded ID prefixes and owns no terminal lifecycle realization;
- Rust `snapshot` owns immutable normalized indexes, directory status, deterministic terminal identity resolution from planned dispositions plus known successful execution effects, and publication of self-consistent RootSnapshots;
- Rust `root_session` owns shared scan, watcher, coverage, metadata/cache state, exact-root weak registration, task state, and strong/weak lifetime boundaries;
- Rust `view` owns `root_node_id`, immutable edit base, semantic intent, CommittedProjection, and frame generations without owning a buffer number;
- Rust `intent` owns CaptureRequest validation and normalized Existing/PendingCreate/PendingCopy desired state;
- Rust `semantic_frame` applies ViewIntent to RootSnapshot and emits bounded projection-neutral ViewSemanticFrame events;
- Rust `metadata` owns platform metadata collection and normalized optional values requested by Lua columns and SortSpecs;
- Rust `planner` is pure and owns validation, final-state normalization, COPY/MOVE/DELETE classification, planned identity dispositions, logical preview data, execution DAG/order, `ValidPlan.affected_paths`, and final-probe expectations;
- Rust `executor` owns serial no-replace system calls, cooperative safe-checkpoint cancellation, fail-stop behavior, known-successful-effect reporting, and `COMPLETED / FAILED|CANCELLED / UNSTARTED` outcomes; it owns neither affected paths nor View refresh;
- Rust `apply` is the sole owner of the two-stage `apply_prepare`/`apply_finish` data flow, sealed one-shot `PreparedApply`, gate timing, authority and final-probe checks, process-wide active mutation affected-path fence registration/removal, generic execution-task cancellation coordination, terminal identity-resolution orchestration, overlap invalidation, deterministic publication orchestration, common finalization, and refresh scheduling;
- Rust `tree_ops` provides the deep cancellable COPY_TREE/DELETE_TREE implementations used only through Executor;
- Rust `task` is generic bounded worker/event/cancellation plumbing and defines no second Apply transaction or prepared-plan registry;
- Rust `runtime` owns RootSession/View registration, native handles, lookup facilities, and consultation of active mutation fences at RootSession scan start/publication without owning Apply state transitions;
- Lua `config` captures setup defaults, validates Lua-only callback/action/layout/column shapes, and resolves direct/child instance inheritance;
- Lua `instance` owns the public object, root/config identity, child RootSession lineage decisions, and native View handles;
- Lua `registry` owns InstanceId/name/buffer lookup and terminal removal;
- Lua `actions` and `keymaps` own function-valued actions, context construction, mode/lhs merge, and buffer-local installation;
- Lua `presentation` owns hidden-file visibility, SortSpec, MaterializationState, NodeId-keyed ExpansionState, presentation generations, the ViewSemanticFrame catalog mirror, and consumption of authoritative terminal NodeId transitions for expansion remapping;
- Lua `projector` owns hidden subtree filtering, sibling-local sorting, depth-first traversal, and production of the common ProjectedRow sequence;
- Lua `projection_codec` defines the internal sealed bidirectional codec contract; `flat_projection` is the only version-one implementation;
- Lua `buffer_projection` owns line rendering/capture, full-line NodeId range extmarks, projection commits, changedtick, authoritative terminal binding/cursor transition consumption, modified state, undo baselines, internal-sync suppression, and synchronous clean outcome commits inside `BufWriteCmd`;
- Lua `buffer` owns buffer creation and write-command routing, preview/confirmation UI coordination, `ApplyBufferGuard`, write epochs, reentrant-write rejection, coordination of the authoritative terminal transition across presentation and buffer projection, terminal guard release, and the internal execution progress/cancellation control;
- Lua `columns` owns built-in column definitions, layout, display-width alignment, highlighted chunks, and metadata requirement declarations;
- Lua `provenance` owns `TextYankPost`, register snapshots, `p`/`P` wrappers, inserted-range tracking, and normalized same-session `copy_from` capture;
- Lua `lifecycle` owns Neovim display reconciliation, Instance state transitions, timers, hidden-group LRU/capacity enforcement, and the idempotent terminal destroy path;
- Lua `layout` constructs native buffer, float, split, vsplit, and tab displays and returns ordinary window ownership records.

Standalone `apply_gate`, `model`, `preview`, and `report` ownership Modules are intentionally absent. Gate/finalization belongs to `apply`; known-effect transformation and immutable publication belong to `snapshot` under Apply orchestration; preview is Planner output; execution reporting is Executor output. Rust module interfaces remain internal. `src/lib.rs` exposes the native module through `nvim-oxi`; `lua/fred/init.lua` exposes the stable Instance facade. Version one exposes no backend, custom column, renderer, projection-mode, custom ProjectionCodec, PreparedApply, execution-task, or cancellation registration seam.


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
      group = "default",
    },
    groups = {
      default = {
        capacity = 10,
      },
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

Direct `fred.new(opts)` deep-merges setup defaults with explicit instance options, except function/list fields such as `on_attach`, which are normalized and concatenated deliberately, and SortSpec. Every explicitly supplied SortSpec at setup, direct `new()`, or child `new()` is validated independently as exactly `{ by, direction, case_sensitive }` and replaces the previous SortSpec wholesale. `instance:new(opts)` starts from the parent's resolved immutable options, replaces its initial hidden-file visibility and SortSpec with snapshots of the parent's current runtime values, then applies explicit child overrides; an explicit child SortSpec replaces that snapshot rather than merging with it. It does not replay setup callbacks. When the child root is a directory inside the parent RootSession, the child also snapshots the parent's current NodeId-keyed expansion entries inside that subtree as runtime state; expansion is not added to immutable `config()`, and later parent/child expansion changes are independent. The resulting configured values are part of the child's immutable `config()` snapshot, while later setters affect only runtime getters and presentation state.

`cleanup.instance` is inherited through resolved direct/child options and may be overridden per construction. `cleanup.groups` is setup-only, process-wide, and immutable after setup; supplying it to direct or child `new()` options is invalid. Every resolved `cleanup.instance.group` must name a setup-defined group, and unknown groups fail during `new()` validation.

`root` belongs to `new()` options and is not an `open()` option. Direct `fred.new()` defaults it to the current working directory; `parent:new()` defaults it to the parent's current root. `layout` belongs to resolved instance options and may be overridden per `open()`/`toggle()` call. Values are `buffer`, `float`, `split`, `vsplit`, or `tab` in version one. `open({ new_display = true })` is a per-call display option, defaults to `false`, and is never stored in resolved configuration.

`keymaps` is keyed by Neovim mode and lhs. Values are functions or `false`; strings are invalid. Functions replace inherited mappings and `false` removes them. `actions.select()` receives entry-kind action functions and returns the final mapping callback.

`on_attach` accepts one function or a function list. Setup callbacks run before direct instance callbacks; inherited child lists run before explicit child callbacks. The first callback error rolls back and destroys the instance and is rethrown.

`columns` defaults to `{"icon"}`. The path remains the only editable field. `view.hidden_files_visible` and `is_hidden_file` are the complete hidden-file configuration; there is no generic filter or second always-hidden predicate. Reveal may persistently enable hidden-file visibility and add exact ignore overrides without creating pinned rows.

`sort` is the complete three-field SortSpec described in Section 13. All fields are required, unknown fields are errors, and only the built-in single keys are supported. Arbitrary comparators, multi-key sorting, directory-first options, and configurable missing-value placement are not public version-one features.

Watcher establishment, plan confirmation, no-follow directory-symlink behavior, and the internal FlatProjectionCodec remain automatic and non-configurable in version one. There is no `view_mode`, tree toggle, renderer selector, or custom ProjectionCodec option.

`cleanup.instance.delay_ms` is a non-negative integer; `0` disables only the hidden-delay timer. `cleanup.instance.group` is a non-empty setup-defined group name. Each `cleanup.groups.<name>.capacity` is a non-negative integer; `0` means that group retains no Hidden instances. Timer expiry and group overflow are the only automatic destroy triggers and both use the same idempotent terminal path.

Setup validates instance names/options, layout values, keymap mode/lhs/function shapes, action selectors, attach callback lists, `hidden_files_visible`, the `is_hidden_file` callback, complete SortSpecs, columns, time formats, limits, cleanup groups/capacities, and cleanup instance values. Invalid configuration produces explicit errors rather than silent fallback.

## 22. Error Handling

Errors fall into six classes.

### 22.1 Scan Or Watch Error

The buffer remains readable. Failed or limited browse scopes are marked partial or degraded. Manual refresh and scope reduction remain available.

Scan or watcher failure does not globally disable the write-driven apply pipeline. A plan is blocked only when its own required source, destination, or parent state cannot be checked.

An individual metadata read failure is reported without invalidating the namespace snapshot, disabling a later valid write, or deleting filesystem data. Cancelled provisional state is released through ownership rather than a fallible cleanup subsystem.

### 22.2 Validation, Stale, Busy, Or Final-Preflight Error

No filesystem action runs. Validation/final-preflight errors, stale authority, and process-global lock contention are pre-execution outcomes: the buffer remains dirty, edits and undo history are preserved, `ApplyBufferGuard` is released, and errors are attached to affected rows or reported directly. Busy explicitly means another FRED mutation holds the process-global gate; it does not queue or retry the write.

A stale changedtick, View/root/projection generation, intent generation, `edit_base_revision`, or `planned_root_revision` requires a fresh write/prepare. Authority is checked before an already-requested execution cancellation may abandon intent, so a stale old capability/task preserves newer intent rather than clean-syncing it. If final-preflight failure occurs after lock/fence acquisition, the common finalizer leaves Apply, coalesces and schedules queued refresh demand, releases scan-generation fences, removes the process-wide active mutation affected-path fence, and releases the global mutation lock.

### 22.3 Conflict

No filesystem action runs. Conflicted entries remain visible in the non-confirmable diagnostic report and require refresh, rebase, or explicit user edits.

Typical conflicts are missing sources, changed source kind, occupied destinations, invalid parents, and incompatible namespace changes. Conflicts never open the confirmation preview.

### 22.4 Execution Failure Or Cancellation

On execution failure, Executor stops immediately, reports `COMPLETED / FAILED / UNSTARTED`, and runs no later operation. On accepted execution cancellation observed before completion, it stops at the next safe checkpoint, reports `COMPLETED / CANCELLED / UNSTARTED`, and runs no later operation. A cancellation first observed after the final execution node succeeded is Success. Known successful effects, including successful internal steps of the failed/cancelled node, update the immutable namespace model deterministically; failed/cancelled and unstarted initiating intent is abandoned; Apply returns a clean semantic frame; filesystem refresh remains best effort.

Neither outcome rolls back known successful work. Cross-filesystem copy success followed by source-delete failure, or cancellation observed after copy success and before deletion starts or before a cancellable deletion completes, leaves the source NodeId at the source and a distinct NodeId at the destination; successful deletion followed only by later cancellation observation is Success. Partial failed/cancelled COPY_TREE discoveries receive new destination NodeIds. Before `BufWriteCmd` returns write failure, Lua synchronously commits the clean frame, clears `'modified'`, establishes the undo baseline, and releases `ApplyBufferGuard`, so `:wq` remains open with a clean buffer. Refresh errors report through Neovim without blocking a later write or creating a recovery, unknown, reconciliation-required, or apply-blocking state.

### 22.5 Instance, Layout, Or Attach Error

Invalid names, roots, layouts, action contexts, keymaps, or lifecycle transitions fail explicitly. An `on_attach` error is fail-fast: FRED stops later callbacks, tears down the partially created windows/buffer/View/timers/RootSession references, marks the instance Destroyed, and rethrows the original error.

A layout-open failure rolls back only resources created by that open attempt. If the instance had no prior buffer/View and buffer initialization cannot complete, it is destroyed; an already valid instance remains usable when an additional window fails to open. Unknown cleanup groups fail during `new()` before buffer/View creation. Writes fail explicitly unless the registered Instance buffer is current, Open, displayed, and targeted at its own URI. `instance:refresh()` requires Open/displayed but not current; `:FredRefresh` additionally requires the registered buffer to be current. Hidden instances reject both refresh forms and must be reopened.

### 22.6 Presentation Error

Immediate public-method validation errors and a different SortSpec on a dirty View throw directly to the caller and leave committed presentation state unchanged; identical normalized values are no-ops. During scheduled frame processing, an `is_hidden_file` exception, non-boolean return, codec error, stale frame token, unknown/duplicate NodeId, or ProjectionCommit rejection propagates as the direct Neovim Lua/native error. FRED does not add a projection lease, renderer rollback protocol, recovery state, promise-like result, or automatic retry. Classification or codec failure before rendering leaves the previous committed projection intact; render or commit failures are fail-fast and require an ordinary later refresh or reopen if the user wants to retry.

## 23. Testing Strategy

### 23.1 Rust Unit Tests

- canonical native/session/View path encoding, decoding, rebasing, and root-escape rejection;
- malformed escapes, encoded separators, literal and encoded `.`/`..`, absolute paths, Linux byte paths, and Windows native names;
- RootSnapshot builder invariants for `nodes_by_id`, `children_by_parent`, path index, one-parent membership, directory-only parents, immutable publication, and Complete-versus-Partial/Failed authority merging;
- one opaque RootSession-scoped NodeId type across Existing, PendingCreate, PendingCopy, dirty move, successful create/copy, and conflict state; namespaced Lua-string round-trip without path-text or lifecycle-prefix ambiguity; capture-time non-rebinding; and provisional-destination retirement only after terminal effects prove MOVE success;
- same-path lineage reuse, FRED rename/MOVE identity preservation, external rename as delete-plus-create, COPY destination allocation, and hard-link separation;
- same-filesystem MOVE, fully successful cross-filesystem MOVE, copy-success/delete-failure and copy-success/delete-cancellation two-NodeId results, and failed/cancelled partial COPY_TREE destination discovery with new NodeIds;
- session root NodeId, movable View `root_node_id`, View-relative/session-relative rebasing, descendant navigation without RootSession replacement, dirty View-relative intent following a same-session root-node MOVE, and root-node Missing only after authoritative deletion;
- exact-root RootSession reuse, parent/child lineage sharing, independent overlapping direct roots, process-wide affected-path fence consultation by existing and newly created sessions at scan start/publication, and post-terminal overlap invalidation;
- directory symlink leaf invariants, target-kind metadata, no child membership, no recursive traversal, and explicit target-root separation;
- ViewIntent capture validation for stale projection revision, changedtick, unknown/duplicate NodeId, existing-kind change, pending-node scope, duplicate destination, and same-session provenance;
- CommittedProjection membership as the sole projection-deletion baseline;
- RootSnapshot plus ViewIntent production of clean, dirty, pending, moved, conflict, validation, deleted-directory suppression, and evacuated-descendant ViewSemanticFrame nodes;
- sealed ViewFrameToken currency and stale/mixed frame-event rejection;
- immutable initial and final namespace-probe specifications for sources, destinations, existing parents, planned-parent chains, nearest ancestors, and symlink traversal;
- pure Planner normalization for create, copy, copy-tree, move, delete, delete-tree, replacement, multiple provenance destinations, planned identity dispositions, rename cycles, conditional ordering, logical preview, affected paths, and final-probe expectations;
- sealed `PreparedApply` one-shot consumption, preview immutability, duplicate/replay rejection, stale older preparation, authority-before-cancellation ordering, explicit cancel/drop memory release, and absence of a prepared-plan registry;
- common post-lock finalization, process-wide active mutation affected-path fencing, serial fail-stop execution, safe-checkpoint cancellation, cancellation-after-final-success classification, deterministic known-effect publication, and no later node after the first failure or accepted cancellation;
- metadata-generation/source-revision checks and NodeId-plus-observed-path binding;
- RootSession Weak registry behavior, View/task strong ownership, cancellation, and natural final release.

### 23.2 Lua Unit Tests

- complete SortSpec normalization, wholesale replacement, equality no-op, missing-values-last, and fixed tie-breakers;
- hidden-file callback inputs, boolean enforcement, hidden-directory subtree propagation, and cache invalidation;
- sibling-local sorting followed by depth-first traversal for every sort key and direction, with every materialized directory subtree contiguous;
- identical ProjectedRow NodeId order feeding FlatProjectionCodec and a test-only tree-shaped codec fixture;
- MaterializationState baseline depth, NodeId-keyed expansions, recursive depth, explicit includes, and coverage-request generation;
- same-instance root-node change retaining expansion, parent/child subtree expansion snapshot, later independence, directory MOVE retention, and COPY non-inheritance;
- catalog mirror assembly from bounded frame events and stale/mixed token rejection;
- FlatProjectionCodec canonical line encode/decode and no public projection-mode or codec registry;
- BufferProjectionEngine full-line NodeId range extmarks, exact gravity, invalidation, undo restoration, internal-sync suppression, duplicate binding rejection, and unique-path rebind only;
- unbound-row NodeId allocation before capture; absence of capture-time or preview-time rebinding; and atomic consumption of authoritative terminal NodeIdTransition values for successful PendingCopy-derived MOVE, including bindings, cursor, and expansion remap;
- `TextYankPost`, register content/type qualification, `p`/`P` native behavior, same-session parent/child provenance, independent-session rejection, and PendingCopy capture;
- Lua columns, virtual-text alignment, display-cell widths, placeholders, metadata requirements, and no editable-text pollution;
- `ApplyBufferGuard` reentrant-write/edit prevention, execute-witness changedtick/write epochs, synchronous clean-frame render/ProjectionCommit/modified-state/undo-baseline/cursor handling inside `BufWriteCmd`, guard release on every success/abort/error/destruction path, and destroyed/generation-mismatched no-resurrection behavior;
- Created/Open/Hidden/Destroyed lifecycle, cleanup timers/group LRUs, configuration inheritance, callbacks, keymaps, and layouts.

### 23.3 Integration Tests

- exact-pair native build/load, bounded worker events, main-thread delivery, and absence of Rust FRED-buffer calls;
- recursive and selectively materialized scanning into immutable normalized RootSnapshots;
- one snapshot publication only after all requested scopes are terminal;
- Complete scopes authoritatively replacing child membership while Partial/Failed scopes retain prior unobserved nodes as RetainedUnverified;
- dirty rebase, conflict detection, projection, and descendant-root status treating RetainedUnverified nodes as Unknown rather than deleted;
- RootSnapshot-to-ViewSemanticFrame event delivery, Lua catalog assembly, common projection, FlatProjectionCodec rendering, and ProjectionCommit;
- arbitrary frame/catalog omission from hidden, ignore, depth, collapse, cancellation, failure, limits, or watcher gaps never generating deletion;
- removal of a NodeId that belonged to CommittedProjection generating deletion intent;
- different exact-root Views sharing snapshots; parent/child and navigate lineage sharing one RootSession and NodeIds; independent nested direct roots remaining separate;
- same-instance navigation preserving buffer/Instance/View/RootSession/NodeIds and rebasing View-relative paths;
- one dirty View root moved or deleted by another FRED View or external refresh: MOVE follows the stable NodeId and re-bases desired View-relative intent, while authoritative deletion preserves the buffer/intent under a non-confirmable root-missing diagnostic and blocks planning/navigation but permits refresh/destroy;
- Partial/Failed non-observation of that root retaining Present/unverified status rather than producing a Missing-root diagnostic;
- child-instance creation sharing the parent session, snapshotting subtree expansion, transferring the window, and preserving the parent dirty buffer;
- directory symlink display as a leaf, inline expansion rejection, link operations affecting the link, and explicit target opening in a separate RootSession;
- watcher-driven clean refresh and dirty semantic rebase across same and overlapping sessions;
- ordinary CREATE, MOVE, DELETE, COPY, REPLACE, COPY_TREE, DELETE_TREE, descendant evacuation, and directory-row suppression;
- normal yank/put provenance, `gp`, deterministic `_N` names, multiple copy destinations, and cross-session rejection;
- `apply_prepare` capture/probe/Planner flow, Planner-owned operation classification and identity dispositions, sanitized preview, sealed capability consumption, old-token staleness, authority validation before pre-gate cancellation, and repeated final preflight for missing sources, wrong kinds, occupied destinations, planned parents, non-directory ancestors, and symlink traversal;
- destination races using no-replace behavior, Busy lock contention, a process-wide affected-path fence registered before session enumeration/first syscall, existing-generation invalidation, and a newly created overlapping RootSession blocked at scan start/publication;
- same- and cross-filesystem file/directory MOVE, including successful provisional-to-source terminal transition, source-delete failure and copy-complete/delete-cancellation identity splitting, PendingCopy provisional retention for split destinations, and allocation for direct Existing-path split destinations;
- a completed PendingCopy-derived MOVE followed by a later-node failure or cancellation: publish the source NodeId at the destination, retire the provisional ID, emit exactly one authoritative transition, synchronously remap binding/cursor/expansion, and then return write failure without rolling back that completed MOVE;
- COPY_TREE failure and execution cancellation before gate, before first syscall, and at safe checkpoints; cancellation after final-node success classified as Success; completed/failed-or-cancelled/unstarted reporting; synchronous clean initiating commit; and best-effort discovery of partial effects;
- metadata-only updates, Lua column refresh, clean sibling reorder, dirty real-row freeze, and reapplication on becoming clean;
- reveal through ignored/uncovered/hidden ancestors without changing root;
- successful and failed `:FredLink` global-lock release and overlapping refresh.

### 23.4 Headless Neovim Tests

- real `filetype=fred`, `buftype=acwrite` buffers rendered exclusively by Lua;
- no `/NNN` identity prefix or `../` row and only complete View-relative paths in version-one editable lines;
- buffer-local `BufWriteCmd`, `FileWriteCmd`, and `FileAppendCmd` validation for current/Open/displayed/exact-URI/full-buffer rules;
- synchronous Lua semantic capture followed by `apply_prepare`, immutable preview/confirmation, and synchronous clean frame render/ProjectionCommit with `'modified' = false` before `BufWriteCmd` returns for empty, success, execution failure, and accepted execution cancellation;
- invalid/conflicted preparation, dirty-preserving preview `Cancel`, empty-plan execute witness, sealed-token replay rejection, stale old task plus already-requested cancellation, Busy lock contention, final-preflight failure, Execute accepted plus cancellation before gate/first syscall, and execution progress cancellation;
- real Neovim 0.12.4 `:wq`: successful/empty writes synchronously clear modified state and exit, while execution failure/cancellation synchronously clear modified state then return write failure and remain open; `ApplyBufferGuard` late-edit protection and release on every path; immediate Hidden cleanup; destroyed/generation-mismatched no-resurrection; cursor and undo-baseline behavior;
- actual range-extmark behavior under edit, delete, move, yank, put, undo, redo, internal render, sort, refresh, and cursor reconciliation;
- large-frame bounded Lua classification, sibling sorting, DFS projection, rendering, and event-loop progress;
- columns and metadata refresh without changing path text or filesystem intent;
- native layouts, multi-window displays, FileType/on_attach ordering, action contexts, registry lookup, cleanup races, and hidden-buffer reopening;
- absence of public tree mode, projection toggle, renderer selector, custom codec registration, or compatibility alias.

### 23.5 Property Tests

Randomly generated snapshots, semantic frames, captures, and intents must preserve:

- normalized Snapshot parent/child/path-index consistency;
- Complete child membership authorizing removal while Partial/Failed child membership preserves old unobserved nodes as Unknown;
- one NodeId per semantic node and no duplicate child membership;
- no child membership under symlinks;
- View-relative/session-relative round-trip under arbitrary descendant root nodes;
- dirty View-relative intent follows a moved root NodeId; an authoritatively removed root cannot silently bind to a replacement node; Partial/Failed non-observation cannot mark it Missing;
- sibling-only sorting and contiguous DFS subtrees;
- accepted ProjectionCommit NodeIds are a unique subset of one current frame;
- projection absence never authorizes deletion;
- user removal affects only NodeIds from the initiating CommittedProjection;
- hidden, sort, depth, expansion, collapse, renderer choice, layout, and lifecycle do not change semantic intent by themselves;
- no duplicate final destinations and no operation outside the View root;
- provenance copies precede removed-source deletion;
- directory move/copy never targets itself or a descendant;
- every required parent exists or is created earlier by the same plan;
- no silent overwrite under injected races;
- no execution node runs after the first failure or accepted cancellation, while cancellation observed only after final-node success preserves Success;
- successful logical MOVE leaves one NodeId at the destination, while copy-success/delete-failure or copy-success/delete-cancellation leaves distinct source and destination NodeIds and partial failed/cancelled COPY_TREE discovery creates new destination NodeIds;
- a scan generation publishes at most one immutable snapshot after all scopes terminate; every existing or future RootSession consults the process-wide active mutation affected-path fence at scan start and publication and cannot publish an overlap across it;
- every post-lock outcome coalesces and schedules queued refresh demand, leaves Apply, releases scan-generation fences, removes the process-wide active mutation affected-path fence, and releases the process-global mutation gate;
- View/task references and sealed capability ownership, not registries or expansion state, determine lifetime; dropped or consumed PreparedApply values release plan memory.

## 24. Performance Requirements

- Scanning and rendering must not block Neovim beyond one bounded main-thread batch.
- Coverage, depth, NodeId-keyed expansion, ignore, or explicit refresh changes cancel obsolete scan generations; same-session `root_node_id` changes invalidate obsolete View frames without replacing the RootSession.
- Late events from cancelled generations are ignored.
- Large COPY_TREE and cross-filesystem MOVE operations run on worker threads through deep operation modules and check execution cancellation between entries/internal steps and at other safe checkpoints; an uninterruptible blocking syscall may finish before cancellation is observed, and cancellation first observed only after final-node success remains Success.
- Scan progress may be displayed while candidate nodes remain internal; editable node projection begins only from a ViewSemanticFrame built after atomic RootSnapshot publication.
- Lua consumes bounded ViewSemanticFrame batches and performs cancellable hidden classification, sibling sorting, and depth-first traversal in bounded main-thread slices that yield observable event-loop progress; it may yield between slices, but only a completed current frame is rendered and committed.
- ProjectionCommit requests are sealed-frame-bound and reject stale tokens or unknown/duplicate NodeIds; a stale presentation generation cancels before Lua renders or commits a newer projection.
- Re-rendering preserves cursor by NodeId when possible.
- Exact-root Views and explicit parent/child/navigate lineage Views share RootSession snapshot and watcher data; independent overlapping direct roots remain separate, consult the process-wide active mutation affected-path fence at scan start/publication, and use post-terminal overlap invalidation.
- Per-generation scan work, Rust semantic-frame construction, each Lua catalog mirror, and common projection scale linearly with that View's covered nodes, while a still-live RootSession cache may scale with scopes visited across many generations rather than only current View coverage.
- Obsolete snapshot versions are released when no View or active task strongly references them; the current snapshot drops with its RootSession.
- Stat metadata is fetched only for columns and Lua SortSpecs that require it; metadata-only refresh must not trigger unnecessary namespace planning.
- Metadata sorting may reorder only clean Views; dirty Views freeze sibling-local real-line orders while virtual columns continue to refresh, then reapply the already-current sort and DFS traversal when clean.
- RootSession provisional task-owned state and dropped/consumed PreparedApply plan data are released promptly through ownership; `max_entries`/`max_directories` and bounded channels/batches constrain each requested generation, not cumulative cache retained by a still-live RootSession.
- Instance window bookkeeping is proportional to actual displays and does not duplicate scan/snapshot data.
- Lua Instance cleanup uses at most one timer per Hidden Instance plus O(1) group-LRU membership updates; redisplay cancels/removes them without polling keyboard activity, and capacity overflow evicts oldest Hidden instances.
- native Lua layout work remains outside scan event draining and the Rust filesystem critical path.
- Version one uses no Tokio runtime and no speculative thread pool.

## 25. Implementation Sequence

All Section 26 acceptance criteria define the version-one contract. Development proceeds through runnable internal milestones:

```text
M0  exact-pair native bridge, Lua Instance lifecycle, BufferProjectionEngine, and empty View
M1  NodeId RootSnapshot, bounded ViewSemanticFrame delivery, common projector, FlatProjectionCodec, and metadata columns
M2  shared RootSession lineage, coverage/watchers, NodeId expansion, navigation, child Views, dirty edit bases, and rebase
M3  semantic capture, NodeId extmarks, same-session provenance, ViewIntent, initial probes, pure Planner, sealed PreparedApply, and preview
M4  two-stage write integration, ApplyBufferGuard, global mutation runner, cancellable Executor, deterministic publication, and terminal reporting
M5  cross-filesystem identity splitting, rename-cycle hardening, symlink-root behavior, and the complete Linux/Windows matrix
1.0 every Section 26 acceptance criterion passes
```

The implementation sequence is:

1. Establish the Cargo workspace, exact Neovim 0.12.4/`nvim-oxi` candidate, native module/`AsyncHandle` bridge compatibility spike, Linux/Windows build paths, local build helper, native loader, and headless Neovim harness. Rust buffer access is not part of the native bridge.
2. Implement Lua configuration capture, direct/child inheritance, immutable resolved options, runtime presentation getters/setters, registry/name lookup, function-valued actions/keymaps, `vim.b.fred`, FileType ordering, attach callbacks, native layouts, and Instance cleanup.
3. Implement Lua `BufferProjectionEngine`, internal sealed ProjectionCodec contract, FlatProjectionCodec, line rendering/capture, NodeId binding specs, virtual decorations, changedtick, cursor, modified state, undo baselines, and internal-sync suppression against a temporary mock semantic frame.
4. Implement Rust RootSessionId/ViewId/NodeId allocation, opaque namespaced NodeId serialization for Lua, canonical path algebra, session root NodeId, normalized immutable RootSnapshot, `nodes_by_id`, `children_by_parent`, path indexes, directory status, and exact-root Weak registry.
5. Implement worker channels, bounded scan/metadata events, cancellation, generation checks, `AsyncHandle` re-wake, and one atomic RootSnapshot publication after all requested scopes are terminal.
6. Implement Rust View with movable `root_node_id`, semantic ViewIntent, CommittedProjection, ViewSemanticFrame construction, sealed frame tokens, bounded frame events, and Lua catalog-mirror assembly.
7. Implement Lua common projector: hidden-directory propagation, sibling-local SortSpec ordering, depth-first traversal, baseline materialization, NodeId-keyed ExpansionState, explicit includes, coverage-request generation, and Lua columns/metadata requirements.
8. Implement exact-root and explicit parent/child/navigate RootSession sharing rules, same-session path rebasing, child subtree expansion snapshots, independent overlapping sessions, reveal, directory-symlink leaf rejection, watchers, and overlap invalidation.
9. Add immutable dirty edit bases, semantic three-way rebase, frame-level conflict/validation nodes, clean/dirty sibling-order freeze behavior, and non-confirmable diagnostics.
10. Add full-line Lua range extmarks, Rust NodeId allocation for unbound rows, `:FredNew`, FlatProjectionCodec semantic capture, Lua `TextYankPost`, `p`/`P` wrappers, same-session provenance, `gp`, and directory-row descendant suppression through ViewIntent.
11. Implement immutable initial/final namespace-probe specifications and the pure Planner with both revisions, final-state COPY/MOVE/DELETE classification, planned identity dispositions, logical preview data, final-probe expectations, COPY_TREE/DELETE_TREE, nested normalization, descendant evacuation, replacement validation, collision checks, and logical NodeId preservation.
12. Implement the deep Rust `apply` module and internal `apply_prepare`: CaptureRequest validation, ViewIntent update, initial probes, Planner invocation, sanitized optional preview, sealed one-shot PreparedApply userdata, no prepared-plan registry, and explicit cancel/drop release.
13. Build synchronous current-buffer `acwrite`/`BufWriteCmd` integration with `apply_finish`, add `FileWriteCmd`/`FileAppendCmd` rejection handlers, Lua `ApplyBufferGuard`/write epochs, confirmation, empty-plan execute witness, duplicate-token rejection, authority-before-cancellation stale checks, authoritative terminal NodeIdTransition consumption across presentation and buffer projection, synchronous clean-frame commit and modified-state transition, real Neovim 0.12.4 `:wq` behavior, and terminal guard release.
14. Implement the process-global mutation runner, `ValidPlan.affected_paths`, process-wide active mutation fence registration before session enumeration/first syscall, existing/future RootSession scan-start/publication checks, common finalization/removal, serial fail-stop Executor, safe-checkpoint execution cancellation, final-node-success classification, deep tree operations, no-replace destinations, rename cycles, deterministic Snapshot terminal identity resolution from planned dispositions plus known effects, Apply-owned overlap refresh, cross-filesystem success identity preservation, and copy-success/delete-failure-or-cancellation identity splitting.
15. Add immediate `:FredLink` through the common mutation runner, buffer-directory resolution, directory-symlink target opening as a separate RootSession, and full failure/cancellation-injection coverage.
16. Run complete configuration, lifecycle, layout, buffer, projection, navigation, mutation, multi-View/session, watcher, metadata, Linux/Windows, large-tree, prepared-capability, and execution-cancellation suites before declaring version one stable.

Each phase must preserve read-only usability before mutation support is enabled. Version one does not implement or expose a tree codec, projection toggle, renderer selector, or custom codec registration.

## 26. Acceptance Criteria

Version one is ready when:

1. FRED builds locally as a required Rust `cdylib` and loads on Linux and Windows for every exact allowlisted Neovim 0.12/`nvim-oxi` pair; native module registration, conversion, `AsyncHandle`, and event-delivery compatibility pass, while missing native libraries produce actionable hard errors.
2. Rust performs no direct FRED View buffer, extmark, virtual-text, cursor, changedtick, modified-state, or undo operation.
3. The Lua facade exposes the approved Instance, presentation, refresh, registry, and function-valued action APIs with no public apply/save method, tree mode, projection toggle, renderer selector, placeholder `view_mode`, custom codec registration, or compatibility alias.
4. `:Fred` opens a responsive version-one FlatProjectionCodec buffer with `filetype=fred`, `buftype=acwrite`, complete View-relative paths, no `/NNN` identity prefix, and no `../` row.
5. One opaque RootSession-scoped NodeId type represents existing files/directories/symlinks, PendingCreate nodes, PendingCopy nodes, dirty moved nodes, and conflicted nodes; Lua receives an unparsed namespaced string that cannot collide with editable filenames and does not encode lifecycle; successful CREATE/COPY keeps its pending NodeId, while a unique removed-source PendingCopy destination is rebound to the source NodeId and retires its provisional NodeId only after terminal effects prove the logical MOVE succeeded.
6. Every published RootSnapshot has one root NodeId and self-consistent `nodes_by_id`, `children_by_parent`, path index, one-parent membership, directory status, and symlink-leaf invariants.
7. A scan generation publishes exactly one immutable RootSnapshot only after every requested scope is terminal; Complete directory membership may remove absent prior children, while Partial/Failed membership remains non-authoritative, carries prior unobserved nodes as RetainedUnverified, and cannot imply conflict, deletion, or Missing root.
8. Exact-root direct Views share RootSession state; same-instance descendant navigation and explicit parent/child lineage share the existing RootSession and NodeIds; independent overlapping direct roots are not automatically merged.
9. Same-instance navigation changes only View `root_node_id`, re-bases View-relative paths, preserves NodeIds and NodeId-keyed expansion, keeps the buffer/Instance/View/RootSession, and requires a clean View; a same-session MOVE of a dirty View's root NodeId follows its new path and carries desired View-relative intent with it, while only authoritative root deletion produces the non-confirmable Missing-root diagnostic and blocks planning/navigation without silent rebinding.
10. A child Instance inside the parent session shares that RootSession, uses the selected directory NodeId as root, snapshots only parent expansion entries in that subtree, evolves independently afterward, and leaves the parent dirty buffer intact.
11. A directory symlink remains a symlink leaf with no children, cannot be expanded inline, and affects only the link under rename/copy/delete; explicit target opening uses a separate exact-root RootSession.
12. Rust applies RootSnapshot plus ViewIntent into a projection-neutral ViewSemanticFrame containing normalized parent/child membership for clean, dirty, pending, moved, conflict, validation, suppressed, and evacuated state.
13. ViewSemanticFrame events are bounded and token-bound; Lua rejects stale or mixed frame batches.
14. Lua assembles a catalog mirror, applies hidden-directory filtering, sorts only direct sibling groups, performs depth-first traversal, and keeps every materialized directory subtree contiguous.
15. Flat and a test-only tree-shaped codec fixture consume the same ProjectedRow NodeId order; only line encoding/decoding differs.
16. Version one implements only the internal sealed FlatProjectionCodec and exposes no public renderer extension seam.
17. Lua BufferProjectionEngine owns full-line NodeId range extmarks, rendering/capture, decorations, cursor reconciliation, changedtick, modified state, undo baselines, and internal-sync suppression.
18. Unbound rows receive Rust-allocated NodeIds before semantic capture; duplicate, unknown, foreign-session, or stale bindings are rejected.
19. Lua semantic capture submits only NodeId, desired View-relative path candidate, kind, optional same-session `copy_from`, projection revision, and changedtick; Rust repeats path and semantic validation and never parses buffer text.
20. Rust records CommittedProjection from a current ViewFrameToken, unique ordered NodeIds, and changedtick; only removal of a bound Existing NodeId from that projection may generate deletion intent.
21. Ignore, hidden-file display, sibling sort, depth, expansion, collapse, renderer choice, scan cancellation/failure/limits, reveal, layout, and watcher gaps never imply deletion.
22. `TextYankPost` and Lua `p`/`P` wrappers preserve native behavior, qualify register content/type, attach provenance only within one RootSession, and treat unqualified or foreign insertions as PendingCreate.
23. Parent/child Views sharing one RootSession may exchange provenance; independent overlapping sessions may not.
24. Source retained produces COPY/COPY_TREE; one destination with removed source makes Planner produce MOVE plus a planned identity disposition without mutating capture or Lua state; only a successfully realized terminal MOVE rebinds the provisional destination to the source NodeId; multiple destinations with removed source retain destination NodeIds, produce copies, then delete the source.
25. `gp` copies selected top-level entries into a directory and generates deterministic `_N` destinations.
26. Dirty Views retain immutable edit bases; external namespace changes refresh clean Views and semantically rebase dirty Views without overwriting intent.
27. Metadata columns are Lua virtual text, never editable syntax, handle unavailable values with placeholders, and request only required Rust metadata fields.
28. Metadata batches independently reject stale metadata generation, source RootRevision, NodeId, or observed path; metadata changes never create namespace conflict or deletion intent.
29. Metadata-backed sorting may reorder only clean sibling groups; dirty Views freeze real sibling orders while virtual columns continue to refresh.
30. Watcher establishment is automatic; failure is visible but does not globally block unrelated valid plans.
31. RootSession Weak registry entries do not retain sessions; Views/tasks do, last-View detach cancels work with no consumers, and final strong-reference release drops snapshots, caches, coverage, metadata, and watchers without a Rust LRU or sweeper.
32. Ordinary edits and `:FredNew file`/`:FredNew dir` express create, move, rename, replace, and delete from final desired state.
33. DELETE_TREE and COPY_TREE each remain one logical operation without advance descendant enumeration.
34. Removing a directory row suppresses unedited descendants in ViewSemanticFrame and evacuates explicitly moved descendants before DELETE_TREE.
35. Directory COPY_TREE and MOVE to self or a descendant are rejected.
36. Directory MOVE emits one parent move plus only required descendant operations in conditional dependency order.
37. Direct namespace probes and repeated final preflight reject missing sources, wrong kinds, occupied destinations, missing/unplanned parents, unrelated planned-parent occupants, non-directory ancestors, and parent traversal through symlinks.
38. Destination creation uses exclusive/no-replace behavior and never overwrites, including races after final preflight.
39. Only ordinary full-buffer write of the current visible Open FRED buffer through Lua `BufWriteCmd` enters planning; one write processes one Instance and exact private URI.
40. Buffer-local `FileWriteCmd` and `FileAppendCmd` reject ranged/append writes before ordinary I/O and never capture or plan.
41. Invalid or conflicted preparation shows non-confirmable diagnostics and preserves dirty edits; a valid plan returns one sealed one-shot PreparedApply; a non-empty plan shows one immutable all-or-nothing preview, while an empty plan skips preview but still passes through final execute-witness checks.
42. `apply_finish` consumes PreparedApply exactly once; duplicate/replayed/foreign/dropped capabilities are rejected, preview mutation cannot alter the plan, and explicit preview cancellation or handle drop releases plan memory without a process-wide prepared-plan registry.
43. Every plan carries projection, edit-base, planned-root, intent, and changedtick authority; Lua establishes `ApplyBufferGuard` before Execute; stale authority is validated before an already-requested execution cancellation may abandon intent; late edits abort without overwrite; every clean or abort path releases the guard.
44. One process-global mutation lock covers every non-empty write-driven Execute and `:FredLink`; before enumerating RootSessions or issuing the first syscall, Apply registers Planner-owned affected paths in a process-wide active mutation fence that existing and newly created RootSessions consult at scan start and publication.
45. Every post-lock outcome coalesces and schedules queued refresh demand, leaves Apply, releases scan-generation fences, removes the process-wide active mutation affected-path fence, and releases the lock before its terminal outcome becomes observable.
46. Executor is serial, no-replace, fail-stop, and cooperatively cancellable; no later node runs after the first failure or accepted cancellation, long/compound operations check safe points, blocking syscalls may finish first, cancellation observed only after final-node success is Success, and no automatic retry or rollback occurs.
47. Same-filesystem MOVE and fully successful cross-filesystem MOVE retain the source NodeId at destination; a PendingCopy-derived MOVE emits an authoritative terminal provisional-to-source NodeIdTransition only after success is known.
48. Cross-filesystem copy success followed by source-delete failure, or cancellation observed after copy success and before deletion starts or before a cancellable deletion completes, retains the source NodeId and a distinct destination NodeId, reusing the PendingCopy provisional destination when available and allocating one otherwise; successful deletion followed only by later cancellation observation remains Success; failed or cancelled partial COPY_TREE discoveries use new NodeIds for unrepresented destination nodes.
49. Execution failure reports COMPLETED/FAILED/UNSTARTED; accepted execution cancellation reports COMPLETED/CANCELLED/UNSTARTED. Both publish known successful effects, resolve terminal identity from those effects, abandon failed/cancelled and unstarted initiating intent, synchronously consume the clean initiating frame and any authoritative NodeIdTransition values, clear `'modified'` before `BufWriteCmd` returns write failure, then defer only best-effort refresh/ancillary UI, so real Neovim 0.12.4 `:wq` remains open with a clean buffer.
50. `:FredLink` resolves relative paths against the buffer directory, creates exclusively, uses the shared mutation runner/global lock/process-wide active affected-path fence/common finalizer, treats directory symlinks as leaves in existing sessions, refreshes overlapping sessions when needed, and always removes the fence and releases the lock.
51. One Instance owns at most one buffer/View while one buffer may appear in multiple windows/tabpages; layouts remain native Lua concerns.
52. Created/Open/Hidden/Destroyed lifecycle, display reconciliation, borrowed/owned layout behavior, cleanup timers, group LRUs, capacity zero, delay zero, and one idempotent terminal destroy path pass.
53. Hidden dirty buffers may be reopened and written before cleanup; terminal destroy discards text/intent without filesystem mutation; synchronous terminal handling that observes destroyed or generation-mismatched state performs no buffer resurrection or overwrite and releases `ApplyBufferGuard`.
54. `vim.b[bufnr].fred`, fixed FileType, keymaps, FileType callbacks, setup/instance `on_attach`, registry lookup, and fail-fast attach rollback preserve their documented order.
55. Reveal obtains required ancestor/target semantic facts before classification, applies precise ignore overrides, persistently enables hidden visibility when needed, uses normal NodeId expansions, and never changes root.
56. All Rust unit, Lua unit, integration, property, and headless Neovim tests pass, FRED runs without Oil, and no Oil runtime dependency exists.

## 27. Residual Risks

- Filesystem races can still occur after direct final preflight begins. FRED intentionally compares source path and kind rather than inode or platform file ID, so a rare external same-path, same-kind replacement may be treated as the original source.
- A fail-stop or cooperatively cancelled batch may partially complete before it stops.
- FRED intentionally does not undo completed operations after execution failure or cancellation.
- Direct COPY_TREE failure or cancellation may leave a partial final destination whose later discovery receives new destination NodeIds.
- Cross-filesystem MOVE is non-atomic and may leave both source and destination if source deletion fails or cancellation is observed after copy success and before deletion starts or before a cancellable deletion completes; the source retains its NodeId and the destination receives a distinct NodeId. If deletion succeeds before cancellation is observed, the logical MOVE succeeds instead.
- Cooperative cancellation cannot forcibly interrupt every blocking system call; the current call may finish before the worker stops at the next safe checkpoint, and a request first observed only after final-node success is Success.
- A process kill or power loss may leave partial state or a temporary rename name.
- Platform-specific metadata may not be preserved by copy operations.
- Linux and Windows path behavior still requires substantial automated testing.
- Other compiling platforms may expose unhandled platform-specific behavior until concrete reports arrive.
- OS watcher behavior and limits differ; degraded watcher coverage must remain visible.
- Very large repositories may require users to narrow baseline depth, local expansion, or ignore rules.
- Retained immutable snapshots increase memory use while dirty Views or active tasks hold strong references; version one relies on ordinary ownership release rather than a separate memory-budget eviction policy.
- A long-lived RootSession may cumulatively retain namespace and metadata cache for scopes visited by earlier generations even after current View coverage moves elsewhere. Per-generation scan limits do not bound that total; all retained cache is guaranteed to release only when the final View/task references release the whole RootSession.
- Metadata columns and metadata-backed Lua sorting add stat calls and platform-specific normalization cost, especially for large covered roots.
- Lua mirrors each View's bounded ViewSemanticFrame catalog, child membership, ProjectedRows, bindings, and classification cache, increasing main-thread work, memory use, and garbage-collection pressure at large coverage limits.
- A slow or failing `is_hidden_file` callback, common projector, FlatProjectionCodec, or ProjectionCommit can delay or abort presentation; errors are intentionally fail-fast and version one adds no renderer rollback or recovery state.
- Actual hidden-file visibility changes intentionally reset Neovim undo history even when visible rows do not change; clean sibling-sort changes reset history only when real-row order changes; unapplied semantic intent survives either reset, but earlier text undo steps do not.
- A failed or unavailable stat may leave a placeholder until the next metadata refresh; it must never be interpreted as a missing namespace entry.
- Extmark and provenance behavior under complex edits requires headless and property testing.
- `nvim-oxi` couples the native build to the exact Neovim 0.12 patch-version/revision pairs listed in build configuration; every new pair requires an explicit native module/`AsyncHandle` bridge compatibility run and may require a rebuild.
- Opening many windows for one instance increases Neovim window/UI cost even though scan and snapshot data remain shared.
- Hidden dirty instances may be destroyed by either `cleanup.instance.delay_ms` or their cleanup group's capacity overflow; unwritten edits are intentionally lost at that point, and `capacity = 0` makes hiding immediately terminal.
- Native jumplist/alternate-buffer behavior depends on normal Neovim window buffer switching and is not supplemented by a Fred history stack.
- Exact-root and explicit lineage sharing means one RootSession may serve Views rooted at different descendant NodeIds; path rebasing, coverage reference counting, and root-node deletion/move cases require dedicated tests.
- Directory symlink targets intentionally open as separate RootSessions, so expansion, NodeId, intent, and provenance do not cross that boundary automatically.
- Version one proves the projection-neutral boundary only with FlatProjectionCodec; a future editable tree codec will still require its own indentation/parent reconstruction, invalid-layout, cut/put, dirty-switch, and undo tests even though Rust types and planner remain unchanged.
- User `on_attach`, keymap, hidden-file, and custom action callbacks execute inside Neovim and may fail or introduce user-side latency.

These risks are explicit constraints. The design favors a small namespace planner, direct platform operations, one global mutation gate, serial fail-stop execution, and best-effort refresh over complex guarantees for rare interruption scenarios.
