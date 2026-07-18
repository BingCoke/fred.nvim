# FRED Lua Presentation And Editable Projection

This file is the normative source for Neovim ownership, flat editable path syntax, Lua path encoding, projection, extmark identity, committed baselines, semantic intent detection, provenance, BufferRuntime, write routing, asynchronous apply UX, Conflict state, multi-buffer synchronization, and reveal presentation.

## 1. Lua State Owners

Each public Instance owns one Lua presentation stack:

```text
Instance
  BufferRuntime
  ViewProjection
  PresentationState
  CoverageLeaseHandle
  RootSessionHandle
```

### 1.1 `ViewProjection`

```text
ViewProjection {
  current_root_node_id,
  snapshot_generation,
  baseline_entries,
  pending_provisionals,
  rendered_node_ids,
  node_to_extmark,
  extmark_to_node,
  expansion_by_node,
  baseline_depth,
  explicit_includes,
  forced_visible_nodes,
}
```

### 1.2 `PresentationState`

```text
PresentationState {
  hidden_files_visible,
  ignore_rules,
  sort,
  columns,
  metadata_cache,
  frozen_dirty_sibling_order,
}
```

### 1.3 `BufferRuntime`

```text
BufferRuntime {
  bufnr,
  extmark_namespace,
  root_session_handle,
  coverage_lease_handle,
  state,
  attempt_id,
  attempt_origin: Editing | Conflict,
  captured_draft,
  pending_task,
  conflict_state,
  model_dirty,
  internal_sync_depth,
}
```

Lua is the sole owner of these objects. Rust receives semantic filesystem requests rather than presentation objects.

## 2. Version-One Flat Projection

Every editable line is one complete path relative to the Instance's current root:

```text
README.md
src/
src/init.lua
src/lib/
src/lib/parser.lua
tests/
```

The projection has these properties:

- directories end with `/`;
- files, symlinks, and other entries do not end with `/`;
- `/` is the logical component separator on every platform;
- the current root itself has no editable row;
- sibling sorting happens within each directory;
- traversal is depth-first, keeping materialized directory subtrees contiguous;
- moving rows within the same parent does not express filesystem order;
- icons, metadata, diagnostics, and operation markers are extmark decorations;
- NodeId is never editable text.

## 3. Canonical Lua Path Codec

Lua converts between native-name component transport and canonical buffer text.

### 3.1 Component Escapes

The canonical escapes are:

```text
\\          literal backslash
\t          tab
\n          newline
\r          carriage return
\xNN        one native byte, with uppercase hexadecimal digits
\u{NNNN}    one Windows UTF-16 code unit, with uppercase hexadecimal digits
```

Canonical encoding rules:

- normal displayable valid Unicode is written directly;
- literal backslash is always `\\`;
- Unix control bytes and bytes outside a valid displayed UTF-8 sequence use `\xNN`;
- Unix leading or trailing ASCII spaces in a component use `\x20`;
- Windows normal Unicode scalar values are displayed directly;
- Windows unpaired UTF-16 surrogate units use `\u{NNNN}`;
- aliases are invalid: decode followed by encode must reproduce the exact component text;
- malformed, truncated, lowercase-hex, overlong, or unknown escapes are syntax errors;
- decoding happens exactly once per component.

A literal Unix filename containing the text `\x41` is displayed as:

```text
\\x41
```

A Unix component with bytes `66 6F 80 6F 20` is displayed as:

```text
fo\x80o\x20
```

### 3.2 Path Grammar

A line is parsed as:

```text
EncodedComponent ("/" EncodedComponent)* [DirectorySlash]
```

Rules:

- the final slash marks a directory row and is not an empty component;
- an empty line is invalid;
- repeated separators are invalid;
- `.` and `..` are invalid after decoding;
- an encoded component cannot decode to a native separator;
- every decoded path remains relative to the Instance root;
- existing row kind is immutable through text editing;
- a new row with a final slash declares a directory; a new row without it declares a file;
- symlink creation uses the explicit symlink action rather than an ambiguous path marker.

Rust independently validates the decoded native components before planning.

## 4. Snapshot-Based Rendering

One render acquires one immutable `SnapshotHandle`:

```text
snapshot = engine.acquire_snapshot(root_session)
```

Lua queries the nodes required by its materialization state through bounded Snapshot pages, then performs:

```text
paged semantic nodes from one SnapshotHandle
  -> hidden and ignore presentation
  -> sibling-local sort
  -> baseline depth and NodeId expansion
  -> explicit and forced visibility
  -> depth-first ordered rows
  -> canonical flat path encoding
  -> buffer lines, extmarks, and decorations
```

