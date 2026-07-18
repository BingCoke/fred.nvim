# FRED Planning And Execution

This file is the normative source for semantic intents, preparation, direct probes, Planner output, one-shot `PreparedApply`, preview, mutation authority, operation ordering, Executor behavior, cooperative cancellation, known-effect publication, partial outcomes, reconciliation, and mutation errors.

## 1. Apply Ownership

Lua owns edit detection, draft state, preview UI, row diagnostics, and Conflict presentation. Rust receives semantic filesystem intents and owns validation, planning, authority, execution, model publication, and reconciliation.

The apply protocol has two native tasks:

```text
start_apply_prepare(request) -> TaskHandle
start_apply_finish(prepared) -> TaskHandle
```

Prepare performs no filesystem mutation. Finish consumes one prepared capability and is the only buffer-driven path into the Executor.

## 2. Semantic Intent Request

Lua submits:

```text
PrepareRequest {
  root_session: RootSessionHandle,
  current_root_node_id: NodeId,
  base_snapshot_generation: SnapshotGeneration,
  intents: [SemanticIntent],
  policy: ApplyPolicy,
}
```

`SemanticIntent` is one of:

```text
Create {
  intent_id,
  provisional_node_id,
  parent_node_id,
  native_name,
  kind: File | Directory,
}

Relocate {
  intent_id,
  source_node_id,
  destination_parent_node_id,
  destination_name,
  provisional_destination_node_id: Option<NodeId>,
}

Copy {
  intent_id,
  source_node_id,
  provisional_destination_node_id,
  destination_parent_node_id,
  destination_name,
}

Delete {
  intent_id,
  node_id,
}
```

The node kind in the current Snapshot determines whether a logical operation is file, directory, symlink, or other. A directory Copy/Delete becomes `COPY_TREE`/`DELETE_TREE` in the logical plan.

Lua changedtick and row positions remain Lua state. `intent_id` is an opaque request-local diagnostic key.

## 3. Prepare Task

Prepare executes these phases:

```text
request validation
  -> Snapshot and NodeId resolution
  -> semantic validation
  -> immutable direct probes
  -> pure Planner
  -> PreparedApplyCore + PlanPreview
```

### 3.1 Request Validation

Prepare verifies:

- every handle belongs to the current Engine and is live;
- the RootSession and current root NodeId exist;
- `base_snapshot_generation` is the current Snapshot generation;
- every NodeId belongs to the RootSession;
- provisional NodeIds are active RootSession reservations and occur at most once in this request;
- `intent_id` values are unique inside the request;
- no source NodeId is consumed by incompatible intents;
- every native name is one valid component;
- every destination parent resolves inside the current Instance root authority;
- existing node kind is not forged by the request.

A provisional reservation survives prepare diagnostics, stale prepare results, and preview cancellation. A later request may reuse it for the same pending Lua row. Successful known-effect publication promotes or retires it; explicit Lua release removes it when the pending row, draft, navigation context, or Instance is discarded. “Once” means once per request and at most once as an installed namespace identity.

### 3.2 Semantic Validation

Validation detects:

- duplicate final destinations;
- two operations claiming the same provisional destination;
- source and destination equivalence where the operation would be meaningless;
- directory move/copy to itself or a descendant;
- deleting a parent while retaining an un-evacuated requested destination beneath it;
- parent cycles;
- conflicting create/delete/relocate combinations;
- replacement requests that do not explicitly remove the current destination identity;
- namespace case/collision conflicts under the destination filesystem rules.

### 3.3 Direct Probes

Prepare captures immutable probe results for:

- every source path and kind;
- every destination path and occupancy;
- every existing destination parent;
- every planned parent chain;
- the nearest existing ancestor of a planned parent chain;
- symlink/reparse traversal at each required directory boundary;
- platform capability facts required by the plan.

A planned parent may be absent when the same plan creates it earlier. An occupied planned-parent path is valid only when the plan explicitly moves or removes its current identity before creation.

Probe failures are diagnostics and produce no `PreparedApply`.

## 4. Pure Planner

Planner receives only immutable domain data:

