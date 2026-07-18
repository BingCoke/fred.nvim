# FRED Filesystem Model

This file is the normative source for native paths, NodeId, RootSession, immutable snapshots, scanning, coverage leases, metadata, watcher hints, snapshot publication, symlink policy, semantic queries, reveal resolution, reconciliation, and filesystem-model lifetime.

## 1. Core Identity Types

The filesystem model uses distinct identity domains:

```text
EngineId
RootSessionId
CoverageLeaseId
SnapshotGeneration
MetadataGeneration
ScanGeneration
RootMutationEpoch
TaskId
NodeId
```

`NodeId` identifies one namespace node inside one RootSession. Its complete logical scope is:

```text
(RootSessionId, NodeId)
```

Lua receives one opaque fixed binary token that already includes the RootSession namespace. Lua compares and stores that token without numeric conversion or parsing.

## 2. Native Path Types

Rust distinguishes:

```text
CanonicalRoot
NativeName
NativeRelativePath
NativeAbsolutePath
```

`NativeRelativePath` is an ordered list of `NativeName` components. Domain requests do not join user-controlled strings into native paths.

A `NativeName` preserves platform-native units:

- Unix uses filename bytes;
- Windows uses UTF-16 code units;
- path display never becomes filesystem identity;
- root-relative path transport crosses Lua as component lists.

The root capability owns canonical resolution and platform namespace behavior. Domain validation rejects:

- empty components;
- `.` and `..` components;
- platform separators inside a component;
- NUL;
- Windows reserved component forms;
- any operation whose resolved location escapes the RootSession root;
- any required directory traversal through a symlink/reparse leaf.

Destination collision checks use the actual destination filesystem's behavior. Display spelling remains unchanged.

## 3. RootSession

A `RootSession` represents one canonical filesystem namespace root:

```text
RootSession {
  id: RootSessionId,
  canonical_root: CanonicalRoot,
  root_node_id: NodeId,
  current_snapshot: Arc<RootSnapshot>,
  metadata_snapshot: Arc<MetadataSnapshot>,
  node_id_allocator,
  snapshot_generation,
  metadata_generation,
  root_mutation_epoch,
  coverage_leases,
  scan_scheduler,
  watcher_state,
  dirty_scopes,
  task_references,
}
```

A direct root open reuses an existing RootSession only when canonical root identity and core namespace configuration are compatible. Presentation configuration does not participate in RootSession identity.

Compatible shared state includes:

- canonical root and platform namespace rules;
- symlink traversal policy;
- watcher backend policy;
- native path normalization policy.

Each Instance still owns independent Lua presentation, baseline, draft, expansion, sort, filter, and cursor state.

Explicit child/navigate lineage may use a descendant directory NodeId as its Lua Instance root while continuing to share the parent RootSession. An independent direct open of a nested absolute root creates or reuses that nested root's exact RootSession.

RootSession registration and scan admission use the Engine fence-coordination critical section. Before a session becomes scan-admissible, registration consults the active canonical physical-path fence. A newly registered overlapping session initializes its first `RootMutationEpoch` value, starts fenced, queues initial coverage demand without starting a scanner, and keeps that epoch through post-fence reconciliation.

## 4. Node Model

```text
NodeObservation = Confirmed
                | RetainedUnverified {
                    source_generation,
                    reason,
                  }

NodeKind = File
         | Directory
         | Symlink
         | Other

SnapshotNode {
  id: NodeId,
  parent_id: Option<NodeId>,
  name: NativeName,
  relative_path: NativeRelativePath,
  kind: NodeKind,
  observation: NodeObservation,
  symlink_info: Option<SymlinkInfo>,
}
```

The RootSession root is a directory node whose `parent_id` is absent. It is not rendered as an ordinary projection row.

A Snapshot enforces:

- each non-root node has exactly one directory parent;
- each parent child-list agrees with every child's `parent_id`;
- each relative path agrees with its parent path and native name;
- each path maps to at most one NodeId under the destination namespace rules;
- symlink and reparse leaf nodes have no child list;
- all indexes are built and published together.

## 5. NodeId Lifetime

NodeId rules are:

