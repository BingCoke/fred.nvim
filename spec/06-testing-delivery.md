# FRED Testing, Compatibility, And Delivery

This file is the normative source for verification layers, compatibility targets, native artifacts, loader diagnostics, performance budgets, milestone exit criteria, release readiness, and operational risks.

## 1. Supported Host Contract

Version one supports:

```text
Neovim >= 0.12.4
LuaJIT host runtime
local filesystem roots
```

The Lua adapter performs a host check before loading the native module. A host diagnostic reports:

```text
FRED Lua version
expected native API version
Neovim version
Lua runtime
OS
architecture
platform ABI
selected artifact path
```

A compatibility release is supported only when its automated load and headless behavior tests pass.

## 2. Native Artifact Identity

Artifact identity is:

```text
FRED version
OS
architecture
platform ABI
Lua runtime ABI
```

Initial release target names are:

```text
linux-x86_64-gnu-luajit
linux-aarch64-gnu-luajit
macos-x86_64-luajit
macos-aarch64-luajit
windows-x86_64-msvc-luajit
```

Each artifact contains the `fred_native` module with the platform extension and a manifest containing:

```text
fred_version
native_api_version
target_triple
platform_abi
lua_runtime_abi
build_profile
checksum
```

The loader validates the manifest and `native.api_version()` before opening the Engine.

`fred.build()` can produce the current target artifact from source using the same naming and manifest contract.

## 3. Loader Behavior

The loader:

1. determines host OS, architecture, platform ABI, and Lua runtime;
2. selects the artifact identity;
3. verifies artifact presence and checksum when supplied by a release manifest;
4. loads `fred_native`;
5. verifies native API version and build information;
6. opens the singleton host Engine with validated limits;
7. reports a structured diagnostic when any stage fails.

An incompatible or missing artifact does not create a partial Instance. Public `new()` reports the native-load diagnostic before buffer creation.

The native library remains loaded until Neovim exits. A binary upgrade takes effect after process restart.

## 4. Verification Layers

FRED uses four primary verification layers:

```text
fred-core Rust tests
fred-lua native-module tests
headless Neovim Lua/integration tests
cross-platform release smoke tests
```

Property, fault-injection, concurrency, and performance tests are distributed into their owning layer.

## 5. `fred-core` Unit Tests

Run:

```text
cargo test -p fred-core
```

The test environment does not start Neovim or Lua.

### 5.1 Native Paths And Names

Verify:

- Unix raw filename bytes;
- Windows UTF-16 code units;
- component validation;
- root containment;
- canonical root identity;
- platform separator rejection;
- Windows reserved components;
- case/collision behavior through the namespace rules layer;
- native path component round trips.

### 5.2 NodeId And Snapshot Invariants

Verify:

- RootSession-scoped uniqueness;
- fixed binary transport round trip;
- stable identity for refresh and FRED relocation;
- external rename as delete plus create;
- create/copy provisional identity retention;
- cross-filesystem identity split;
- parent/child/path-index consistency;
- symlink/reparse leaf invariants;
- immutable publication;
- Complete/Partial/Failed/Dirty coverage behavior;
- RetainedUnverified propagation.
- provisional reservation allocation, pre-commit reuse, promotion, retirement, explicit release, and destruction cleanup;
- Snapshot counter/event/task-result/SnapshotHandle generation equality.

### 5.3 Coverage And Lifetime

Verify:

- lease union and release;
- traversal inclusion when any lease requires a path;
- metadata requirement union;
- temporary reveal lease release;
- weak RootSession registry behavior;
- Snapshot/task strong references;
- final RootSession release.

### 5.4 Planner

Verify:

- every semantic intent type;
- duplicate destination rejection;
- parent creation order;
- descendant evacuation;
- directory move rebasing;
- replace ordering;
- rename swaps, cycles, and case-only renames;
- cross-filesystem move program;
- identity dispositions;
- final-probe expectations;
- deterministic preview;
- deterministic output for identical immutable input.

### 5.5 Executor

Verify:

- serial group order;
- exclusive destinations;
- fail-stop behavior;
- safe-point cancellation;
- cancellation after final success;
- incremental file copy;
- incremental copy-tree;
- post-order incremental delete-tree;
- rename-cycle group cancellation deferral;
- known-effect report;
- post-commit terminal classifications;
- common mutation finalizer and gate release.

