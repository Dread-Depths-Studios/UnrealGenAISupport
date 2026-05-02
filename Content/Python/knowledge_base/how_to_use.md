# Unreal Engine MCP Plugin — Operator Guide

Not for humans. LLMs (Claude, etc.) read this at the start of a session to understand what they can and can't do, and how to drive the MCP without burning round-trips on trial and error.

---

## What this MCP actually is

A set of tools that drive a running UE5 editor over a local TCP socket. The most powerful tool is `execute_python_script`, which is a **remote shell into the editor**. It runs arbitrary Python with full `import unreal` access. It can:

- Delete assets, destroy actors, save packages
- Run console commands
- Modify project files and Blueprints
- Read/write any file the editor process has access to

There are **no real safety checks**. The original plugin had a destructive-keyword regex; it was removed because it was bypassable by saying "Yes, execute it" in chat. Be honest with yourself about this when planning operations.

## How to be safe

The user is expected to put their project under version control with a snapshot tag before letting you drive. If they haven't, ask once before doing anything destructive. The standard recovery is:

```
git reset --hard <snapshot-tag>
git submodule update --init --recursive
```

Plus you have **single-command undo**: every mutating command runs inside a UE5 editor transaction labeled `MCP: <command_type>`. The user can press **Ctrl+Z** in the level viewport (with focus on it) to reverse the most recent MCP-driven mutation. There is no MCP tool to invoke undo programmatically — only the user can do it.

If you're about to do something irreversible (delete an asset, overwrite a file, modify a Blueprint already in use), prefer:

1. Use the dedicated tool over `execute_python_script` when one exists — they're more predictable and have better error messages.
2. Read-before-write — use `get_all_scene_objects`, `get_all_nodes_in_graph`, `get_files_in_folder` to understand state first.
3. Surface the operation in the chat before executing, so the user can intercept.

## Response contract

Tools return human-readable strings. Underneath, the in-editor handlers return a uniform dict shape over the socket:

```json
{"success": true, "<result-fields>": ...}
{"success": false, "error": "<message>", "<optional-suggestions>": ...}
```

When a tool fails and includes `suggestions`, those are alternative inputs you can retry with (e.g. node-type fuzzy matches).

## Operational patterns (things that bite if you don't know them)

- **Idempotency on retries.** If you might call a tool more than once in a session (or across smoke runs), use a unique-per-run name suffix: `MyComponent_<timestamp>` instead of `MyComponent`. Otherwise `add_component_with_events`, `add_function_to_blueprint`, etc. fail with `"already exists"` on the second call.

- **Screenshot timing.** `take_editor_screenshot` is async — the engine queues it for the next render tick of the level viewport. If the level viewport isn't the foregrounded panel (because a BP editor is open), the screenshot can take 30+ seconds. **If `take_editor_screenshot` is your last operation**, queue any cheap follow-up (e.g. `handshake_test`) afterward to drive a render tick. The smoke harness solves this by issuing the screenshot first and verifying the resulting file at the end of the run.

- **BP editor steals viewport focus.** `create_blueprint`, `add_function_to_blueprint`, `add_component_with_events` open the BP editor. Subsequent `take_editor_screenshot` calls capture late or empty until the level viewport is foregrounded. Either tell the user to click into the level viewport, or sequence so any screenshot happens before the first BP-editor-opening command.

- **Connection failures return pin suggestions.** When `connect_blueprint_nodes` fails, the response includes `source_available_pins` and `target_available_pins`. Use those to retry with the correct pin name. Don't guess.

- **When a dedicated tool is buggy, fall back to `execute_python_script`.** Specifically: getter/setter spawning, complex node connections, and edge-case Blueprint surgery. Use `unreal.BlueprintEditorLibrary` and friends directly.

## Tool-specific gotchas

### Pin connections
For built-in events like `BeginPlay`, the execution pin is `"then"`, **not** `"OutputDelegate"`. `OutputDelegate` is for delegate-bound events. Verify pin names by attempting `connect_blueprint_nodes` and reading `source_available_pins`/`target_available_pins` on failure.

### Node types
Pass node types like `"EventBeginPlay"`, `"Multiply_FloatFloat"`, `"Branch"`, `"Sequence"` to `add_node_to_blueprint`. Unrecognized types return suggestions — use them and retry. Function libraries (`KismetMathLibrary`, `KismetSystemLibrary`, `KismetStringLibrary`) hold most common Blueprint functions; if a short name doesn't work, try the full name (`KismetMathLibrary.Multiply_FloatFloat` instead of `Multiply`).

### Node spacing
When laying out a graph, set `node_position` to space nodes at least **400 units horizontally and 300 vertically** apart. Tighter spacing produces overlapping nodes and an unreadable graph.