```text
base Snapshot
current Snapshot
current root NodeId
semantic intents
initial probe results
platform namespace rules
ApplyPolicy
```

Planner performs no filesystem I/O and mutates no Engine or RootSession state.

It returns:

```text
ValidPlan {
  authority,
  logical_operations,
  execution_groups,
  identity_dispositions,
  reservation_dispositions,
  owner_affected_scopes,
  canonical_affected_paths,
  final_probe_expectations,
  intent_digest,
}

PlanPreview {
  operations,
  warnings,
}
```

or structured non-confirmable diagnostics.

## 5. Plan Authority

A valid plan carries:

```text
PlanAuthority {
  engine_id,
  root_session_id,
  current_root_node_id,
  base_snapshot_generation,
  base_root_mutation_epoch,
  intent_digest,
  policy_digest,
  capability_nonce,
}
```

`PreparedApplyCore` owns the immutable `ValidPlan` and one state cell:

```text
Fresh -> Consumed
```

`fred-lua` wraps it as `PreparedApplyHandle`.

Rules:

- the handle belongs to one Engine and RootSession;
- preview data cannot mutate the plan;
- dropping a Fresh handle releases its plan memory but leaves Lua-owned provisional reservations available for retry;
- finish consumes Fresh exactly once;
- duplicate, foreign, released, or shutdown handles fail deterministically;
- a changed Snapshot generation or RootSession mutation epoch makes the plan stale before fence installation;
- a stale plan returns before filesystem mutation.

## 6. Logical Preview

A non-empty preview contains semantic logical operations:

```text
CREATE
COPY
COPY_TREE
MOVE
DELETE
DELETE_TREE
REPLACE
LINK
```

Each preview row contains:

```text
intent_id
operation kind
source identity/path when present
destination identity/path when present
node kind
warnings
```

Internal temporary names, low-level execution steps, capability nonces, probe records, and platform handles are not preview fields.

Confirmation is for the complete immutable plan. A changed draft requires a new prepare request.

A valid empty plan still returns a one-shot prepared capability so finish can revalidate authority and produce a deterministic Empty result.

## 7. Finish Task And Commit Boundary

Lua starts finish after confirming the preview and rechecking its own attempt and changedtick:

```text
start_apply_finish(prepared_apply)
```

Finish performs:

```text
consume PreparedApply
  -> wait for Engine mutation gate
  -> authority validation
  -> final probes
  -> commit boundary
  -> serial Executor
  -> known-effect publication
  -> reconciliation scheduling
  -> terminal result retention
```

The commit boundary is the point immediately before the first filesystem-changing syscall.

Before that boundary:

- cancellation yields `CancelledBeforeCommit`;
- stale authority yields `StalePreparedApply`;
- final-probe mismatch yields `PreconditionChanged`;
- no filesystem effect has occurred;
- Lua preserves its draft.

After that boundary, cancellation is cooperative and may produce a partial filesystem result.

## 8. Mutation Gate

Finish tasks wait in an Engine-owned FIFO mutation queue. The queue is bounded by active task limits. A waiting task reports progress state `WaitingForMutationGate` and can be cancelled.

When a task acquires the gate, the mutation coordinator:

1. verifies the prepared Engine and RootSession handles;
2. verifies the current Snapshot generation and `RootMutationEpoch` equal the plan's base authority;
3. enters the Engine fence-coordination critical section, which serializes active-fence installation, RootSession registration, and every RootSession scan admission;
4. registers `ValidPlan.canonical_affected_paths` as the Engine physical-path fence before enumerating existing RootSessions;
5. while still in that critical section, advances `RootMutationEpoch` exactly once for every existing intersecting RootSession, including the owner, marks each session fenced, and converts intersecting queued/racing scan requests into coalesced demand rather than tasks;
6. leaves the critical section;
7. requires every RootSession registered during the mutation to use the same section, initialize its own epoch, start fenced, and queue initial scan demand when overlapping;
8. records the owner RootSession's NodeId-based affected scopes;
9. repeats final probes under the installed fence;
10. checks cancellation;
11. enters the commit boundary;
12. invokes Executor.