## 6. Fault-Injection Tests

`FaultInjectingFs` supports deterministic failures and barriers.

Required scenarios include:

```text
third rename fails
copy creates destination then fails
cross-filesystem copy succeeds and source delete fails
delete-tree cancels after N entries
rename-cycle cancellation arrives after first temporary rename
final destination appears after final probes
permission failure on one child
watcher hint arrives during mutation
worker panic while mutation coordinator is active
```

Assertions cover:

- exact known effects;
- unstarted operations;
- NodeId outcomes;
- partial/dirty scopes;
- reconciliation demand;
- terminal-result retention;
- mutation gate release;
- future scan publication safety.

## 7. Deterministic Concurrency Tests

Concurrency tests use barriers and explicit channels rather than timing sleeps.

Required publication scenario:

```text
scan starts from Snapshot generation 10
scanner pauses before handoff
mutation publishes generation 11
scanner resumes
candidate generation 10 is rejected
requested scope is rescanned
```

Required fence-admission scenario:

```text
mutation installs overlapping physical fence and advances an existing session epoch
mutation pauses after an intermediate filesystem effect
scan demand reaches that session's admission point
no scanner worker starts and no candidate handoff is possible
mutation reaches its terminal finalizer and clears the fence
queued current demand is admitted from the current Snapshot generation and fixed epoch
the post-clearance scanner reaches candidate handoff after reading terminal filesystem state
the candidate publishes no intermediate Snapshot
```

Additional scenarios:

- scanner candidate reaches publication while an overlapping canonical physical-path fence is active;
- a scan admitted immediately before fence installation reaches handoff after clearance and is rejected by its stale epoch;
- two scan generations complete in reverse order;
- a nested RootSession registered during an intersecting active mutation initializes its epoch, starts fenced, admits no initial scanner, preserves that epoch through clearance, and then receives reconciliation/current demand;
- every RootSession already intersecting at fence installation advances `RootMutationEpoch` exactly once;
- the first post-fence reconciliation/current-demand scan captures each session's fixed epoch;
- known-effect Snapshot, RootSession counter, model event, task result, and SnapshotHandle expose one identical generation;
- watcher overflow coalesces with mutation reconciliation;
- cancellation races with gate acquisition;
- shutdown races with terminal task delivery;
- terminal result is consumed exactly once;
- a full transient queue does not block cancellation or terminal storage.

## 8. Property Tests

Generated snapshots, intents, and plans verify:

- one-parent namespace trees;
- path index and child membership agreement;
- no child membership under symlink/reparse leaves;
- Complete coverage alone authorizes absent-child removal;
- NodeId uniqueness and scope;
- no operation escapes root;
- no duplicate final destination;
- directory copy/move never targets itself or a descendant;
- parent dependencies precede child installation;
- copy precedes source delete;
- no group runs after failure or accepted cancellation;
- successful move leaves one source identity at destination;
- split cross-filesystem outcome leaves distinct identities;
- rejected scan candidates never advance current Snapshot;
- Snapshot contents, counters, events, task results, and handles expose the same generation;
- canonical physical fences cover existing and newly registered overlapping RootSessions;
- every existing intersecting RootSession advances `RootMutationEpoch` once at fence installation;
- a newly registered intersecting RootSession initializes its epoch, admits no scan while fenced, and keeps that epoch through post-fence reconciliation;
- scan admission atomicity ensures every racing intersecting demand either starts before the epoch increment and becomes stale or remains queued until clearance;
- every mutation terminal path releases the Engine mutation gate.

## 9. `fred-lua` Tests

Run Rust-side bridge tests plus a real LuaJIT module load test.

Verify:

- module-mode loading;
- API/build version records;
- compatible singleton Engine open;
- incompatible re-open error;
- opaque userdata liveness;
- fixed binary NodeId tokens;
- Unix byte-name transport;
- Windows UTF-16LE transport;
- malformed request tables;
- foreign/stale/released handles;
- `PreparedApply` one-shot consumption;
- poll count and payload bounds;
- terminal-result retention;
- cancel delivery under queue pressure;
- poll/shutdown behavior;
- worker panic conversion;
- absence of Lua references in worker-owned task data.
- provisional reservation reuse after validation failure and preview cancellation;
- explicit provisional release on pending-row removal and buffer destruction;
- reservation promotion/retirement dispositions in terminal results.