| Filesystem/model event | Identity result |
|---|---|
| ordinary refresh confirms the same parent/name/kind lineage | retain NodeId |
| FRED rename or same-filesystem move succeeds | retain source NodeId at destination |
| FRED create succeeds | retain the provisional destination NodeId |
| FRED copy succeeds | retain the provisional destination NodeId |
| fully successful cross-filesystem move | retain source NodeId at destination |
| cross-filesystem copy succeeds and source deletion does not complete | source and destination retain distinct NodeIds |
| external rename | old NodeId disappears and a new NodeId is allocated |
| authoritative external delete followed by later create | allocate a new NodeId |
| same-path, same-kind refresh without an observed authoritative absence | retain NodeId |
| hard-linked directory entries | assign distinct NodeIds |

FRED does not use inode, file ID, content hash, size, or timestamp as public namespace identity.

## 6. Immutable RootSnapshot

```text
DirectoryCoverage = Unscanned
                  | Complete { observed_at }
                  | Partial { reason, observed_at }
                  | Failed { error, observed_at }
                  | Dirty { previous }

RootSnapshot {
  root_session_id: RootSessionId,
  generation: SnapshotGeneration,
  root_node_id: NodeId,
  nodes_by_id: Map<NodeId, SnapshotNode>,
  children_by_parent: Map<NodeId, Arc<[NodeId]>>,
  node_id_by_path: Map<NativeRelativePath, NodeId>,
  directory_coverage: Map<NodeId, DirectoryCoverage>,
}
```

A published Snapshot is immutable. Readers hold `Arc<RootSnapshot>` through a `SnapshotHandle`. A worker never mutates `current_snapshot` in place.

Directory authority rules:

- `Complete` replaces the prior direct-child set and may prove absence;
- `Partial` and `Failed` merge newly confirmed observations while retaining previously known unobserved children as `RetainedUnverified`;
- `Unscanned` does not authorize any absence conclusion;
- `Dirty` preserves the prior model until a later scan publishes a terminal observation;
- a retained unverified node remains queryable and may be rendered with degraded status;
- only authoritative model evidence or a known successful FRED operation removes a node.

## 7. Coverage Leases

Lua presentation consumers request filesystem coverage through leases:

```text
CoverageLeaseRequest {
  scopes: [
    {
      directory_node_id,
      depth,
      traversal_ignore_rules,
    }
  ],
  required_metadata_fields,
  limits,
}
```

A RootSession's requested coverage is the union of active leases and temporary domain tasks.

Rules:

- an Instance updates its lease when baseline depth, expansion, explicit include, reveal, or ignore traversal policy changes;
- hidden-file visibility and sorting do not themselves add filesystem coverage;
- a path is traversed when at least one active lease requests it under that lease's traversal rules;
- the RootSession may scan a superset needed by several leases;
- Lua applies each Instance's presentation ignore policy independently to queried nodes;
- releasing a lease removes future demand and watcher references;
- releasing coverage does not immediately delete previously observed nodes from the Snapshot cache;
- a temporary reveal lease is released when reveal resolves, fails, or is cancelled;
- metadata requirements are the union of active lease requirements.

Default per-request limits begin with:

```text
max_entries = 50000
max_directories = 10000
```

A limit produces `Partial` coverage for the affected scope.

## 8. Scanner

The scanner uses explicit directory traversal built around native directory reads and an `ignore`-crate matcher for gitignore-like traversal rules. It controls cancellation, scope depth, NodeId reuse, metadata demand, directory status, and publication generations directly.

A scan task records:

```text
ScanTask {
  task_id,
  scan_generation,
  base_snapshot_generation,
  base_root_mutation_epoch,
  requested_scopes,
  cancellation,
}
```

The worker builds one detached candidate:

```text
ScanCandidate {
  scan_generation,
  base_snapshot_generation,
  base_root_mutation_epoch,
  requested_scope_results: Map<Scope, Complete | Partial | Failed>,
  candidate_contents,
}
```

The worker may report bounded progress, but candidate namespace nodes remain Rust-owned until the RootSession coordinator accepts the candidate.

The scanner:

- allocates or reuses RootSession-scoped NodeIds;
- uses `symlink_metadata` semantics for entry kind;
- never recursively traverses symlink/reparse leaves;
- records each requested directory as Complete, Partial, or Failed;
- checks cancellation between directory entries and scopes;
- stops at configured limits;
- constructs every Snapshot index consistently before handoff;
- emits one terminal candidate for one scan generation.

## 9. Snapshot Publication Coordinator

Only the RootSession coordinator may publish a candidate Snapshot.

A candidate may publish only when all of the following hold:

```text
candidate.scan_generation is current
candidate.base_snapshot_generation == current_snapshot.generation
candidate.base_root_mutation_epoch == root_session.root_mutation_epoch
the Engine canonical physical-path fence does not intersect the candidate scopes
candidate.requested_scope_results contains one terminal result for every requested scope
```

