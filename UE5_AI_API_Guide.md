# UE5 ↔ AI API Guide — DeadHeadz57

**Project:** DeadHeadz57 | UE 5.7
**Audience:** future-me, future-Claude, anyone driving this project's AI pipeline
**Sister docs:** [UE5_MCP_Plugin_Plan.md](UE5_MCP_Plugin_Plan.md) (MCP plugin status & roadmap), [UE5_AI_LevelGen_Plan.md](UE5_AI_LevelGen_Plan.md) (level-gen pipeline), [Plugins/GenerativeAISupport/Content/Python/knowledge_base/how_to_use.md](Plugins/GenerativeAISupport/Content/Python/knowledge_base/how_to_use.md) (MCP tool reference)

---

## Why this exists

The UE5 Python API is wider than 5.4-era docs suggest, but **strategically thin in a few load-bearing places** — and the gaps are not where you'd guess. This doc captures, in one place:

1. The four execution paths into the editor and when to pick each
2. What works first-shot vs. what burns tokens on retries
3. Where the Python binding has hard walls (the **AnimGraph** finding is the headline)
4. Project-specific patterns learned the hard way

If you're authoring assets through AI in this project, read this before [how_to_use.md](Plugins/GenerativeAISupport/Content/Python/knowledge_base/how_to_use.md). That doc is the per-tool reference; this one is the architecture and decision tree.

---

## The four execution paths

| Path | When to use | Cost | Reversibility |
|---|---|---|---|
| **Dedicated MCP tool** (`add_component_to_blueprint`, `spawn_blueprint_actor`, etc.) | A tool exists, isn't on the broken list, and the op fits its shape | 1 round-trip; tool is wrapped in `ScopedEditorTransaction` | Ctrl+Z if the underlying handler calls `Modify()` |
| **`execute_python_script`** | Composite work, missing tool, or escape-hatch when an MCP tool is buggy | 1 round-trip but you ship a whole script; full `import unreal` | Ctrl+Z works for whatever stdlib calls you make that hit reflection-aware setters |
| **Hand-author the asset, drive references via Python** | The Python API can't construct the asset type (AnimBP, complex Materials, behaviour trees) | One-time author cost, reused per character/instance | Trivial — assets are versioned in git |
| **C++ editor helper exposed as `UFUNCTION(BlueprintCallable)`** | You need a C++ API that isn't reflected to Python (`UbergraphPages`, AnimGraph schema, etc.) | Plugin rebuild, but pays back across many calls | Whatever you wrap into the helper |

The MCP plugin ([Plugins/GenerativeAISupport](Plugins/GenerativeAISupport)) is itself an instance of pattern 4 — every tool in [GenBlueprintNodeCreator.cpp](Plugins/GenerativeAISupport/Source/GenerativeAISupportEditor/Private/MCP/GenBlueprintNodeCreator.cpp) is a `UFUNCTION` wrapper around things Python can't reach directly. Adding a new C++ helper is the established pattern when you discover a new Python gap.

---

## What works well today

### FBX import + shared-skeleton reuse

The pattern in [ue_character_pipeline.py:import_mhr_character](Plugins/ComfyUIBridge/Content/Python/ue_character_pipeline.py) is the clean shape:

- Build `unreal.FbxImportUI` with `options.skeleton = sk_mhr` so all characters share `SK_MHR`
- Run via `unreal.AssetToolsHelpers.get_asset_tools().import_asset_tasks([task])`
- Filter `task.imported_object_paths` to find the SkeletalMesh (skip `_Skeleton`, `_PhysicsAsset`)

Every character imported this way inherits every animation in `/Game/Animations/` for free, because all anims target `SK_MHR`. No per-character anim work.

### Asset library traversal and duplication

`unreal.EditorAssetSubsystem` (5.5+) is well-exposed:

- `does_asset_exist`, `does_directory_exist`, `make_directory`
- `list_assets(path, recursive=False)` — returns object paths, not asset data
- `duplicate_asset(source, dest)` — solid for the **template-and-duplicate** pattern (relevant below for AnimBPs)