Fence installation atomically validates the owner's base epoch and advances it to the execution epoch. Final probes and known-effect publication use that execution epoch; terminal completion does not advance it again.

Every outcome exits through one finalizer that publishes known effects, records dirty physical overlaps, uses the fence-coordination critical section to clear the active physical fence and create fresh reconciliation/current-demand scan requests, releases the gate, admits those requests through the normal current-fence check from each live intersecting RootSession's fixed epoch, and stores one terminal result. A scan-admission race either starts before an existing session's epoch increment and becomes stale or observes the fenced state and remains coalesced demand.

## 9. Final Preflight

Final preflight repeats every mutable namespace assumption required by `ValidPlan`:

- source still exists;
- source kind is compatible;
- destination occupancy matches the plan;
- existing parents remain directories;
- planned-parent paths remain available in the expected order;
- nearest existing ancestors remain directories;
- directory traversal still contains no symlink/reparse leaf;
- temporary rename names remain available;
- root and operation paths remain inside the RootSession authority.

A mismatch returns `PreconditionChanged` without crossing the commit boundary.

Destination installation still uses exclusive/no-replace behavior, so a race after final preflight fails the affected execution step rather than overwriting an entry.

## 10. Logical Operation Semantics

### 10.1 Create

A file create uses exclusive creation. A directory create uses an exclusive directory operation. Missing parent directories must already exist or be earlier planned directory creates.

### 10.2 Relocate

A same-filesystem relocate uses the platform rename primitive with no-replace destination semantics. Rename within one parent is a rename; a changed parent is a move. Both retain the source NodeId after success.

### 10.3 Copy

A file copy creates the final destination exclusively and copies data incrementally. A directory copy is one logical `COPY_TREE` whose Executor traverses the tree incrementally.

The logical plan does not contain one preview row per descendant. Internal traversal records known effects and safe points.

### 10.4 Delete

A file or symlink delete removes that entry. A directory delete is one logical `DELETE_TREE` executed incrementally in post-order:

```text
children first
parent last
```

### 10.5 Replacement

Replacement is valid only when semantic intents explicitly remove the existing destination identity and install another identity at that path. Planner orders removal and installation according to dependencies and exposes one logical preview grouping.

### 10.6 Immediate Link

The immediate symlink helper builds one internal `LINK` plan and enters the same mutation gate, preflight, exclusive-destination, publication, and reconciliation protocol.

## 11. Directory Operation Ordering

Planner produces a linear sequence of execution groups that satisfies:

1. planned parent creation precedes descendant installation;
2. an existing planned-parent occupant is moved or removed before directory creation;
3. all copies from a source precede deletion of that source;
4. a descendant moving outside a directory subtree is evacuated before parent move/delete;
5. a parent directory move precedes descendant edits that remain inside the moved subtree;
6. child deletion precedes parent directory deletion;
7. cross-filesystem destination copy completes before source deletion begins;
8. replacement target removal precedes installation;
9. irreversible deletes occur as late as dependencies permit.

A directory rename/move remains one logical operation. Unedited descendants are rebased rather than emitted as redundant logical moves.

## 12. Rename Cycles And Safe Groups

Swaps, case-only renames, and longer cycles may use unique same-directory temporary names.

Example:

```text
A -> temporary
B -> A
temporary -> B
```

Planner wraps these steps in one execution group whose next cancellation safe point follows group completion. A cancellation request during the group becomes `CancelRequested`; Executor finishes the group unless a syscall fails, then stops before the next group.

Temporary names are unique under destination namespace rules and are not shown in logical preview.

## 13. Cross-Filesystem Move

When same-filesystem rename reports a cross-filesystem condition, Executor performs:

```text
copy source -> exclusive final destination
then
remove source
```

For directories this uses incremental `COPY_TREE` followed by incremental `DELETE_TREE`.

Identity outcomes are:

- copy and source deletion both complete: source NodeId moves to destination;
- copy completes and source deletion does not complete: source and destination remain distinct identities;
- a provenance destination uses its provisional NodeId for the surviving destination when identities split;
- a direct relocate without a provisional destination receives a newly allocated destination NodeId when identities split;
- cancellation first observed after source deletion completes classifies the move as Success;
- partial destination nodes discovered by reconciliation receive destination identities rather than source identities unless a provisional identity already represents them.