Publication is one atomic coordinator operation:

```text
new_generation = current_snapshot.generation + 1
published_snapshot = RootSnapshot {
  generation = new_generation,
  ...candidate_contents
}
current_snapshot = Arc::new(published_snapshot)
snapshot_generation = new_generation
emit ModelChanged { generation = new_generation, ... }
```

The Snapshot, RootSession counter, `ModelChanged` event, and terminal task result therefore expose the same generation.

### 9.1 Stale Candidate Handling

When a candidate fails a publication check:

1. its detached candidate data is dropped;
2. its still-requested scopes remain or become dirty;
3. the scheduler coalesces those scopes with newer demand;
4. a new scan starts from the current Snapshot generation and `RootMutationEpoch`.

Version one drops the stale candidate and restarts the requested scopes from the current Snapshot generation and epoch.

### 9.2 Mutation Interaction

Before a mutation changes the filesystem, the Engine uses one fence-coordination critical section to serialize active-fence installation, RootSession registration, and every RootSession scan admission. It registers the canonical physical-path fence before enumerating existing sessions.

For an existing intersecting RootSession:

- while the coordination section remains held, fence installation advances `root_mutation_epoch` exactly once;
- the session becomes fenced before another scan admission can pass;
- intersecting queued or racing requests become coalesced demand rather than scanner tasks;
- no new intersecting scanner worker starts while fenced;
- a scanner already admitted before installation carries the old epoch and becomes stale.

A RootSession registered during the mutation enters the same coordination section before it becomes scan-admissible. When overlapping, it initializes its first epoch value, starts fenced, and queues initial coverage demand without starting a scanner.

After execution reaches a terminal result:

- known successful effects publish first through the owner RootSession coordinator using the fixed owner epoch;
- every live intersecting RootSession records its physical overlap as dirty;
- uncertain owner scopes retain degraded authority until reconciliation;
- in the fence-coordination critical section, the Engine clears the physical fence and converts reconciliation plus current coalesced demand into fresh pending scan demand;
- after leaving that section and releasing the mutation gate, those fresh scans seek admission through the normal fence check from each session's current published generation and unchanged epoch;
- scanner candidates admitted before fence installation remain stale and cannot restore the pre-mutation model.

This admission ordering prevents a scanner from observing an intermediate mutation state and publishing it after fence clearance.

## 10. Metadata Snapshot

Namespace identity and optional metadata have separate revisions:

```text
MetadataRecord {
  node_id,
  observed_path,
  permissions,
  size,
  mtime,
  ctime,
  atime,
  birthtime,
}

MetadataSnapshot {
  generation: MetadataGeneration,
  records,
}
```

A metadata result is accepted only when:

- its metadata task generation is current;
- its source Snapshot generation is still valid for that request;
- the NodeId still exists;
- the NodeId's current native path matches `observed_path`;
- at least one active lease still requires the field.

Metadata failure produces an unavailable value and diagnostic. It does not remove a namespace node or change namespace generation.

## 11. Watchers

FRED uses `notify` with basic debouncing suitable for coalescing filesystem-change hints. Native backends are selected by platform, with polling available when native watching is unavailable or unreliable.

Watcher events flow as:

```text
platform event
  -> normalize affected directory scope
  -> debounce/coalesce
  -> mark scope dirty
  -> schedule scoped scan
```

Rules:

- watcher events never publish nodes directly;
- watcher rename events do not transfer NodeId identity;
- queue overflow becomes `NeedsRescan(scope)`;
- backend rescan indications become `NeedsRescan(scope)`;
- an unrecoverable watcher error marks watcher state degraded and keeps manual refresh available;
- events caused by FRED mutations coalesce with the mutation's reconciliation scopes;
- watcher coverage follows active coverage leases.

## 12. Symlink And Reparse Policy

Version one uses:

```text
follow_links = false
```

Node classification describes the directory entry itself:

- Unix symlinks are `Symlink` leaves;
- Windows junctions and directory-like reparse points are symlink/reparse leaves;
- broken symlinks remain valid leaf nodes;
- optional target metadata may report `target_kind` for display;
- target kind does not add child membership;
- reveal may select the symlink node itself;
- path resolution does not traverse a symlink as an intermediate directory.

Opening a directory target explicitly creates or reuses the target's exact RootSession. Target children are not mounted beneath the symlink NodeId.

## 13. Semantic Snapshot Queries