### Inputs
Two-step pattern:
1. `add_input_binding(action_name="Jump", key="SpaceBar")`
2. `add_node_to_blueprint(..., node_type="K2Node_InputAction", node_properties={"action_name": "Jump"})`

The `action_name` strings in step 1 and step 2 must match exactly.

### Colliders with overlap events
Use `add_component_with_events("MyBox", "BoxComponent")` rather than `add_component_to_blueprint` followed by manual event creation. The response includes `begin_overlap_guid` and `end_overlap_guid` you can wire from with `connect_blueprint_nodes`. The C++ utility now compiles the BP after adding the SCS node so the component property resolves correctly — earlier versions left the BP with a "valid matching component" warning, that's fixed.

### Materials
Set via `edit_component_property` with:
- `property_name`: `"Material"`, `"SetMaterial"`, or `"BaseMaterial"`
- `value`: a quoted asset path like `"'/Game/Materials/M_MyMaterial'"`

Default targets material slot 0.

### Screenshots
`take_editor_screenshot` captures the active level viewport via UE's `HighResShot` console command. Returns a PNG `Image` to the MCP client. Caveats:
- Level viewport must be the foregrounded rendering target (see "Operational patterns" above).
- Issued asynchronously — file appears on the next render tick. If you call it as the last command in a sequence, queue a follow-up command to drive the tick.

### EventGraph addressing (important — easy to miss)

Tools that take a `function_id` argument (`add_node_to_blueprint`,
`add_nodes_to_blueprint_bulk`, `connect_blueprint_nodes`,
`connect_blueprint_nodes_bulk`, `get_all_nodes_in_graph`) **also accept the
literal string `"EventGraph"`** to address the Blueprint's event graph
instead of a function graph's GUID.

So to wire a bound overlap event to call your function:
1. Find the bound event's GUID — it's returned by `add_component_with_events`,
   or use `get_blueprint_outline` to enumerate event-graph nodes.
2. Use `add_call_function_node` (see below) with `graph_identifier="EventGraph"`
   to add a node that calls your custom function.
3. `connect_blueprint_nodes(function_id="EventGraph", source_node_id=<event_guid>,
   source_pin="then", target_node_id=<call_node_guid>, target_pin="execute")`.

### Calling a Blueprint's own functions

`add_node_to_blueprint` only resolves library functions (`KismetMathLibrary`,
`GameplayStatics`, etc.) — it can't find a function you just created on the
Blueprint itself with `add_function_to_blueprint`. Use `add_call_function_node`
instead. It looks up the function on `Blueprint->GeneratedClass` and calls
`SetFromFunction` so the node compiles cleanly.

```
add_call_function_node(
    blueprint_path="/Game/Test/BP_OverlapDemo",
    graph_identifier="EventGraph",
    target_function_name="OnTrigger",
    node_x=400, node_y=0
)
```
Response includes the node GUID and the function's input/output pins so you
can immediately wire connections.

### Bulk node operations
`add_nodes_to_blueprint_bulk` adds N nodes in a single round-trip. Each node entry needs:
- `id`: your reference ID (string) — used to map to actual GUIDs in the response
- `node_type`: same vocabulary as `add_node_to_blueprint`
- `node_position`: `[X, Y]`
- `node_properties`: optional dict

Response: `{"success": true, "nodes": {"<your_id>": "<actual_guid>", ...}}`. Use those actual GUIDs with `connect_blueprint_nodes_bulk`.

### Class path conventions
- Native classes: `/Script/Engine.PointLight`, `/Script/Engine.Actor`
- Blueprint classes (when used as actor classes for spawning): suffix with `_C`, e.g. `/Game/Blueprints/BP_Guard.BP_Guard_C`
- Asset references (mesh, material, etc.): no suffix, e.g. `/Engine/BasicShapes/Cube.Cube`, `/Game/Materials/M_MyMat.M_MyMat`

### Editor subsystem migration (UE 5.5+)
`EditorLevelLibrary` and `EditorAssetLibrary` are deprecated in 5.5+. New `execute_python_script` content should use:
- `unreal.get_editor_subsystem(unreal.UnrealEditorSubsystem)` for world/level access
- `unreal.get_editor_subsystem(unreal.EditorAssetSubsystem)` for asset operations
- `unreal.get_editor_subsystem(unreal.EditorActorSubsystem)` for actor spawn/destroy

The old APIs still work in 5.7 with deprecation warnings; prefer the new ones in any Python you generate.

---

## Tool reference

### Meta / escape hatches