A render never mixes Snapshot generations. Large child lists are consumed across scheduled slices using the handle's query cursor and record budget. The Snapshot handle remains pinned until projection and baseline commit finish, then it is released.

Internal rendering executes under `internal_sync_depth > 0`. Text-change handlers, provenance capture, and intent detection ignore edits produced by that guarded render.

## 5. NodeId Bindings

Each rendered existing or provisional row receives one full-line extmark mapped to one opaque NodeId token.

The initial binding shape is:

```text
start: row, column 0
end:   row + 1, column 0
right_gravity = false
end_right_gravity = true
invalidate = true
undo_restore = true
```

Lua stores:

```text
extmark_id -> NodeId token
NodeId token -> extmark_id
```

Binding rules:

- editing text on a bound row preserves its NodeId while the extmark remains valid;
- changing the row's parent path with the same binding is a candidate move;
- deleting the complete bound row invalidates its binding and is a candidate delete;
- a copied or newly inserted row has no existing binding and is a candidate create or provenance copy;
- one row may carry at most one NodeId;
- one NodeId may occur in at most one captured row;
- Lua never reconstructs a NodeId from a filename;
- ambiguous lost identity is reported rather than guessed.

M0 records actual Neovim behavior for edit, `:move`, `dd`/`p`, copy/paste, undo, and redo. Presentation commands that promise identity-preserving moves use operations whose extmark behavior is covered by those tests.

## 6. Committed Lua Baseline

After a successful render, Lua commits:

```text
Baseline {
  snapshot_generation,
  current_root_node_id,
  entries_by_node_id,
  visible_node_ids,
  encoded_lines,
  changedtick,
}
```

Each baseline entry contains:

```text
BaselineEntry {
  node_id,
  parent_node_id,
  native_name,
  native_relative_components,
  kind,
}
```

The baseline is Lua's authority for detecting changes in the editable projection.

Projection absence outside `visible_node_ids` is not deletion. Hidden, ignored, unscanned, collapsed, depth-excluded, filtered, failed, partial, or newly discovered model nodes that were not part of the baseline cannot become delete intents merely because no row is present.

### 6.1 Provisional Binding Ledger

Rows allocated for pending creates or provenance destinations are tracked separately from committed baseline entries:

```text
PendingProvisional {
  node_id,
  root_session_id,
  row_binding,
  origin: Create | ProvenanceDestination,
  reservation_state: Reserved,
}
```

Lifecycle rules:

- allocation inserts one Reserved ledger entry and one provisional extmark binding;
- a pre-commit validation error, stale result, preview cancellation, or cancellation preserves the reservation for a later write of the same draft;
- each prepare request may reference a reservation at most once;
- successful create/copy promotes the NodeId into the committed Snapshot and removes the ledger entry;
- a successful provenance-derived move retires the provisional NodeId through the authoritative terminal transition;
- deleting or undoing the pending row releases the reservation;
- discard/reload, navigation, and Instance destruction release every remaining reservation;
- a bound provisional row absent from the committed baseline remains provisional and is never mistaken for an existing filesystem node.

## 7. Parsing And Semantic Intent Detection

When the user writes the buffer, Lua parses it into:

```text
ParsedRow {
  row_index,
  binding: Existing(NodeId) | Provisional(NodeId) | Unbound,
  native_relative_components,
  parent_reference,
  declared_kind,
  provenance,
}
```

Lua first validates presentation syntax:

- canonical path escapes;
- duplicate encoded paths;
- duplicate NodeId bindings;
- path-parent relationships;
- existing kind preservation;
- new row kind declaration;
- root-relative structure;
- provenance qualification;
- provisional ledger ownership and reservation state.

Lua then compares parsed rows with the committed baseline and provisional ledger.

### 7.1 Existing Baseline Row

```text
same parent + same name
  -> no intent

same parent + new name
  -> Rename

new parent + same name
  -> Move

new parent + new name
  -> Move with new_name
```

### 7.2 Missing Baseline Row

A baseline NodeId absent from the new capture becomes:

```text
Delete
```

Deleting a directory row creates one semantic directory-delete intent. Its unmaterialized descendants do not need separate rows or delete intents.

### 7.3 Provisional Bound Row

A row whose NodeId appears in `pending_provisionals` reuses that reservation. Its current decoded path and provenance produce the same pending Create/Copy/Relocate destination semantics as the earlier attempt. Editing the pending row does not allocate another NodeId.

### 7.4 Unbound Row

