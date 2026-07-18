# FRED Design Specification

**Status:** Written specification pending final review
**Date:** 2026-07-15
**Last revised:** 2026-07-18
**Project:** `fred.nvim`
**Expansion:** Filesystem Representation Editor

## 1. Product Summary

FRED is a buffer-native file browser for Neovim. It projects selected parts of a local filesystem root into an ordinary editable buffer. The user browses a flat list of complete root-relative paths, edits that list with normal text operations, and writes the current FRED buffer to preview and execute the resulting filesystem operations.

Example projection for `/home/user/project`:

```text
README.md
docs/
docs/design.md
src/
src/init.lua
src/parser.lua
tests/
```

Directories end with `/`. Every other editable byte belongs to the canonical Lua-owned path representation. Icons, permissions, size, timestamps, diagnostics, and operation state are Neovim decorations rather than editable path text.

FRED owns its filesystem model, scanner, watcher integration, planner, executor, Lua presentation, Instance lifecycle, layouts, actions, and tests. Its product description is:

> FRED is a buffer-native file browser for Neovim. Browse files, edit the filesystem.

## 2. Version-One Contract

Version one provides:

1. Local filesystem browsing from a configurable root.
2. A flat editable projection of complete paths relative to the current Instance root.
3. Baseline recursion depth plus per-directory expansion and collapse.
4. Sorting, hidden-file visibility, ignore rules, metadata columns, scan limits, and partial coverage.
5. Normal text editing for file and directory creation, rename, move, copy, replacement, and deletion.
6. Same-RootSession copy provenance from supported Vim yank and put operations.
7. Stable RootSession-scoped `NodeId` identity independent of path and lifecycle state.
8. Logical directory copy, move, and deletion operations with incremental cancellable execution.
9. Immutable filesystem snapshots shared by Instances that use the same compatible RootSession.
10. Watcher-driven scoped refresh with manual refresh available at all times.
11. Validation, logical preview, confirmation, serial execution, partial-outcome reporting, and reconciliation.
12. One Engine-wide mutation gate for every FRED filesystem mutation in one Neovim process.
13. A stable Lua Instance API with native Neovim layouts, actions, keymaps, attach callbacks, cleanup timers, and cleanup-group capacity.
14. Lua-owned rendering, edit detection, buffer state, event-loop integration, and Neovim lifecycle handling.
15. An editor-independent Rust core that can be reused by additional presentation adapters.
16. A LuaJIT native module built with `mlua` and loaded through a versioned platform artifact.

The version-one projection is flat. A directory and its materialized descendants remain contiguous through sibling-local sorting followed by depth-first traversal, while every displayed row remains a complete root-relative path.

## 3. Component Map

```text
┌──────────────────────────────────────────────────────────────┐
│ Lua Neovim presentation                                     │
│                                                              │
│ Instance API, layouts, buffers, extmarks, changedtick,       │
│ undo, cursor, actions, keymaps, lifecycle, cleanup,          │
│ path text codec, baseline comparison, semantic intent        │
│ detection, preview UI, polling timer, rendering, reveal UI   │
└──────────────────────────────┬───────────────────────────────┘
                               │ Lua calls and polling
┌──────────────────────────────▼───────────────────────────────┐
│ fred-lua                                                     │
│                                                              │
│ mlua module-mode cdylib, opaque userdata and token           │
│ conversion, native-name transport, poll/cancel/shutdown      │
└──────────────────────────────┬───────────────────────────────┘
                               │ ordinary Rust API
┌──────────────────────────────▼───────────────────────────────┐
│ fred-core                                                    │
│                                                              │
│ Engine, RootSession, native paths, NodeId, immutable          │
│ snapshots, scanner, metadata, watchers, coverage leases,     │
│ semantic queries, tasks, Planner, Executor, reconciliation   │
└──────────────────────────────────────────────────────────────┘
```

The dependency direction is:

```text
fred-lua -> fred-core
Lua plugin -> fred-lua
```

`fred-core` is a normal Rust library. It has no editor, Lua, buffer, renderer, cursor, extmark, changedtick, window, or event-loop types.

## 4. Normative Specification Index

The specification is split by state owner. Each contract has one normative home.

1. [`spec/01-architecture.md`](spec/01-architecture.md)
   Process architecture, dependency direction, Engine topology, native bridge, task transport, polling, handles, shutdown, and mutation coordination.

2. [`spec/02-filesystem-model.md`](spec/02-filesystem-model.md)
   Native paths, NodeId, RootSession, immutable snapshots, scanner publication, coverage leases, metadata, watchers, symlinks, semantic queries, reveal resolution, and lifetime.

3. [`spec/03-lua-presentation.md`](spec/03-lua-presentation.md)
   Flat projection, Lua path codec, extmark identity, committed baseline, intent detection, BufferRuntime, asynchronous write state, Conflict state, multi-buffer synchronization, and reveal presentation.

4. [`spec/04-planning-execution.md`](spec/04-planning-execution.md)
   Semantic intents, prepare, probes, Planner, one-shot `PreparedApply`, preview, mutation gate, Executor, safe-point cancellation, operation ordering, identity effects, partial outcomes, and reconciliation.