- **`how_to_use()`** — fetches this guide. Call once at session start.
- **`handshake_test(message)`** — confirms the in-editor socket listener is alive. Useful as a cheap follow-up to drive a render tick after `take_editor_screenshot`.
- **`execute_python_script(script)`** — runs arbitrary Python in the editor. Full `import unreal` access. **No safety checks**; treat as a remote shell. Use for anything not covered by the dedicated tools, or when a dedicated tool is buggy.
- **`execute_unreal_command(command)`** — runs an editor console command (`stat fps`, `obj list`, etc.). Rejects `py …` commands (use `execute_python_script` for Python). Output capture is limited; for detailed output, wrap in Python.
- **`take_editor_screenshot()`** — captures the active level viewport as PNG. See "Screenshots" gotcha above.

### Scene & actors

- **`spawn_object(actor_class, location, rotation, scale, actor_label)`** — spawns an actor.
  - For basic shapes: `actor_class="Cube"|"Sphere"|"Cylinder"|"Cone"`.
  - For other native classes: `actor_class="PointLight"` or full path `/Script/Engine.PointLight`.
  - For Blueprint actors: use `spawn_blueprint_actor` instead.
  - Returns: success message with the actor name.

- **`spawn_blueprint_actor(blueprint_path, location, rotation, scale, actor_label)`** — spawns an instance of a Blueprint asset.
  - `blueprint_path`: path to the BP asset (e.g. `/Game/Blueprints/BP_Pickup`). The C++ class path with `_C` is added internally.

- **`get_all_scene_objects()`** — returns JSON of all actors in the current level (name, class, location). Read-only.

- **`edit_component_property(blueprint_path, component_name, property_name, value, is_scene_actor=False, actor_name="")`** — sets a property on a component, either inside a Blueprint or on a live scene actor.
  - Vector/rotator/scale: `"100,200,300"`
  - Object refs: quoted asset path `"'/Game/Materials/M_MyMat.M_MyMat'"`
  - Booleans: `"true"`/`"false"`
  - For scene actors: pass `is_scene_actor=True, actor_name="Cube_1"` and leave `blueprint_path` empty.

### Materials

- **`create_material(material_name, color)`** — creates a new material asset with a flat RGB tint.
  - `color`: `[R, G, B]` 0–1.
  - Saves to `/Game/Materials/<material_name>`.

### Blueprints — creation

- **`create_blueprint(blueprint_name, parent_class="Actor", save_path="/Game/Blueprints")`** — creates a BP class.
  - Parent class: native class name like `"Actor"` or full path `/Script/Engine.Actor`.
  - If the BP already exists at the path, returns the existing one (idempotent).

- **`compile_blueprint(blueprint_path)`** — recompiles a BP. Necessary after structural edits if you want to spawn instances or read updated properties.

### Blueprints — components

- **`add_component_to_blueprint(blueprint_path, component_class, component_name=None)`** — adds a component to a BP's SCS.
  - `component_class`: e.g. `"StaticMeshComponent"`, `"PointLightComponent"`.

- **`add_component_with_events(blueprint_path, component_name, component_class)`** — adds a collision-shape component AND wires up its overlap events. Use this when you want `OnComponentBeginOverlap`/`OnComponentEndOverlap` event nodes auto-created. Component class must derive from `UShapeComponent` (`BoxComponent`, `SphereComponent`, `CapsuleComponent`).
  - Response: `{"success": true, "begin_overlap_guid": "...", "end_overlap_guid": "..."}` — those GUIDs identify the event nodes for `connect_blueprint_nodes`.

### Blueprints — variables & functions

- **`add_variable_to_blueprint(blueprint_path, variable_name, variable_type, default_value=None, category="Default")`** — adds a typed variable.
  - `variable_type`: `"float"`, `"vector"`, `"boolean"`, etc.

- **`add_function_to_blueprint(blueprint_path, function_name, inputs=[], outputs=[])`** — adds a function. Returns `function_id` (a GUID) for use with subsequent node tools.
  - `inputs`/`outputs`: lists like `[{"name": "param1", "type": "float"}]`.

### Blueprints — nodes

- **`add_node_to_blueprint(blueprint_path, function_id, node_type, node_position=[0,0], node_properties={})`** — adds one node to a function graph. Pass `function_id="EventGraph"` (literal) to target the event graph. Resolves library functions (KismetMathLibrary, GameplayStatics, …); for the Blueprint's own functions use `add_call_function_node`.
- **`add_call_function_node(blueprint_path, graph_identifier, target_function_name, node_x, node_y)`** — adds a `K2Node_CallFunction` that targets a function on the Blueprint itself or any parent class. Returns node GUID + pin layout. Use after `add_function_to_blueprint` to wire your custom function into a graph.
- **`add_nodes_to_blueprint_bulk(blueprint_path, function_id, nodes)`** — adds N nodes in a single round-trip. See "Bulk node operations" gotcha above.
- **`delete_node_from_blueprint(blueprint_path, function_id, node_id)`** — removes a node by GUID.
- **`get_all_nodes_in_graph(blueprint_path, function_id)`** — returns JSON of all nodes (GUID, type, position). Read-only.
- **`get_node_suggestions(node_type)`** — fuzzy-search for node type strings. Read-only.
- **`get_blueprint_node_guid(blueprint_path, graph_type="EventGraph", node_name=None, function_id=None)`** — finds an existing node's GUID by name (e.g. `BeginPlay`). Use for FunctionEntry nodes too. Read-only.