## 14. Executor Interface

Executor receives:

```text
execution_groups
cancellation token
FsOps implementation
```

Executor does not perform semantic intent detection or preview construction.

The internal filesystem seam is:

```text
trait FsOps {
  symlink_metadata(...)
  read_dir(...)
  create_file_exclusive(...)
  create_dir(...)
  copy_read(...)
  copy_write(...)
  rename_no_replace(...)
  remove_file(...)
  remove_dir(...)
  create_symlink_exclusive(...)
}
```

Production uses `RealFs`. Tests use `FaultInjectingFs`.

## 15. Execution And Cancellation

Executor runs one execution group at a time.

Rules:

1. check cancellation before each group;
2. check cancellation at each Planner-defined internal safe point;
3. record every successful syscall and known namespace effect;
4. stop on the first syscall error;
5. stop at the next safe point after an accepted cancellation;
6. run no later group after failure or accepted cancellation;
7. classify cancellation observed only after the final group completes as Success.

### 15.1 Incremental Delete

Large directory deletion uses an explicit traversal stack. Executor checks cancellation before each entry removal and after each successful removal. It does not issue one whole-tree removal call that prevents cooperative stopping.

A cancellation may leave some children deleted and the remaining tree present. The execution report records exact known deletions and reconciliation observes the rest.

### 15.2 Incremental Copy

Large file copy uses bounded chunks. Directory copy checks cancellation between entries, directories, and file chunks.

A cancelled or failed copy may leave a partial exclusive destination. The report records known-created paths and reconciliation determines the actual final state.

### 15.3 In-Flight Syscalls

A cancellation request does not forcibly terminate the worker thread or an active syscall. The syscall returns normally, its result is recorded, and cancellation is honored at the next safe point.

## 16. Execution Report

Executor returns:

```text
ExecutionReport {
  group_results,
  known_effects,
  failed_operation: Option<OperationId>,
  cancelled_operation: Option<OperationId>,
  unstarted_operations,
  uncertain_scopes,
}
```

Group states are:

```text
Completed
Failed
Cancelled
Unstarted
```

At most one group is Failed or Cancelled. A compound group may include successful internal effects before that terminal state.

## 17. Terminal Apply Results

Finish returns one terminal result:

```text
Empty {
  snapshot_generation,
}

Succeeded {
  snapshot_generation,
  node_id_transitions,
  reservation_dispositions,
  execution_report,
  reconciliation_tasks,
}

CancelledBeforeCommit

FailedBeforeCommit {
  code,
  diagnostics,
}

ExecutionFailed {
  commit_crossed = true,
  snapshot_generation,
  node_id_transitions,
  reservation_dispositions,
  execution_report,
  reconciliation_tasks,
}

CancelledAfterPartialApply {
  commit_crossed = true,
  snapshot_generation,
  node_id_transitions,
  reservation_dispositions,
  execution_report,
  reconciliation_tasks,
}
```

Post-commit terminal results remain retained until Lua consumes them. Every `ExecutionFailed` and `CancelledAfterPartialApply` result requires reconciled Conflict handling even when the report contains no known successful effect.


## 18. Known-Effect Publication

After Executor stops, the mutation coordinator combines:

```text
ValidPlan identity dispositions
+ ExecutionReport known effects
+ current RootSnapshot
```

The owner RootSession coordinator publishes one self-consistent immutable Snapshot containing every authoritative known effect. It allocates `new_generation = current + 1`, stamps the Snapshot with that generation, atomically swaps it into `current_snapshot`, updates the RootSession counter, and emits the same generation in model events and the terminal result.

Examples:

- successful create installs its provisional NodeId and marks the reservation Promoted;
- successful copy installs its provisional NodeId and marks the reservation Promoted;
- successful relocate places the source NodeId at destination;
- successful provenance-derived relocate retires the provisional destination, marks it `RetiredTo(source_node_id)`, and emits one `NodeIdTransition`;
- an unstarted provisional destination remains Reserved for Conflict retry;
- an abandoned provisional with no remaining draft owner is Released;
- cross-filesystem identity split retains source and destination identities separately;
- confirmed deletions remove affected NodeIds;
- uncertain operation scopes become `Partial`/`Dirty` and retain previously known unverified members until reconciliation.

Known-effect Snapshot publication occurs before the mutation gate is released. The Snapshot, RootSession counter, model event, and terminal result name the same generation.

## 19. Reconciliation

Every non-empty mutation schedules scoped reconciliation after the canonical physical fence is cleared. Partial outcomes give reconciliation elevated priority.

Reconciliation:

- scans the owner RootSession's Planner-owned NodeId scopes;
- scans physical overlaps for every other live intersecting RootSession;
- incorporates watcher hints received during execution;
- captures the already-fixed `RootMutationEpoch` for each session;
- replaces degraded scope status when terminal observations are available;
- discovers partial copy destinations, remaining delete-tree members, temporary rename names, and external concurrent changes;
- emits `ModelChanged` for every accepted Snapshot generation.

Lua successful rendering waits for the owner reconciliation task named in the result. Other intersecting Instances receive their own RootSession model events. Lua post-commit failure/cancellation handling enters Conflict with the reconciled owner Snapshot and retained draft.

## 20. Error Model

Stable preparation/final-preflight errors include:

```text
InvalidIntent
StaleGeneration
StalePreparedApply
UnknownNode
ForeignNode
IllegalNativeName
OutsideRoot
DuplicateDestination
ConflictingIntents
ParentCycle
DestinationExists
MissingSource
WrongSourceKind
MissingParent
ParentNotDirectory
SymlinkTraversalDisabled
PermissionDenied
FilesystemProbeFailed
PreconditionChanged
EngineShutdown
```

Execution errors include:

```text
CreateFailed
CopyFailed
RenameFailed
DeleteFailed
LinkFailed
ExecutionCancelled
WorkerPanic
```

Errors may carry:

```text
intent_id
node_id
operation_id
native path components
retryable
OS error code/message
```

Lua maps domain identities to rows, preview entries, notifications, and Conflict decorations.

## 21. Fault Injection

`FaultInjectingFs` supports deterministic tests such as:

```text
fail the third rename
fail remove_file for one native path
pause a syscall at a barrier
create a destination after final probes
cancel after N delete effects
fail source deletion after cross-filesystem copy
panic inside a worker step
```

Fault injection verifies gate release, known-effect publication, terminal-result retention, cancellation safe points, reconciliation scopes, and NodeId outcomes.

## 22. Planning And Execution Conformance

Conformance requires:

- Lua submits semantic intents with unique `intent_id` values;
- provisional reservations survive pre-commit retry and are promoted, retired, retained, or released explicitly;
- prepare performs no filesystem-changing syscall;
- Planner is deterministic and pure for identical inputs;
- every valid plan returns one sealed one-shot `PreparedApply`;
- finish consumes that capability once;
- final preflight repeats every mutable namespace assumption;
- one Engine mutation gate serializes mutations;
- the active fence is expressed as canonical physical paths and covers existing and newly registered RootSessions;
- every existing intersecting RootSession advances `RootMutationEpoch` once at fence installation;
- a newly registered intersecting RootSession initializes its epoch, starts fenced, and uses that unchanged epoch for reconciliation;
- scan admission is atomic with fence installation and starts no intersecting worker while fenced;
- destination installation is exclusive;
- execution follows the Planner's linear group order;
- cancellation before commit has no filesystem effect;
- cancellation after commit stops at the next safe point;
- large copy/delete work exposes frequent safe points;
- rename-cycle groups complete to their next safe boundary;
- no later group runs after failure or accepted cancellation;
- known effects publish with one generation before terminal observation;
- every non-empty mutation schedules reconciliation for all live intersecting RootSessions;
- Empty completes without a reconciliation task;
- every post-commit failure/cancellation preserves the Lua draft and enters Conflict through the presentation protocol.
