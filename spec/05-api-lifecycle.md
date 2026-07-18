# FRED Public API And Lifecycle

This file is the normative source for the Lua public API, configuration, Instance ownership, layouts, actions, commands, buffer metadata, attach callbacks, lifecycle, cleanup timers, cleanup groups, refresh/navigation methods, and user-visible error behavior.

## 1. Module API

```lua
local fred = require("fred")

fred.build(opts?)
fred.setup(opts)
fred.new(opts)

fred.get_instance(instance_id)
fred.get_instance_by_name(name)
fred.get_instance_by_buf(bufnr)

fred.actions
```

`fred.build()` builds the native module for the current supported host target and installs it in the plugin's native artifact location. The normal loader selects a compatible release artifact when present and reports an actionable load/build diagnostic otherwise.

`fred.setup()` captures process-wide defaults, initializes the native loader, and validates setup-only configuration. It is idempotent for a compatible configuration.

## 2. Creating Instances

```lua
local explorer = fred.new({
  name = "project_files",
  profile = "project",
  root = "/project",
  layout = "float",
})
```

A direct `fred.new(opts)` resolves:

```text
latest setup configuration
  + explicit instance options
```

`root` is canonicalized during `new()`. A direct instance without `root` captures the current working directory at that moment.

Every Instance has an opaque unique InstanceId. A non-empty `name` is unique among live Instances. Destroying an Instance releases its name.

## 3. Child Instances

```lua
local child = explorer:new({
  root = "src",
  layout = "vsplit",
})
```

A child resolves:

```text
parent resolved construction configuration
  + snapshot of parent runtime presentation values
  + explicit child options
```

The runtime presentation snapshot includes:

- current hidden-file visibility;
- current SortSpec;
- current ignore rules;
- expansion entries within the selected child subtree.

Later parent changes do not alter the child. Child expansion evolves independently.

When the child root is a directory NodeId inside the parent's RootSession, the child shares that RootSession and uses the selected NodeId as its presentation root. Otherwise it opens or reuses the exact canonical RootSession for its root.

Automatically created directory children are unnamed unless an action supplies a name.

## 4. Instance API

Each Instance exposes:

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
instance:ignore()
instance:set_ignore(ignore_rules)
```

`config()` returns the immutable resolved construction snapshot. Runtime presentation getters return read-only copies. Runtime changes use explicit methods rather than a generic mutable configuration API.

## 5. Instance Lifecycle

```text
Created
  Instance exists; root/configuration are resolved; no buffer exists

Open
  the Instance buffer exists and at least one valid window displays it

Hidden
  the buffer exists and no valid window displays it

Destroyed
  terminal state; presentation resources and RootSession references are released
```

Transitions are:

```text
new()                         -> Created
open()                        -> Open
last displaying window gone  -> Hidden
open() while Hidden           -> Open
cleanup trigger               -> Destroyed
buffer delete/wipeout         -> Destroyed
destroy()                     -> Destroyed
```

One Instance owns at most one buffer and one presentation root, while that buffer may be displayed in multiple windows and tabpages.

Lua reconciles lifecycle from actual Neovim window/buffer state. Relevant triggers include:

```text
WinNew
BufWinEnter
WinEnter
WinClosed
BufHidden
BufDelete
BufWipeout
```

`WinClosed` reconciliation is scheduled after the closing window leaves normal window queries. Buffer deletion/wipeout enters the terminal destroy path directly.

## 6. RootSession And Presentation Sharing

Several Instances may share one RootSession. They share:

- immutable filesystem snapshots;
- NodeIds;
- metadata cache;
- scanner results;
- watcher coverage;
- RootSession task resources.

They do not share:

- buffers;
- drafts and committed baselines;
- modified state;
- sort, hidden, ignore, columns, or expansion state;
- cursor and windows;
- preview or Conflict state;
- cleanup timers or LRU position.

A model change is broadcast by RootSession identity. Each Instance decides independently whether to render, mark itself model-dirty, or ignore a stale event.

## 7. Opening And Displaying

```lua
explorer:open()
explorer:open({ layout = "vsplit", new_display = true })
```

`new({ layout = ... })` sets the Instance default layout. Per-call layout options select only the new display and do not mutate resolved configuration.

If the current tabpage already displays the Instance buffer, `open()` focuses one existing display. If it does not, `open()` creates a display in that tabpage. `new_display = true` explicitly creates an additional display.

`focus(opts)` selects an existing display according to explicit current-tab/window options and reports an error when no matching display exists.

## 8. Native Lua Layouts

Version one layouts are:

```text
buffer
float
split
vsplit
tab
```

Ownership rules:

- `buffer` borrows the current window and restores its previous, alternate, or a scratch buffer when hidden;
- `float` owns and closes its floating window;
- `split` and `vsplit` own their created split when closing it is safe, otherwise they restore another buffer;
- `tab` owns its created tabpage when another tabpage remains, otherwise it restores another buffer in the last ordinary window;
- hiding never forcibly deletes the final ordinary Neovim window.

`instance:hide()` targets the current window and requires that window to display the Instance. `instance:hide({ winid = ... })` targets one explicit display.

`instance:toggle(opts)` operates in the current tabpage:

```text
if the Instance is displayed there:
  hide those current-tab displays
