# FRED Architecture And Native Boundary

This file is the normative source for process architecture, dependency direction, Engine topology, native module behavior, task transport, polling, handles, shutdown, and process-wide mutation coordination.

## 1. Workspace And Dependency Direction

The Rust workspace contains two crates:

```text
crates/
  fred-core/
  fred-lua/
```

The Lua plugin lives under:

```text
lua/fred/
plugin/fred.lua
ftplugin/fred.lua
```

The dependency direction is:

```text
fred-lua -> fred-core
Lua plugin -> fred-lua
```

`fred-core` is an editor-independent Rust library. Its public model consists of filesystem/domain types and ordinary Rust handles. `fred-lua` is a module-mode `cdylib` that converts between Lua values and those Rust types.

A representative native crate configuration is:

```toml
[lib]
name = "fred_native"
crate-type = ["cdylib"]

[dependencies]
fred-core = { path = "../fred-core" }
mlua = {
  version = "0.12",
  features = ["luajit", "module", "macros"]
}
```

The module uses the Lua runtime already hosted by Neovim. Lua values remain on the calling Neovim thread. Rust worker threads exchange only owned Rust data.

## 2. Ownership Matrix

### 2.1 `fred-core`

`fred-core` owns:

- `Engine` and its process-wide mutation coordinator;
- canonical filesystem roots and `RootSession` values;
- native path and native filename types;
- RootSession-scoped `NodeId` allocation;
- immutable namespace and metadata snapshots;
- scanning, direct probes, metadata collection, watcher state, and coverage leases;
- semantic filesystem queries and reveal resolution;
- task handles, cancellation tokens, bounded worker delivery, and terminal-result retention;
- semantic intent validation;
- Planner, `ValidPlan`, `PreparedApplyCore`, Executor, operation reports, known-effect publication, and reconciliation.

### 2.2 `fred-lua`

`fred-lua` owns:

- native module registration;
- conversion between Lua tables and Rust request/result DTOs;
- `mlua::UserData` wrappers for opaque Rust handles;
- fixed binary token conversion for `NodeId`;
- lossless native-name transport conversion;
- exported `open`, `poll`, `cancel`, `shutdown`, query, task, and apply methods;
- conversion of Rust errors into structured Lua errors/results.

### 2.3 Lua Neovim Presentation

Lua owns:

- every `vim.api`, `vim.fn`, `vim.bo`, `vim.wo`, `vim.uv`, autocmd, timer, buffer, window, extmark, cursor, changedtick, undo, and scheduling operation;
- the public Instance object, configuration inheritance, registry, actions, commands, keymaps, layouts, attach callbacks, lifecycle, timers, and cleanup groups;
- presentation state, sorting, hidden-file classification, ignore presentation, expansion, materialization, metadata columns, and cursor reconciliation;
- flat path text encoding and decoding;
- extmark identity bindings, committed buffer baselines, provenance tracking, semantic intent detection, diagnostics-to-row mapping, preview UI, and Conflict UI;
- the one adaptive polling timer and all native-event draining.

## 3. Engine Topology

`fred-core` exposes an ordinary constructible `Engine` so tests, command-line programs, and future adapters may create isolated engines.

The Lua host uses exactly one Engine instance:

```lua
local native = require("fred_native")
local engine = native.open(config)
```

`native.open(config)` has the following behavior:

- the first compatible call creates the host Engine;
- later compatible calls return the same Engine handle;
- a call with incompatible Engine configuration returns `EngineAlreadyConfigured`;
- `shutdown()` moves that host Engine to its terminal state;
- an Engine handle used after terminal shutdown returns `EngineShutdown`.

The single host Engine owns one process-wide mutation gate. Every FRED filesystem mutation, including an immediate symlink request, enters through that gate.

The Engine contains:

```text
Engine {
  state,
  root_session_registry,
  task_registry,
  mailbox,
  terminal_result_store,
  mutation_coordinator,
  configuration,
}
```

The RootSession registry uses canonical root identity and compatible core configuration. Registry entries do not hold presentation state.

## 4. Opaque Handles

Lua receives userdata handles for stateful Rust objects:

```text
EngineHandle
RootSessionHandle
CoverageLeaseHandle
SnapshotHandle
TaskHandle
PreparedApplyHandle
```

Handle rules:

- each handle carries an Engine identity and generation/liveness witness;
- a handle from another Engine is rejected;
- released, consumed, or shutdown handles return structured errors;
- release operations are idempotent where ownership permits;
- `PreparedApplyHandle` has one executable `Fresh -> Consumed` transition;
- `SnapshotHandle` pins one immutable `Arc<RootSnapshot>` and is read-only;
- dropping Lua userdata releases only its Rust reference; filesystem side effects follow task and commit rules rather than garbage-collection timing.

`NodeId` is a value identity rather than a userdata object. Across Lua it is a fixed opaque binary token. Lua may compare it and use it as a table key, but it does not parse or construct it. The representation never passes through a Lua number.