5. [`spec/05-api-lifecycle.md`](spec/05-api-lifecycle.md)
   Public Instance API, configuration, actions, commands, layouts, buffer metadata, attach callbacks, lifecycle, cleanup timers, cleanup groups, and user-visible errors.

6. [`spec/06-testing-delivery.md`](spec/06-testing-delivery.md)
   Layered tests, compatibility, release artifacts, performance verification, milestone exit criteria, and operational risks.

When explanatory text in another file appears to conflict with the normative owner, the normative owner controls and the duplicate must be corrected rather than implemented as a second protocol.

## 5. Cross-Specification Invariants

### 5.1 Ownership

All Neovim API calls and all editor state belong to Lua. Rust owns filesystem/domain state and operations. `fred-lua` performs conversion only.

### 5.2 Identity

Path is not identity. `NodeId` is opaque, RootSession-scoped, and transported to Lua as a fixed binary token rather than a Lua number or editable string.

### 5.3 Projection And Intent

Lua renders the filesystem model, tracks extmark identity, compares the current buffer with its committed Lua baseline, and submits already semantic intents. Rust never parses FRED buffer text.

### 5.4 Snapshot Publication

Only a RootSession coordinator publishes immutable snapshots. Scanner candidates built against an obsolete model generation or `RootMutationEpoch` cannot publish and cause a scoped rescan. Canonical physical mutation-fence installation is atomic with intersecting scan admission: a racing scan either starts before the existing session's epoch increment and becomes stale, or remains queued until the fence clears.

### 5.5 Watchers

Watcher events are coalesced filesystem-change hints. Scans and direct probes establish model facts.

### 5.6 Mutation Authority

Prepare performs validation and planning without filesystem mutation. Finish consumes one sealed `PreparedApply`, obtains the Engine mutation gate, repeats required probes, and invokes the serial Executor.

### 5.7 Cancellation

Tasks are cooperatively cancellable. After the first filesystem mutation, Executor stops at the next Planner-defined safe point. It does not forcibly terminate an in-flight system call. Completed effects remain authoritative and are reconciled.

### 5.8 Draft Preservation

A pre-commit error or cancellation preserves the edited Lua draft. A post-commit failure or cancellation reconciles the filesystem model and enters Lua `Conflict` state without silently replacing the draft.

### 5.9 Exclusive Destinations

Create, copy, move, replacement installation, and symlink creation use exclusive destination behavior. An occupied destination produces an error rather than overwrite.

### 5.10 Symlinks

Filesystem scanning treats symlinks and Windows reparse-point directory aliases as leaf nodes. Version one does not traverse them as directory ownership edges.

### 5.11 Bounded Delivery

Rust workers publish through bounded queues and retained terminal-result storage. Lua owns one adaptive scheduled polling loop and drains bounded work per turn.

### 5.12 Lifecycle

Lua Instance destruction discards only unapplied editor state. Filesystem mutation tasks that crossed the commit boundary continue to a safe terminal outcome, publish known effects, and reconcile even if their originating buffer disappears.

## 6. End-To-End Data Flows

### 6.1 Browse And Render

```text
Lua coverage lease request
  -> Rust scanner/metadata tasks
  -> immutable RootSnapshot publication
  -> ModelChanged event
  -> Lua acquires one SnapshotHandle
  -> Lua queries semantic nodes
  -> Lua applies sort/filter/expansion
  -> Lua encodes flat path lines
  -> Lua renders buffer and NodeId extmarks
  -> Lua commits its local baseline
```

### 6.2 Edit And Apply

```text
user edits FRED buffer
  -> Lua parses canonical path text
  -> Lua compares extmarks and committed baseline
  -> Lua constructs semantic intents
  -> Rust prepare validates, probes, and plans
  -> Lua previews the immutable logical plan
  -> user confirms
  -> Rust finish consumes PreparedApply
  -> Engine mutation gate and final probes
  -> serial safe-point-cancellable execution
  -> known-effect snapshot publication
  -> scoped reconciliation
  -> Lua renders success or enters Conflict
```

### 6.3 Reveal

```text
Lua requests NodeId or root-relative native components
  -> Rust resolves/scans target and ancestors
  -> Lua expands ancestor NodeIds
  -> Lua applies temporary forced visibility
  -> Lua renders one fixed snapshot
  -> Lua moves cursor to the target extmark
```

## 7. Milestones

Development proceeds through separately planned milestones:

```text
M0  native bridge and Neovim event-loop proof
M1  filesystem model, scanning, coverage, metadata, and watchers
M2  Lua projection, edit detection, multi-buffer synchronization, and reveal
M3  Planner, preview, Executor, cancellation, partial outcomes, and reconciliation
M4  release artifacts, compatibility matrix, performance, and distribution hardening
```

The written design is the umbrella architecture. After this written specification receives final written approval, the next planning activity is an implementation plan for M0 only. M1, M2, M3, and M4 each receive a separate implementation plan only after the preceding milestone has completed its exit criteria and supplied verified constraints.

## 8. Specification Discipline

Normative text states the current design directly. State machines, ownership, schemas, and acceptance conditions are defined once in their owning file. Implementation plans may add task ordering and file-level work, but they must preserve the boundaries and invariants linked above.