else:
  open one display using opts.layout or the Instance default
```

Displays in other tabpages remain unchanged.

## 9. Default Directory Selection

The default directory selection action creates a child Instance and transfers the triggering FRED window to the child buffer.

When the selected directory belongs to the parent RootSession:

- the child shares the RootSession and NodeIds;
- the child uses the directory NodeId as its root;
- expansion inside that subtree is snapshotted;
- parent and child presentation state evolve independently.

The parent becomes Hidden when it loses its final display. Its dirty buffer remains available until reopened or destroyed by cleanup.

Users may configure directory selection to call same-Instance `navigate()` instead.

## 10. Buffer Identity And Options

Each Instance buffer has a unique private URI:

```text
fred://<engine-nonce>/<buffer-nonce>
```

The URI is presentation identity and contains no filesystem path.

Buffer options include:

```text
buftype=acwrite
swapfile=false
undofile=false
filetype=fred
```

The path list remains editable while the BufferRuntime is in `Editing`. Apply states may temporarily set `modifiable=false` according to `03-lua-presentation.md`.

## 11. Buffer Metadata

Before setting `filetype=fred`, Lua populates:

```lua
vim.b[bufnr].fred = {
  instance_id = "opaque-id",
  name = "project_files",
  profile = "project",
  root = "/project",
  layout = "float",
  state = "open",
  snapshot_generation = 12,
  metadata_generation = 4,
}
```

The table contains metadata rather than the live Instance object. Live retrieval uses registry functions.

`filetype` is exactly `fred`. Optional Instance categories use `name` and `profile`.

## 12. Buffer Initialization Order

Initialization order is:

```text
1. create the buffer and Lua BufferRuntime
2. open/reuse RootSession and create the coverage lease
3. populate vim.b[bufnr].fred
4. apply buffer-local options
5. install resolved buffer-local keymaps and write handlers
6. set filetype=fred and run FileType autocmds
7. run resolved on_attach callbacks
8. request initial coverage and begin rendering
```

This lets FileType autocmds inspect metadata and override configured mappings, while explicit Instance attach callbacks run last.

## 13. Attach Callbacks

`on_attach` accepts one function or a list of functions and is normalized to a list.

Direct Instance order is:

```text
setup.on_attach
then instance on_attach
```

A child inherits the parent's resolved callback list and appends explicit child callbacks. Setup callbacks occur once in that inherited list.

Each callback runs once when the Instance creates its single buffer:

```lua
on_attach = function(ctx)
  local instance = ctx.instance
  local bufnr = ctx.bufnr
  local actions = ctx.actions
  local config = ctx.config