## 10. Lua Unit Tests

Lua unit tests cover pure presentation and state logic:

### 10.1 Path Codec

- canonical escapes and uppercase hexadecimal;
- Unix invalid UTF-8 bytes;
- Windows unpaired UTF-16 units;
- leading/trailing spaces;
- literal backslashes;
- malformed and alias escapes;
- flat component separators;
- directory trailing slash;
- native transport round trip.

### 10.2 Projection

- sibling-local sorting;
- depth-first contiguous subtrees;
- depth and expansion;
- hidden-directory subtree propagation;
- ignore rules and negation;
- forced-visible reveal chain;
- metadata columns and display width;
- dirty sibling-order freeze;
- fixed Snapshot generation per render.

### 10.3 Intent Detection

- rename;
- move;
- rename plus move;
- create file/directory;
- delete file/tree;
- same-parent reorder as no-op;
- moved subtree producing one root relocation;
- duplicate binding rejection;
- existing kind change rejection;
- provisional NodeId allocation;
- validation failure followed by retry with the same provisional binding;
- preview cancellation followed by retry with the same provisional binding;
- undo/removal releasing a provisional reservation;
- same-session provenance classification;
- multiple provenance destinations;
- foreign provenance becoming ordinary create.

### 10.4 State Machines

- exhaustive guarded BufferRuntime transitions, including `Editing -> Preparing`, `Conflict -> Preparing`, `Preparing + NonEmptyPrepared -> Previewing`, `Preparing + EmptyPrepared -> Executing`, and `Previewing + Confirmed -> Executing`, plus every terminal Apply result;
- attempt-id staleness;
- ordinary-attempt and Conflict-retry origins;
- pre-commit draft preservation;
- Empty completion without reconciliation;
- success baseline commit after owner reconciliation;
- post-commit zero-known-effect failure entering Conflict;
- Conflict construction;
- retry-remaining transition `Conflict -> Preparing`;
- retry pre-commit failure returning to the same Conflict;
- discard-and-reload transition `Conflict -> Editing`;
- model-dirty handling;
- destroy from every state;
- cleanup timer and LRU races.

## 11. Headless Neovim Tests

Run against real supported Neovim executables with a minimal init.

### 11.1 Native/Event-Loop Boundary

Verify:

- `fred_native` loads;
- one Lua-owned timer polls;
- timer callback enters scheduled Lua;
- Rust progress remains responsive;
- bounded drain work yields between batches;
- no worker invokes a Lua callback.

### 11.2 Buffer Identity And Routing

Verify:

- unique `fred://` names;
- `filetype=fred` and `buftype=acwrite`;
- current/Open/displayed/exact-URI write checks;
- ranged writes route to `FileWriteCmd` and are rejected;
- append writes route to `FileAppendCmd` and are rejected;
- `BufWriteCmd` changes no buffer lines inside the autocmd;
- asynchronous write returns with `modified=true`;
- combined write-and-quit remains open while apply is pending.

### 11.3 Extmarks

Record and assert behavior for:

```text
filename edit
path-parent edit
:move
dd/p
copy/paste
undo/redo
full internal render
line deletion
line duplication
```

The tests establish which native edits preserve the authoritative NodeId binding.

### 11.4 Apply UX

Verify:

- prepare freezes the buffer after the autocmd returns;
- preview cancellation restores editable dirty draft and keeps provisional reservations reusable;
- stale/precondition errors preserve draft, undo, and valid provisional bindings;
- empty plan renders immediately without a reconciliation wait and clears modified;
- success waits for the owner reconciled model and clears modified;
- cancellation while waiting for gate preserves draft;
- cancellation during large delete stops at a safe point;
- every post-commit failure/cancellation, including zero known effects, retains the draft and enters read-only Conflict after reconciliation;
- retry remaining creates a new prepare attempt and a retry pre-commit error returns to the existing Conflict;
- discard and reload releases remaining provisional reservations and commits the reconciled model;
- destroying a buffer releases provisional reservations and does not interrupt Core finalization after commit.