Lua renders from a fixed `SnapshotHandle`:

```text
snapshot = engine.acquire_snapshot(root_session)
```

The handle supports read-only paged queries:

```text
snapshot.generation()
snapshot.query_node(node_id)
snapshot.query_children(node_id, cursor, max_records) -> NodePage
snapshot.query_ancestors(node_id, cursor, max_records) -> NodePage
snapshot.resolve_relative_components(components)
snapshot.coverage_status(node_id)
```

```text
NodePage {
  records: [NodeRecord],
  next_cursor: Option<QueryCursor>,
}
```

`max_records` may not exceed the Engine's configured snapshot-query record budget. Every page from one handle reads the same immutable Snapshot, and a cursor is valid only for that handle and query. Lua can therefore render a large directory across bounded scheduled slices without combining generations.

Query results contain semantic filesystem data:

```text
NodeRecord {
  node_id,
  parent_id,
  native_name,
  kind,
  observation,
  coverage,
  requested_metadata,
}
```

They contain no row number, indentation, icon, highlight, cursor, extmark, window, changedtick, or buffer text.

## 14. Refresh And Reconciliation

Manual refresh creates a scoped scan request:

```text
start_refresh {
  root_session,
  scopes,
}
```

Reconciliation is the same model operation with a required cause and affected scope set:

```text
ReconciliationRequest {
  root_session,
  caused_by_task,
  affected_scopes,
  priority,
}
```

Reconciliation runs after:

- successful mutation;
- partial failure;
- post-commit cancellation;
- watcher overflow;
- a syscall outcome whose exact namespace effects are uncertain;
- stale candidate rejection when active demand remains.

Known effects may publish before reconciliation. Reconciliation observes the actual filesystem and replaces degraded scope status when it reaches a terminal result.

## 15. Reveal Resolution

Rust resolves filesystem identity; Lua performs presentation.

Input is either:

```text
RevealByNodeId {
  root_session,
  node_id,
}
```

or:

```text
RevealByPath {
  root_session,
  relative_components: [NativeName],
}
```

When required ancestors are unscanned, the reveal task creates temporary precise coverage and scans component by component.

Terminal results are:

```text
RevealResolved {
  snapshot_generation,
  target_node_id,
  ancestor_node_ids,
}

RevealNotFound {
  snapshot_generation,
  nearest_existing_node_id,
  missing_component_index,
}

RevealFailed {
  code,
  component_index,
  os_error,
}

RevealCancelled
```

Resolution validates that:

- every component stays inside the RootSession;
- every intermediate component is a directory node;
- no intermediate component is a symlink/reparse leaf;
- the returned ancestor chain belongs to one immutable Snapshot generation.

Lua expansion, forced visibility, render, window selection, and cursor movement are normative in `03-lua-presentation.md`.

## 16. RootSession Lifetime And Cache

The RootSession registry holds weak references. Active coverage leases, Snapshot handles, and tasks hold strong references.

When the final presentation consumer releases its lease:

- future watcher and scan demand is removed;
- tasks with no remaining consumer are cancelled;
- active committed mutation work retains the RootSession until terminal completion;
- the RootSession drops after its final strong reference is released.

Per-scan limits bound one requested generation. A long-lived RootSession may retain nodes and metadata from scopes visited by earlier generations. Full cache release occurs when the RootSession itself is released.

## 17. Filesystem-Model Conformance

Conformance requires:

- every published Snapshot satisfies all indexes and parent/child invariants;
- one scan generation publishes at most one Snapshot;
- Snapshot contents, RootSession counter, model event, task result, and SnapshotHandle report the same generation;
- Complete coverage alone authorizes absence from directory membership;
- stale or mutation-overlapping scan candidates never publish;
- rejected candidates cause requested scopes to be rescanned;
- canonical physical fences cover existing and newly registered overlapping RootSessions;
- every existing intersecting RootSession advances `RootMutationEpoch` once at fence installation;
- a newly registered intersecting RootSession initializes its epoch, starts fenced, and uses that unchanged epoch for reconciliation;
- scan admission is atomic with fence installation and starts no intersecting worker while fenced;
- watcher events only create dirty/rescan demand;
- external rename does not transfer NodeId;
- known FRED move effects preserve NodeId according to the operation result;
- symlink/reparse leaves never gain child membership;
- one SnapshotHandle returns one generation across every paged query;
- reveal returns filesystem identities and ancestors without presentation state;
- final RootSession release drops snapshots, metadata, watchers, and retained cache through ordinary ownership.