end
```

Context:

```lua
{
  instance = instance,
  instance_id = instance:id(),
  bufnr = bufnr,
  actions = require("fred").actions,
  config = instance:config(),
}
```

The first callback error stops later callbacks, tears down resources created for that Instance, marks it Destroyed, and rethrows the original error.

## 14. Actions And Keymaps

Keymaps are grouped by Neovim mode. Mapping values are action functions or `false`:

```lua
local actions = require("fred").actions

fred.setup({
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

Merging uses `mode + lhs`:

- absent inherits;
- function replaces;
- `false` removes.

Actions use:

```lua
action(ctx, opts?)
```

The immutable invocation context is:

```lua
{
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

`actions.select()` dispatches by semantic entry kind using supplied functions.

## 15. Commands

```text
:Fred [root]                 create an unnamed Instance and open it
:FredRefresh                 refresh the current visible Open FRED Instance
:FredDepth {n|all}           change baseline recursion depth
:FredExpand [depth]          expand the current directory
:FredCollapse                collapse the current directory
:FredPasteInto               paste a FRED-aware yank into the current directory
:FredNew {file|dir}          insert an explicitly typed pending entry
:FredLink {from} {to}        create a symlink through the Engine mutation runner
```

Writing the current visible Open FRED buffer is the user entry point for buffer-derived filesystem operations. Preview, confirmation, progress, and cancellation are internal BufferRuntime behavior rather than separate public apply objects.

## 16. Refresh

`instance:refresh()` accepts an Open Instance with at least one valid display, even when another buffer is current.

`:FredRefresh` requires the current buffer to belong to a visible Open FRED Instance.

Refresh requests current coverage scopes, leaves a dirty draft intact, and sets model-dirty until a new Snapshot generation arrives. A clean buffer re-renders automatically.

## 17. Reveal

```lua
instance:reveal("src/lib/parser.lua", opts)
instance:reveal_current_file(opts)
```

`reveal()` accepts a root-relative path in the public Lua path form. Absolute paths first pass through:

```lua
instance:relative_path(absolute_path)
instance:contains(absolute_path)
```

Rust resolves filesystem target and ancestor NodeIds. Lua expands ancestors, applies temporary forced visibility, renders, and moves the cursor. Reveal does not change the Instance root.

## 18. Navigation

```lua
instance:navigate("src")
```

Navigation requires a clean buffer and a real directory inside the current RootSession. It preserves InstanceId, buffer, windows, RootSession, NodeIds, configuration, presentation values, and NodeId-keyed expansion state while changing the Instance root NodeId.

Lua updates buffer metadata, coverage, path display, cursor, and baseline. FileType and `on_attach` do not run again.

A symlink/reparse leaf target opens through an exact target RootSession rather than same-Instance navigation through the leaf.

## 19. Runtime Presentation Setters

### 19.1 Columns

`set_columns(columns)` validates built-in column specifications, updates metadata demand, and re-renders decorations. It does not change filesystem intent.

### 19.2 Hidden Files

`set_hidden_files_visible(boolean)` and `toggle_hidden_files()` follow the projection behavior in `03-lua-presentation.md`. A same-value call is a no-op.

### 19.3 Sort

`set_sort(spec)` requires a complete SortSpec and replaces the prior SortSpec as one value. An identical normalized value is a no-op. A dirty buffer accepts no different sort until it becomes clean.

### 19.4 Ignore

`set_ignore(rules)` validates the complete ordered rule list, updates coverage traversal demand and presentation filtering, and preserves any current semantic draft. `ignore()` returns a read-only copy.

## 20. Cleanup Configuration

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

`cleanup.instance` is inherited by direct and child Instances. `cleanup.groups` is setup-only, process-wide, and immutable after setup.

`group` must name a setup-defined group.

## 21. Hidden Cleanup

On `Open -> Hidden`, Lua:

1. inserts the Instance as the newest member of its cleanup-group LRU;
2. starts a one-shot timer when `delay_ms > 0`;
3. enforces group capacity by destroying oldest Hidden members until within capacity.

On `Hidden -> Open`, Lua:

- removes the Instance from the LRU;
- cancels its timer.

Semantics:

```text
delay_ms = 0   hidden-delay timer disabled
capacity = 0   group retains no Hidden Instances
```

Timer expiry and capacity eviction call the same idempotent destroy path.

A Hidden modified buffer retains its unapplied text until it is reopened or destroyed. Destruction discards that editor state without initiating filesystem mutation.

## 22. Destroy

`instance:destroy()` is idempotent.

It:

- closes/restores owned displays according to layout rules;
- removes registry entries and name ownership;
- cancels cleanup timer and removes LRU membership;
- wipes or releases the Instance buffer;
- releases Lua projection, baseline, draft, preview, and Conflict state;
- releases coverage and RootSession references;
- requests cancellation for pre-commit tasks owned by the Instance.

A committed mutation task continues under Engine ownership to a safe terminal result and reconciliation. It performs no later UI work for a destroyed Instance.

## 23. Public Configuration

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
    n = {},
    i = {},
    v = {},
  },
  on_attach = {},
})
```

### 23.1 SortSpec

```lua
{
  by = "path", -- path|type|size|mtime|ctime|atime|birthtime
  direction = "asc", -- asc|desc
  case_sensitive = false,
}
```

All three fields are required. Each explicit SortSpec replaces the previous value as a whole.

### 23.2 Columns

A column is a built-in name or a table whose first item is the built-in name:

```lua
columns = {
  "icon",
  "permissions",
  { "size", align = "right" },
  { "mtime", format = "%Y-%m-%d %H:%M", width = 16 },
}
```

Supported options include `width`, `align`, `format`, `highlight`, `separator`, and column-specific icon values.

### 23.3 Hidden Callback

```lua
is_hidden_file(path, ctx) -> boolean
```

`ctx` contains canonical display root and normalized kind. Exceptions and non-boolean returns are presentation errors.

### 23.4 Limits

Limits are positive integers and are validated during setup/new resolution. Per-Instance scan limits may narrow setup defaults. `render_batch_size` bounds one Lua projection/query slice. Engine mailbox and polling limits are native host defaults defined by the architecture and are not per-Instance configuration.

## 24. Symlink Command

```vim
:FredLink {from} {to}
```

`from` is the link target text passed according to platform symlink semantics. `to` is the destination.

Relative `to` resolves against:

- the current FRED Instance root for a FRED buffer;
- a normal file buffer's parent directory;
- the current window working directory for an unnamed/non-file buffer.

The destination is exclusive. The command uses the Engine mutation gate and the planning/execution publication protocol in `04-planning-execution.md`.

## 25. User-Visible Error Classes

Lua presents structured errors in these classes:

```text
NativeLoadError
ConfigurationError
LifecycleError
LayoutError
AttachError
PresentationError
ScanOrWatcherDiagnostic
PrepareDiagnostic
PreconditionError
ExecutionError
Conflict
```

Behavior:

- configuration and immediate lifecycle errors throw to the caller;
- scan/watch diagnostics leave readable model state and manual refresh;
- prepare and pre-commit errors preserve the edited draft;
- execution success renders the reconciled model and clears modified state;
- partial failure/cancellation preserves the draft in Conflict;
- attach failure tears down the partially created Instance;
- errors for one Instance do not select another Instance implicitly.

## 26. API And Lifecycle Conformance

Conformance requires:

- public lookup returns the live Instance by id, name, or buffer;
- one Instance owns one buffer while supporting multiple displays;
- lifecycle follows actual Neovim display state;
- layout hide behavior preserves a valid ordinary Neovim window;
- buffer metadata and initialization ordering match this specification;
- attach callbacks run once and in resolved order;
- keymap merge uses mode/lhs/function-or-false semantics;
- runtime setters use explicit methods and preserve documented dirty-state rules;
- cleanup timers and group capacity are Lua-owned and share one destroy path;
- destroying unapplied editor state performs no filesystem mutation;
- shared RootSession model data never merges Lua drafts or presentation state;
- write, refresh, reveal, navigation, and symlink commands delegate to their owning protocols rather than implementing parallel behavior.