After syntax validation, Lua requests a provisional NodeId from the RootSession, records it in `pending_provisionals`, and attaches it to the pending row.

An unbound ordinary row becomes:

```text
Create
```

A provenance-qualified row becomes a copy/move destination as described below.

### 7.5 Row Order

Changing only the order of rows whose parent NodeId remains the same produces no filesystem intent. Filesystem sibling order is presentation state.

Moving a directory and its materialized descendants produces one move for the directory when descendant parent identities remain unchanged.

## 8. Semantic Intent Records

Lua submits already semantic records:

```text
CreateIntent {
  intent_id,
  provisional_node_id,
  parent_node_id,
  native_name,
  kind,
}

RelocateIntent {
  intent_id,
  source_node_id,
  destination_parent_node_id,
  destination_name,
  provisional_destination_node_id: optional,
}

CopyIntent {
  intent_id,
  source_node_id,
  provisional_destination_node_id,
  destination_parent_node_id,
  destination_name,
}

DeleteIntent {
  intent_id,
  node_id,
}
```

Lua keeps:

```text
intent_id -> source row(s)
```

Rust diagnostics use `intent_id` and `NodeId`; Lua maps them to current rows and decorations.

## 9. Yank And Put Provenance

Lua owns register-qualified FRED provenance.

`TextYankPost` records:

```text
register name
register text
register type
RootSession identity
source NodeIds
```

Buffer-local `p` and `P` wrappers preserve the chosen register, count, direction, undo behavior, and cursor behavior, then record the inserted range.

Provenance is attached only when:

- the actual used register still matches the recorded text and type;
- source NodeIds belong to the same RootSession;
- the inserted range can be matched unambiguously.

Final semantic classification happens in Lua from the captured desired state:

```text
source retained + destination
  -> CopyIntent

source removed + one provenance destination
  -> RelocateIntent with provisional destination

source removed + multiple provenance destinations
  -> CopyIntent for each destination + DeleteIntent for source
```

Unqualified, ambiguous, or foreign-RootSession text creates ordinary pending entries.

`:FredPasteInto`/`gp` uses the same provenance model and keeps only top-level selected sources. Destination suffix selection is deterministic and remains subject to Rust final collision validation.

## 10. Presentation Controls And Coverage

### 10.1 Expansion And Depth

Lua stores expansion by directory NodeId:

```text
baseline depth 0: direct root children
baseline depth 1: direct children plus one child level
baseline depth all: recursive demand within configured limits
```

Per-directory expansion adds coverage scopes. Collapse releases corresponding lease demand while preserving cached model state.

### 10.2 Hidden Files

Lua owns:

```lua
hidden_files_visible
is_hidden_file(path, ctx)
```

When hidden visibility is false, an entry is excluded when it or a covered ancestor directory is classified hidden. Hidden classification affects projection, not filesystem identity.

A hidden-visibility change while dirty first derives the current semantic draft, reprojects model plus draft under the new visibility, resets the text undo baseline, and preserves `modified=true`.

### 10.3 Ignore Rules

Ignore rules use ordered gitignore-like semantics with negation and root anchoring. Lua applies presentation exclusion and sends traversal rules with its coverage lease so Rust can avoid unrequested recursive work.

Changing ignore rules updates the lease and projection. A dirty Instance preserves its semantic draft and becomes model-dirty until the requested coverage reaches a terminal generation.

### 10.4 Sort

Sort is sibling-local and uses a complete `SortSpec`. Reordering never creates intents.

A dirty buffer freezes its real-row sibling order. Metadata decorations may update, but rows do not move until the buffer becomes clean. A different runtime SortSpec is accepted only for a clean buffer; an identical normalized SortSpec is a no-op.

## 11. Metadata Columns

The editable line remains only the encoded path. Lua renders columns with extmark virtual text.

Built-in columns are:

```text
icon
permissions
size
mtime
ctime
atime
birthtime
type
```

Columns declare required metadata fields. The coverage lease carries their union. Missing values render stable placeholders and do not change namespace intent.

Column width is computed over one completed projection using Neovim display width. The path remains the final visible field. Changing columns updates metadata demand and decorations without creating filesystem operations.

## 12. BufferRuntime State Machine

```text
Editing
  -> Preparing

Conflict
  -> Preparing                 retry remaining

Preparing + NonEmptyPrepared
  -> Previewing

Preparing + EmptyPrepared
  -> Executing

Previewing + Confirmed
  -> Executing

Executing + Empty
  -> Editing

Executing + Succeeded
  -> Reconciling
  -> Editing

Executing + ExecutionFailed | CancelledAfterPartialApply
  -> Reconciling
  -> Conflict

Preparing | Previewing | pre-commit Executing
  -> attempt_origin
     Editing for an ordinary write
     Conflict for a retry attempt

Conflict
  -> Editing                   discard and reload

any state
  -> Destroyed                 terminal buffer lifecycle
```