### Blueprints — wiring

- **`connect_blueprint_nodes(blueprint_path, function_id, source_node_id, source_pin, target_node_id, target_pin)`** — connects one source-pin → target-pin. On failure, response includes `source_available_pins` and `target_available_pins`.
- **`connect_blueprint_nodes_bulk(blueprint_path, function_id, connections)`** — connects N pin pairs in a single round-trip. `connections`: list of `{source_node_id, source_pin, target_node_id, target_pin}` dicts. Per-connection error reporting on partial failure.

### UI / UMG widgets

- **`add_widget_to_user_widget(user_widget_path, widget_type, widget_name, parent_widget_name="")`** — adds a widget to a User Widget Blueprint.
  - `widget_type`: `"TextBlock"`, `"Button"`, `"Image"`, `"CanvasPanel"`, `"VerticalBox"`, `"HorizontalBox"`, `"SizeBox"`, `"Border"`. Case-sensitive.
  - `parent_widget_name`: name of an existing panel widget to attach to. Empty = root or first available CanvasPanel.

- **`edit_widget_property(user_widget_path, widget_name, property_name, value)`** — sets a widget property. Layout properties controlled by the parent panel (position, size, anchors in a CanvasPanel) require a `Slot.` prefix. Value uses Unreal `ImportText` syntax:
  - Text: `'"Hello World!"'`
  - Float: `'150.0'`
  - LinearColor: `'(R=1.0,G=0.0,B=0.0,A=1.0)'`
  - Vector2D: `'(X=200.0,Y=50.0)'`
  - Anchors: `'(Minimum=(X=0.5,Y=0.0),Maximum=(X=0.5,Y=0.0))'`
  - Texture: `"Texture2D'/Game/Textures/MyIcon.MyIcon'"`
  - Enum: `'ScaleToFit'`

### Introspection (read-only)

Use these to **read state before mutating** — saves round-trips and prevents collisions.

- **`get_blueprint_outline(blueprint_path)`** — returns JSON of components, variables, functions, and event-graph nodes for a Blueprint. Use before `add_component_with_events` / `add_function_to_blueprint` / etc. to check for name collisions, or to find existing node GUIDs without regenerating them.

- **`list_actors_by_class(class_name)`** — returns scene actors whose class matches `class_name` (simple name like `"StaticMeshActor"` or full path `"/Script/Engine.StaticMeshActor"`). Exact match — does not include subclasses. Use to target operations precisely instead of scanning all actors.

- **`get_node_pins(blueprint_path, node_id)`** — returns input and output pins on a specific node by GUID, with name + type for each pin. Use to plan `connect_blueprint_nodes` calls without trial-and-error. Note that the same info is returned by `connect_blueprint_nodes` only on connection failure — this surfaces it preemptively.

### Project / files

- **`create_project_folder(folder_path)`** — creates a folder under `/Game`.
- **`get_files_in_folder(folder_path)`** — lists assets in a `/Game/...` folder. Read-only.
- **`create_game_mode(game_mode_path, pawn_blueprint_path, base_class="GameModeBase")`** — creates a GameMode BP, sets its default pawn, and assigns it as the level's default game mode.
- **`add_input_binding(action_name, key)`** — registers an action mapping in Project Settings. `key` examples: `"SpaceBar"`, `"LeftMouseButton"`, `"E"`.

---

## Things this plugin can't do (yet)

- No `connect_nodes` for delegate-bound events to BP functions in some configurations — fall back to `execute_python_script` with `unreal.BlueprintEditorLibrary`.
- No programmatic undo. Only the user pressing Ctrl+Z reverses MCP transactions.
- No "find this asset" / "search assets by name" tool — use `get_files_in_folder` recursively or `execute_python_script` with `unreal.EditorAssetLibrary`/`EditorAssetSubsystem` for asset queries.
- `take_editor_screenshot` doesn't currently force the level viewport to the foreground; you may get a stale/empty capture if a BP editor is on top. Workaround in "Operational patterns" above.

---

*Append additional gotchas as discovered.*