### Bulk node operations on K2 graphs

`add_nodes_to_blueprint_bulk` and `connect_blueprint_nodes_bulk` (re-enabled in MCP plan Phase 3.1) — pass an array of nodes with reference IDs, get back a `{ref_id: GUID}` mapping in one round-trip. Use these instead of per-node calls for any non-trivial graph. Confirmed working end-to-end via the smoke harness.

### Introspection (read-only)

`get_blueprint_outline`, `get_node_pins`, `list_actors_by_class`, `get_all_nodes_in_graph`. These are C++-shimmed because Python can't walk `UbergraphPages` / `FunctionGraphs` / `SimpleConstructionScript` / `NewVariables` directly (see [Where Python hits walls](#where-python-hits-walls)). Use them before authoring — much cheaper than trial-and-error.

### Screenshot capture

`take_editor_screenshot` was rewritten in MCP Phase 2.4 to use `HighResShot` (real viewport, not desktop). Async timing caveat: queue a cheap follow-up command (e.g. `handshake_test`) after it if it's the last operation in your chain — the render tick that finalizes the file only fires when something else drives the level viewport.

### Transactional safety

Every mutating MCP command runs inside `unreal.ScopedEditorTransaction("MCP: <command_type>")`. Ctrl+Z reverses MCP-driven edits **as long as the underlying handler called `Modify()`**. Epic's subsystem APIs do this correctly. Some custom utilities may not — verify per handler if undo turns out incomplete.

---

## Where Python hits walls

### AnimGraph authoring — confirmed against UE 5.7 docs (2026-05-06)

**Verdict:** classic `UAnimBlueprint` AnimGraph authoring is **not exposed** in Python. There is no `add_node` / `create_state_machine` / `connect_pins` API for animation graphs. This is the load-bearing finding for any character-pipeline work.

#### What is exposed

| Class | Surface |
|---|---|
| `unreal.AnimBlueprint` | `add_node_asset_override(target, override)` — swap which anim asset an existing node plays. Read: `get_animation_graphs()`, `get_nodes_of_class()`, `graph_nodes` |
| `unreal.AnimationGraph` | `get_graph_nodes_of_class()` — read-only |
| `unreal.AnimGraphLibrary` | Runtime math only (`calculate_direction`, `look_at`, `two_bone_ik`) — no graph construction |
| `unreal.AnimGraphNode_Base` and many subclasses | Constructible as bare UObjects (`unreal.AnimGraphNode_StateMachine(outer=..., name=...)`), but no `AllocateDefaultPins`, no schema, no pin-connection API |

#### What is not exposed

- No `add_node` / `spawn_node_from_template` / `create_node` for AnimGraphs (the K2 spawn helpers used by [GenBlueprintNodeCreator.cpp](Plugins/GenerativeAISupport/Source/GenerativeAISupportEditor/Private/MCP/GenBlueprintNodeCreator.cpp) target `UEdGraphSchema_K2`; AnimGraphs use `UAnimationGraphSchema`, with no Python counterpart)
- No exposed pin-connection API for AnimGraph pose pins
- **State machines** (`UAnimStateMachineGraph`, `UAnimStateNodeBase`, `UAnimStateTransitionNode`) have no exposed creation methods at all
- No way to trigger `AllocateDefaultPins` / `PostPlacedNewNode` / `ReconstructNode`, so even a `new_object()`-allocated AnimGraphNode is an unwired carcass that won't compile

#### One nuance

5.7 ships `unreal.AnimNextAnimGraphBuilder` and `AnimNextSimpleAnimGraphBuilder` (plugin `UAFAnimGraph`) — these **are** procedural builders, but they target Epic's new experimental **AnimNext** framework, not classic `UAnimBlueprint`. They cannot build the AnimBP that drives `SK_MHR` characters.

#### The three viable workarounds, ranked