### 11.5 Multi-Instance And Lifecycle

Verify:

- exact-root Instances share model data;
- buffers retain independent expansion/sort/filter/baselines;
- a clean buffer auto-renders model changes;
- a dirty buffer becomes model-dirty without overwrite;
- navigation preserves same-session NodeIds;
- child Instance shares the parent session when applicable;
- layouts and display reconciliation;
- FileType/attach ordering;
- cleanup delay and group capacity;
- timer/LRU/destroy idempotency.

### 11.6 Reveal

Verify:

- uncovered ancestors trigger scoped resolution;
- hidden/ignored target chain is temporarily forced visible;
- expansion and cursor movement use NodeId;
- stale target retries once;
- symlink intermediate traversal fails;
- reveal cancellation releases temporary coverage.

## 12. Cross-Platform Filesystem Matrix

### 12.1 Linux

- arbitrary non-NUL filename bytes;
- inotify watcher behavior and overflow;
- permission-denied directories;
- broken symlinks;
- cross-filesystem move between test mounts when CI permits.

### 12.2 macOS

- FSEvents behavior;
- x86-64 and ARM64 module loading;
- case-insensitive and case-sensitive volume behavior where available;
- symlink leaf behavior;
- dynamic module symbol loading.

### 12.3 Windows

- UTF-16 native names;
- reserved components;
- case-insensitive collision;
- files held open by another process;
- junction and reparse leaf behavior;
- drive roots and path containment;
- `ReadDirectoryChangesW` watcher behavior.

## 13. Performance Budgets

Default runtime budgets are:

```text
Rust transient queue capacity:   1024 events
one poll event limit:             128 events
one poll node-record limit:       500 records
Lua scheduled drain target:       <= 4 ms
Lua scheduled drain hard yield:   8 ms
active polling interval:          16 ms
watcher-only polling interval:    200 ms
```

Performance verification includes:

- initial scan and projection at configured entry limits;
- one 50,000-entry directory queried and rendered through bounded Snapshot pages;
- repeated expansion and collapse;
- metadata-column demand;
- large Snapshot query batches;
- large semantic intent preparation;
- long copy/delete cancellation latency;
- watcher storms and overflow;
- multiple buffers sharing one RootSession;
- garbage-collection pressure from repeated render batches.

A still-live RootSession may retain previously visited model and metadata cache. Per-request scan limits bound current task work rather than cumulative RootSession lifetime cache.

## 14. Milestone M0: Bridge And Event Loop

M0 includes:

```text
fred-core/fred-lua workspace
mlua module load
native API/build version checks
singleton Engine
opaque handles and NodeId token
bounded mailbox and retained terminal results
poll/cancel/shutdown
Lua scheduled adaptive timer
private buffer URI
BufWriteCmd/FileWriteCmd/FileAppendCmd routing
asynchronous write-state experiment
extmark behavior matrix
native filename transport round trip
headless Neovim harness
release-artifact smoke workflow
```

M0 may use controlled fake tasks and mock semantic data.

M0 exit criteria:

- every target artifact loads on its target host or is explicitly absent from the release matrix;
- worker delivery reaches Lua only through bounded polling;
- cancellation and terminal retention work under queue pressure;
- Neovim operations occur only in scheduled Lua;
- write autocmd routing is verified on Neovim 0.12.4;
- buffer contents remain unchanged during `BufWriteCmd`;
- extmark behavior is recorded by repeatable tests;
- native name and NodeId transport are lossless.

## 15. Milestone M1: Filesystem Model

M1 includes:

```text
RootSession
native paths and NodeId
immutable RootSnapshot
custom scanner
coverage leases
metadata snapshot
watcher hints
canonical physical mutation fence
RootMutationEpoch
publication coordinator
stale candidate rejection
paged semantic Snapshot queries
symlink/reparse leaf policy
```

M1 exit criteria:

- pure Core tests pass on the platform matrix;
- scanner/mutation admission and publication races are deterministic and safe across overlapping RootSessions;
- each physical fence advances every existing intersecting epoch once, newly registered intersecting sessions initialize one unchanged fenced epoch, and post-fence reconciliation/current-demand scans capture those fixed epochs;
- no intersecting scanner starts while its RootSession is fenced;
- Snapshot generations agree across stored model, counters, events, results, and handles;
- watcher overflow recovers through scoped rescan;
- Complete/Partial/Failed authority is correct;
- paged large-directory queries preserve responsiveness and one pinned generation;
- shared RootSession lifetime and cache behavior are verified.

## 16. Milestone M2: Lua Projection

M2 includes:

```text
Instance presentation state
flat path codec
Snapshot rendering
NodeId extmarks
committed baseline
semantic intent detection
provenance
columns
multi-buffer synchronization
reveal
navigation
BufferRuntime draft states
```

M2 exit criteria:

- Lua renders and edits the flat filesystem projection;
- every supported edit maps deterministically to semantic intents;
- projection absence cannot create unintended delete;
- modified buffers survive external model updates;
- reveal is split correctly between Core resolution and Lua presentation.

## 17. Milestone M3: Planner And Executor

M3 includes:

```text
prepare validation and probes
pure Planner
one-shot PreparedApply
preview
mutation queue/gate
final preflight
serial Executor
incremental copy/delete
safe-point cancellation
known-effect publication
partial Conflict outcomes
reconciliation
fault injection
```

M3 exit criteria:

- create/copy/move/delete/replace/tree operations plan and execute correctly;
- cancellation before commit has no effects;
- cancellation after commit stops at a safe point;
- every partial outcome reports known effects and reconciles;
- exclusive destinations hold under injected races;
- Lua success and Conflict state match the terminal result.

## 18. Milestone M4: Distribution And Hardening

M4 includes:

```text
complete artifact matrix
checksums and manifests
loader diagnostics
platform CI
compatibility smoke tests
large-repository benchmarks
performance tuning within fixed boundaries
installation documentation
release automation
```

M4 exit criteria:

- every published artifact has a real load test;
- loader errors identify the exact host/artifact mismatch;
- native API mismatch is rejected before Engine creation;
- performance budgets pass representative repositories;
- cross-platform path, watcher, symlink, cancellation, and packaging suites pass.

## 19. Planning Sequence

After this written specification receives final written approval, the next planning activity is an implementation plan for M0 only.

M1, M2, M3, and M4 receive separate implementation plans after the preceding milestone has completed its exit criteria and supplied verified constraints.

## 20. Release Readiness

Version one release readiness requires:

- all normative conformance clauses in the six specification files pass;
- all supported artifact targets load and report matching API/build information;
- pure Core, native bridge, Lua unit, headless Neovim, fault-injection, property, concurrency, and cross-platform suites pass;
- no operation overwrites an occupied destination;
- stale scanner candidates cannot overwrite a newer mutation model;
- canonical physical mutation fences cover overlapping existing and newly registered RootSessions and atomically block intersecting scan admission until clearance;
- existing intersecting sessions increment once at installation, while newly registered intersecting sessions initialize and retain their own epoch through reconciliation;
- Snapshot generation is identical across model, counters, events, results, and handles;
- provisional NodeIds have deterministic retry, promotion, retirement, and release lifecycles;
- pre-commit outcomes preserve drafts;
- post-commit partial outcomes reconcile and preserve Conflict drafts;
- cooperative cancellation is responsive at documented safe points;
- Instance API, layouts, lifecycle, columns, provenance, cleanup, reveal, and multi-buffer behavior pass their headless tests.

## 21. Operational Risks

The release documentation communicates these constraints:

- filesystem namespace races may still occur after final probes and are handled by exclusive syscalls plus reconciliation;
- a blocking syscall may complete before cancellation reaches the next safe point;
- post-commit failure or cancellation may leave a partial filesystem result;
- cross-filesystem moves may temporarily or terminally contain both source and destination;
- process termination may leave partial copies or temporary rename names;
- watcher delivery and platform limits vary, so manual refresh and degraded status remain visible;
- very large roots may require narrower depth, expansion, or ignore rules;
- long-lived RootSessions may retain cache from previously visited scopes;
- metadata columns add stat work;
- extmark behavior depends on tested Neovim editing semantics;
- user callbacks and actions execute in Neovim and may add latency or raise errors.

These risks are verified through explicit tests and surfaced through structured state rather than hidden from the user.