## 5. Native Names And Paths Across Lua

Rust keeps filesystem identity in native types:

```text
NativeName
NativeRelativePath
CanonicalRoot
```

The Lua boundary uses tagged lossless transport:

```text
UnixNameTransport {
  encoding = "unix-bytes",
  data = binary Lua string,
}

WindowsNameTransport {
  encoding = "windows-utf16le",
  data = binary Lua string,
}
```

A root-relative path is transported as an ordered list of native-name components. The bridge converts components to and from Rust native types. Lua owns the editable text representation described in `03-lua-presentation.md`; the bridge does not parse buffer syntax.

## 6. Task Model

Every operation with potentially long-running filesystem I/O or unbounded CPU work returns a `TaskHandle`:

```text
start_scan
start_refresh
start_metadata
start_reveal_resolution
start_apply_prepare
start_apply_finish
```

Bounded setup and in-memory operations remain synchronous:

```text
open_root                 canonicalize one local root, then perform registry lookup
create_coverage_lease
update_coverage_lease
acquire_snapshot
query_snapshot_page
allocate_node_ids
cancel
release
shutdown
```

`open_root` completes canonical root validation before returning its `RootSessionHandle`. Snapshot queries are read-only and paged so one synchronous call obeys the configured record budget.

Each task has:

```text
TaskRecord {
  task_id,
  kind,
  root_session_id,
  cancellation: AtomicBool,
  phase,
  terminal_state,
}
```

Task phases are domain-specific, but every task has exactly one terminal state:

```text
Succeeded
Failed
Cancelled
PartiallyApplied
```

A terminal task result is retained until Lua consumes it or the owning Engine is shut down.

## 7. Worker Delivery And Backpressure

Rust worker threads communicate through bounded `crossbeam-channel` queues. Version one uses no general async runtime.

The Engine mailbox separates transient delivery from terminal ownership:

```text
TransientMailbox {
  bounded progress events,
  bounded model-change hints,
  coalesced watcher-rescan scopes,
}

TerminalResultStore {
  at most one retained terminal result per task,
  consumed exactly once,
}
```

Rules:

1. progress events may be replaced by newer progress for the same task;
2. repeated watcher paths coalesce into the smallest safe `NeedsRescan(scope)` set;
3. queue pressure never drops a terminal task result;
4. cancellation is an atomic task flag and is never queued behind a full result channel;
5. shutdown signals are not queued behind ordinary progress;
6. producers use bounded/nonblocking delivery for transient events;
7. a terminal task stores its result before publishing a transient completion hint;
8. Lua polling always drains retained terminal results before lower-priority progress.

Default delivery limits are configurable under native limits and begin with:

```text
transient_event_capacity = 1024
poll_max_events = 128
poll_max_node_records = 500
```

The implementation may tune defaults through verified benchmarks while preserving boundedness and terminal retention.

## 8. Polling Contract

Lua calls:

```lua
local batch = engine:poll({
  max_events = 128,
  max_node_records = 500,
})
```

`poll()`:

- is synchronous and nonblocking;
- creates Lua values only on the calling thread;
- returns complete event records rather than Lua callbacks;
- obeys count and payload bounds;
- returns the current Engine work summary so Lua can choose its next timer interval;
- consumes terminal results only when they are included in the returned batch;
- leaves undispatched results available for the next call.

Event families include:

```text
TaskProgress
TaskTerminal
ModelChanged
MetadataChanged
NeedsRescan
WatcherDegraded
EngineDiagnostic
```

Every event carries the identities and generations needed by Lua to reject stale work.

## 9. Lua Polling Timer

Lua owns one polling timer for the whole plugin process. Its libuv callback enters scheduled Lua before polling or applying events:

```lua
vim.schedule_wrap(function()
  drain_native_events()
end)
```

The default adaptive cadence is:

```text
active scan/apply/render work: 16 ms
watcher-only sessions:          200 ms
no sessions/tasks/watchers:     timer stopped
```

A drain turn also has a Lua time budget. If a batch or pending render exceeds that budget, Lua schedules another bounded turn rather than completing unbounded work in one callback.

Watcher-only cadence and active cadence are presentation policy. Rust reports work state but does not create or control the timer.

## 10. Mutation Coordination

The Engine owns:

```text
MutationCoordinator = Idle
                    | Running {
                        task_id,
                        sequence: EngineMutationSequence,
                        owner_root_session_id,
                        canonical_affected_paths,
                        cancellation,
                      }
```

`EngineMutationSequence` identifies one mutation for diagnostics and ordering. Each RootSession separately owns a `RootMutationEpoch` used by scanner and prepared-plan staleness checks.

One non-empty mutation may run at a time. Prepare tasks, metadata work, semantic queries, non-intersecting scans, and scanner workers admitted before fence installation may continue concurrently. New intersecting scan admission obeys the mutation-fence protocol below.

At mutation start the coordinator:

1. acquires the Engine mutation gate;
2. allocates one `EngineMutationSequence`;
3. enters the Engine fence-coordination critical section, which serializes active-fence installation, RootSession registration, and every RootSession scan admission;
4. registers the canonical physical-path fence before enumerating RootSessions or issuing the first filesystem-changing syscall;
5. while still in that critical section, increments `RootMutationEpoch` exactly once for every existing RootSession whose canonical root intersects the fence, marks each session fenced, and converts intersecting queued/racing scan requests into coalesced demand rather than tasks;
6. leaves the critical section;
7. requires every later RootSession registration to use the same critical section, intersect its canonical root with the active fence before becoming scan-admissible, initialize its own epoch, start fenced, and queue its scan demand when overlapping;
8. records the owner RootSession's affected NodeId scopes;
9. performs final probes;
10. enters the first filesystem-changing execution step only after every precondition succeeds.

At terminal completion it:

1. publishes every known effect through the owner RootSession coordinator under the already-fixed epoch;
2. marks every live intersecting RootSession's physical overlap dirty;
3. records uncertain scopes and coalesced watcher demand;
4. in the Engine fence-coordination critical section, clears the canonical physical-path fence and converts reconciliation plus current coalesced demand into fresh pending scan demand;
5. leaves the critical section and releases the mutation gate;
6. admits those fresh pending scans through the normal fence check, from each session's current Snapshot generation and unchanged epoch;
7. stores the task terminal result with the reconciliation task identities.

A scan-admission attempt racing with fence installation either starts before the existing session's epoch increment and becomes stale, or observes the fenced state and remains queued. No scan starts inside an intersecting scope while that session is fenced.

`RootMutationEpoch` is not incremented again at terminal completion. Reconciliation therefore captures the final epoch established when the fence was installed.

A mutation task whose originating Lua buffer is destroyed continues under Engine ownership after crossing its commit boundary. Lua buffer lifetime does not own filesystem authority.

## 11. Cancellation

Lua requests cancellation with:

```lua
engine:cancel(task_handle)
```

The bridge sets the task cancellation flag and returns immediately.

Cancellation semantics belong to the task kind:

- scan, metadata, reveal, and prepare tasks stop at their next cooperative checkpoint;
- a finish task waiting for the mutation gate or still in final probes can finish as `CancelledBeforeCommit`;
- a finish task after commit stops at the next Planner-defined safe point and may finish as `CancelledAfterPartialApply`;
- an in-flight filesystem syscall is allowed to return before cancellation is observed;
- an execution group with no safe internal boundary completes the group before honoring the request.

The detailed Executor contract is normative in `04-planning-execution.md`.

## 12. Panic And Error Boundary

Every exported native entrypoint returns a normal Lua result or structured error. Rust unwinding does not cross the Lua C ABI.

Worker panics are caught at the task boundary and stored as:

```text
TaskFailed {
  code = "WorkerPanic",
  task_id,
  task_kind,
}
```

A task panic terminates that task. Engine and unrelated RootSessions remain available when their invariants are intact. A panic while the mutation coordinator is active enters the common terminal finalizer so the mutation gate and publication fence are released and affected scopes are reconciled.

## 13. Shutdown

Engine shutdown is idempotent:

```text
Running -> ShuttingDown -> Shutdown
```

Shutdown:

1. rejects new sessions and tasks;
2. requests cancellation for cancellable tasks;
3. lets committed mutation work reach a safe terminal outcome;
4. closes watcher resources;
5. releases retained terminal results after the final drain or Engine destruction;
6. drops RootSessions when their final strong references disappear;
7. returns `EngineShutdown` from later handle calls.

The native library remains loaded for the lifetime of the Neovim process. Binary upgrades take effect after Neovim restarts.

## 14. Native Module Versioning

The module exports:

```text
api_version()
build_info()
open(config)
```

Lua checks `api_version()` before constructing the Engine. `build_info()` includes:

```text
FRED version
Rust target triple
platform ABI
Lua runtime ABI
build profile
```

Artifact selection and compatibility testing are normative in `06-testing-delivery.md`.

## 15. Architecture Conformance

Architecture conformance requires:

- `cargo test -p fred-core` runs without Neovim or Lua;
- all Neovim operations are observable in Lua modules and headless Lua tests;
- worker threads hold no Lua state or Lua references;
- native events reach Lua only through `poll()`;
- transient delivery is bounded and terminal results remain consumable;
- one host Engine owns one mutation gate;
- the active mutation fence uses canonical physical paths and applies to existing and newly registered overlapping RootSessions;
- every existing intersecting RootSession advances `RootMutationEpoch` once at fence installation;
- a newly registered intersecting RootSession initializes its epoch, starts fenced, and uses that unchanged epoch for post-fence reconciliation;
- scan admission is atomic with fence installation and admits no intersecting worker while fenced;
- stale, released, foreign, consumed, and shutdown handles fail deterministically;
- every active mutation leaves the coordinator through one terminal finalizer.