1. **Hand-author `ABP_MHR` once, assign by reference.** Build the state machine in the editor, then have Python do `comp.set_anim_class(load_class('/Game/.../ABP_MHR.ABP_MHR_C'))`. Per-character variation goes in a `UDataAsset` the ABP reads — also Python-authorable, since `UDataAsset` properties **are** reflected. **Default to this.**
2. **Duplicate a template per character** via `unreal.EditorAssetLibrary.duplicate_asset` and only retarget references (target skeleton, idle pose, etc.) on the duplicate. Useful when characters need genuinely divergent state machines but share a topology.
3. **C++ editor helper.** Add `UFUNCTION(BlueprintCallable)` wrappers in `GenerativeAISupportEditor` around `FBlueprintEditorUtils` + `FEdGraphSchemaAction_K2NewNode` against `UAnimationGraphSchema`. Real engineering project — only do this if routes 1 and 2 genuinely don't fit.

Sources verified 2026-05-06: [unreal.AnimBlueprint (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimBlueprint?application_version=5.7), [unreal.AnimationGraph (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimationGraph?application_version=5.7), [unreal.AnimGraphLibrary (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimGraphLibrary?application_version=5.7), [unreal.AnimGraphNode_Base (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimGraphNode_Base?application_version=5.7), [unreal.AnimNextAnimGraphBuilder (5.7)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/class/AnimNextAnimGraphBuilder?application_version=5.7).

### Blueprint internals walking

Confirmed during MCP Phase 4.2: `UbergraphPages`, `FunctionGraphs`, `SimpleConstructionScript`, and `NewVariables` are **not exposed** on `unreal.Blueprint` — neither as direct attributes nor via `get_editor_property`. Anything that needs to walk these requires a C++ shim. The introspection tools (`get_blueprint_outline`, `get_node_pins`) are exactly that shim.

**General rule established:** if you need to walk Blueprint internals from script, expect to write C++. The Python binding is more restrictive in 5.7 than the 5.4-era docs imply.

### Other holes worth knowing

- **Behaviour Trees** — same shape as AnimGraph; constructible as assets, but the graph schema isn't reflected. Hand-author + reference is the move.
- **Material graphs** — partial Python support via `unreal.MaterialEditingLibrary` (good for parameter swaps, MaterialInstance creation), but full graph authoring is C++-only.
- **Physics Asset graph** — auto-generation works (`create_physics_asset` on `FbxImportUI`), but tweaking constraints/bodies post-hoc has no first-class Python API.

---

## What's still rough in the MCP

Per [UE5_MCP_Plugin_Plan.md](UE5_MCP_Plugin_Plan.md), several items remain partial or deferred. If your AI-driven session hits one of these, **don't burn cycles fighting it** — fall back per the workaround.

| Issue | Workaround |
|---|---|
| Getter/setter node spawning is flagged buggy by the upstream author | Drop to `execute_python_script` and use the Python reflection API directly |
| Some custom C++ utilities may not call `Modify()` → Ctrl+Z is incomplete in spots | Verify undo on the specific op before relying on it; spot-fix the C++ if it bites |
| `take_editor_screenshot` async timing | Queue a cheap follow-up command (`handshake_test`) after it if it's last in chain |
| Editor docking issues when MCP focuses BP editors mid-session | Cosmetic; does not affect correctness |
| **No socket auth on `localhost:9877`** | **Deferred by design** — single-user Windows box; threat model in [UE5_MCP_Plugin_Plan.md](UE5_MCP_Plugin_Plan.md#21-socket-auth-on-port-9877--deferred). Revisit if a teammate joins, the editor runs on a shared box, or the port is forwarded |

---

## Project-specific patterns

### Pickup / trigger collision setup

Pickups (and similar overlap-driven actors) need:

- **Mesh component:** `CollisionEnabled = QueryAndPhysics`, `CollisionProfileName = "OverlapAll"`
- **Trigger component (sphere/box):** `CollisionEnabled = QueryOnly`, `CollisionProfileName = "Trigger"`

Two gotchas:

1. `CollisionProfileName` is a `FName` UPROPERTY that **does not round-trip through `set_editor_property` cleanly from MCP** in some cases. When in doubt, set it via `execute_python_script`: `comp.set_collision_profile_name("OverlapAll")`.
2. **Live instances do not auto-update** when you edit the Blueprint default. After a profile change in the BP, existing actors in the level keep their old profile until respawned or manually re-applied. If you're testing in PIE and "the change didn't take," check this before assuming the edit failed.

### ComfyUI bridge — extend, don't side-channel

All ComfyUI traffic goes through the bridge MCP ([Plugins/ComfyUIBridge](Plugins/ComfyUIBridge)). **Never** raw-`curl` the ComfyUI server from a Python script in the editor — even if it'd work for a one-shot, it bypasses the bridge's job tracking, caching, and OBJ→FBX conversion.

If you need a new ComfyUI capability (new workflow, new model), add it as a bridge tool. Examples already in place: `gen_character_image`, `gen_character_rig`, `gen_character_turnaround`. The pattern is documented in [Plugins/ComfyUIBridge/SETUP.md](Plugins/ComfyUIBridge/SETUP.md).

### Bridge polling — check `status.completed`, not output shape

On Windows, the bridge daemon's sync request loop can wedge past Ctrl+C if you poll for output-shape completeness instead of the explicit completion flag. Always loop on `status.completed == True` (or equivalent in the response envelope), never on "did the output dict have the keys I expected."

If the daemon does hang, kill the Python process directly — Ctrl+C in the terminal that launched it may not propagate.

---

## Decision tree: which path do I take?

```
Authoring or modifying an asset?
├── It's a SkeletalMesh, StaticMesh, Texture, anim asset, DataAsset
│   → MCP tool or execute_python_script. Python API is solid here.
│
├── It's a Blueprint (regular K2)
│   ├── Need bulk node placement
│   │   → add_nodes_to_blueprint_bulk + connect_blueprint_nodes_bulk
│   ├── Need a getter/setter node
│   │   → execute_python_script (MCP tool is buggy)
│   ├── Need to walk graph internals
│   │   → get_blueprint_outline / get_node_pins (C++-shimmed)
│   └── Anything else
│       → MCP node-creation tools
│
├── It's an AnimBlueprint
│   → STOP. Hand-author once, duplicate per character, drive references via Python.
│   → Do NOT try to construct AnimGraph nodes from script. The API isn't there.
│
├── It's a Material
│   ├── MaterialInstance from existing parent → unreal.MaterialEditingLibrary
│   └── Full Material graph authoring → hand-author or C++ helper
│
├── It's a Behaviour Tree
│   → Hand-author. Same shape as AnimGraph.
│
└── It's something else (UMG widget, GameMode, level actor)
    → MCP has dedicated tools; check how_to_use.md first.
```

---

## References

- [Plugins/GenerativeAISupport/Content/Python/knowledge_base/how_to_use.md](Plugins/GenerativeAISupport/Content/Python/knowledge_base/how_to_use.md) — per-tool MCP reference (~250 lines, kept current with plan phases)
- [UE5_MCP_Plugin_Plan.md](UE5_MCP_Plugin_Plan.md) — what's done, what's deferred, why
- [UE5_AI_LevelGen_Plan.md](UE5_AI_LevelGen_Plan.md) — level-gen pipeline architecture
- [Plugins/ComfyUIBridge/SETUP.md](Plugins/ComfyUIBridge/SETUP.md) — bridge setup and tool authoring
- [Unreal Python API 5.7 (root)](https://dev.epicgames.com/documentation/en-us/unreal-engine/python-api/?application_version=5.7)
- [Unreal Engine 5.7 Documentation](https://dev.epicgames.com/documentation/unreal-engine/unreal-engine-5-7-documentation)

---

*Last updated: 2026-05-06. AnimGraph verdict re-verified against 5.7 docs same day. Update this doc when (a) a UE point release changes Python surface area, (b) a new C++ helper unblocks something previously listed as a wall, or (c) the MCP plan moves an issue from "rough" to "fixed."*