`modified` distinguishes clean and dirty text while in `Editing`. `Conflict` remains read-only and modified until one of its explicit exits completes.

## 13. Write Routing

Every FRED buffer uses:

```text
buftype=acwrite
swapfile=false
undofile=false
filetype=fred
```

The private buffer URI is:

```text
fred://<engine-nonce>/<buffer-nonce>
```

It contains no filesystem path.

Buffer-local handlers are:

- `BufWriteCmd` for the full-buffer apply request;
- `FileWriteCmd` to reject ranged writes;
- `FileAppendCmd` to reject append writes.

Before accepting `BufWriteCmd`, Lua verifies:

- the event buffer is the registered Instance buffer;
- the buffer is current;
- the Instance is Open and displayed;
- the target is the exact private URI;
- the runtime state is `Editing`;
- no other apply attempt owns the buffer.

## 14. Asynchronous `BufWriteCmd`

The `BufWriteCmd` autocmd does not change buffer lines.

Inside the autocmd Lua:

1. records a new `attempt_id`;
2. captures changedtick, lines, and current extmark bindings;
3. stores the exact draft;
4. schedules semantic parsing and prepare work;
5. returns from the autocmd with `modified=true`.

The scheduled continuation:

1. verifies the buffer and changedtick are still current;
2. parses the draft and derives semantic intents;
3. sets `modifiable=false`;
4. enters `Preparing`;
5. starts Rust prepare.

`:write` and `:update` start this asynchronous process. A combined write-and-quit invocation remains open while the buffer is modified; the user may quit normally after a successful terminal render clears `modified`.

## 15. Prepare And Preview UX

When prepare succeeds, Lua verifies:

```text
buffer still exists
attempt_id is current
state == Preparing
captured changedtick is current
```

A valid non-empty plan enters `Previewing` and displays the semantic preview. A valid empty plan proceeds directly to finish.

Preview cancellation:

- releases `PreparedApplyHandle`;
- keeps all referenced provisional reservations reusable;
- restores `modifiable=true`;
- preserves the exact draft;
- preserves `modified=true`;
- returns to `attempt_origin`.

Validation, stale-model, probe, or planning errors follow the same draft-preserving path and add diagnostics through `intent_id`. For an ordinary write, `attempt_origin` is `Editing`; for Conflict retry, the runtime restores the existing read-only Conflict state and keeps its report while displaying the new pre-commit diagnostics.

## 16. Execution UX

On confirmation Lua verifies the attempt and changedtick again, then calls finish and enters `Executing`.

The buffer remains read-only while the task runs. Progress UI may request cooperative cancellation through the task handle.

### 16.1 Empty Plan

`Empty` names the current Snapshot generation and has no reconciliation task. Lua immediately:

1. acquires that fixed Snapshot;
2. renders canonical current model state;
3. rebuilds extmark bindings;
4. commits a new baseline;
5. clears `modified`;
6. restores `modifiable=true`;
7. returns to `Editing`.

### 16.2 Successful Non-Empty Plan

After Rust publishes known effects and the result's required reconciliation task reaches a terminal Snapshot generation, Lua:

1. acquires that fixed Snapshot;
2. applies authoritative NodeId transitions and reservation dispositions;
3. renders canonical current model state;
4. rebuilds extmark bindings;
5. restores cursor by NodeId when possible;
6. commits a new baseline;
7. clears `modified`;
8. restores `modifiable=true`;
9. clears any prior Conflict state;
10. returns to `Editing`.

### 16.3 Pre-Commit Cancellation Or Error

The original text was never replaced. Lua:

- preserves every reusable provisional reservation;
- preserves `modified=true`;
- preserves undo history and draft;
- displays diagnostics;
- returns to `attempt_origin`.

An ordinary attempt restores `modifiable=true` and `Editing`. A retry attempt restores the prior read-only `Conflict` state and report.

### 16.4 Post-Commit Failure Or Cancellation

Every `ExecutionFailed` and `CancelledAfterPartialApply` result enters `Reconciling -> Conflict`, including a first-syscall failure with no known successful effect, because the post-commit syscall outcome still requires authoritative observation.

Lua retains:

```text
ConflictState {
  draft_lines,
  draft_bindings,
  draft_baseline,
  reconciled_snapshot_generation,
  apply_report,
  completed_intents,
  failed_or_cancelled_intent,
  unstarted_intents,
  provisional_dispositions,
}
```

The buffer remains read-only and `modified=true`. Lua decorates completed, failed/cancelled, and unstarted rows distinctly.

The required exits are:

```text
Retry remaining
  Conflict -> Preparing
  -> derive remaining semantic intents against the reconciled Snapshot
  -> reuse valid unstarted provisional reservations
  -> prepare a new immutable plan
  -> pre-commit retry error returns to the same Conflict
  -> success commits a clean baseline and enters Editing

Discard and reload
  Conflict -> Editing
  -> release remaining provisional reservations
  -> render the reconciled Snapshot
  -> commit a new baseline
  -> clear modified
```

FRED never silently replaces a conflicted draft with the reconciled model.

## 17. Model Updates And Multiple Buffers

Several Lua Instances may share one RootSession while retaining separate baselines and drafts.

On `ModelChanged`:

### 17.1 Unmodified Buffer

Lua acquires the new Snapshot and schedules a bounded render. Expansion, filter, sort, cursor, and layout remain Instance-local.

### 17.2 Modified Buffer

Lua does not overwrite text. It sets:

```text
model_dirty = true
```

The UI shows a filesystem-change indicator. A later prepare carries the baseline Snapshot generation; Rust may return `StaleGeneration`, after which Lua offers refresh/reconcile while retaining the draft.

### 17.3 Applying Buffer

Events are matched by RootSession, task, attempt, and generation. A newer unrelated buffer render cannot consume another buffer's apply terminal state.

## 18. Reveal Presentation

Rust returns:

```text
snapshot_generation
target_node_id
ancestor_node_ids
```

Lua then:

1. adds ancestors to expansion state;
2. updates its coverage lease;
3. adds target and ancestors to `forced_visible_nodes`;
4. acquires the returned or a newer valid Snapshot;
5. verifies the target and ancestor chain still exist;
6. renders;
7. locates the target NodeId extmark;
8. moves cursor in the selected Instance window.

Forced visibility bypasses hidden and ignore presentation for exactly the resolved chain. It is cleared by the next reveal, a presentation-filter change, navigation to another Instance root, or Instance destruction.

If the target becomes stale before cursor movement, Lua retries resolution once. A second failure reports `RevealNotFound` without moving to a different node.

## 19. Navigation

Same-RootSession navigation changes the Lua Instance's current root NodeId. It preserves RootSession, NodeIds, buffer, windows, configuration, and NodeId-keyed expansion state.

Navigation requires a clean buffer. Lua rebases complete relative path display against the new root, updates coverage, renders one fixed Snapshot, commits a new baseline, and updates buffer metadata.

Opening a directory through a new child Instance creates an independent Lua presentation and coverage lease. When the selected directory belongs to the parent's RootSession, the child reuses that RootSession and snapshots expansion entries inside the selected subtree.

## 20. Buffer Destruction

`BufWipeout` enters one idempotent destroy path.

Lua:

- removes registry and extmark state;
- releases the coverage lease and Snapshot handles;
- cancels prepare/reveal/scan tasks owned only by that buffer;
- releases an unconsumed `PreparedApplyHandle`;
- releases every remaining provisional NodeId reservation;
- discards unapplied draft and Conflict state.

A finish task before commit is cancelled. A finish task after commit continues under Engine ownership and reconciles without attempting to update the destroyed buffer.

## 21. Presentation Conformance

Conformance requires:

- every Neovim operation occurs in Lua scheduled main-thread code;
- every rendered line follows the canonical flat path grammar;
- NodeId never appears as editable text or Lua number;
- extmark identity plus committed Lua baseline is the only edit-detection authority;
- provisional bindings are tracked separately, survive pre-commit retry, and are released or promoted deterministically;
- projection absence outside the baseline never creates deletion;
- same-parent reorder creates no filesystem intent;
- Lua submits semantic intents rather than raw buffer lines;
- `BufWriteCmd` changes no buffer lines inside the autocmd;
- Empty completes without waiting for reconciliation;
- pre-commit outcomes return to the attempt origin and preserve the draft;
- success clears `modified` only after canonical terminal rendering;
- every post-commit failure/cancellation reconciles, enters Conflict, and preserves the draft;
- Conflict retry and discard/reload have explicit terminal transitions;
- modified buffers are never silently overwritten by shared RootSession updates;
- Snapshot queries are paged and bounded within one pinned generation;
- reveal expansion, forced visibility, rendering, and cursor movement remain Lua-owned.
