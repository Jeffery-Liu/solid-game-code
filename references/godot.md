# SOLID in Godot (4.x)

Verified against Godot **4.5–4.7** (`@abstract`, typed Dictionaries, `duplicate_deep()` exist from 4.5/4.4). On an older 4.x, fall back to the alternatives noted inline.

Contents: [Node vs RefCounted vs Resource](#node-vs-refcounted-vs-resource) · [Nodes as components](#nodes-are-your-components) · [Procedural-draw projects](#procedural-draw-projects-no-scenes) · [Resources](#resources-for-data-driven-design) · [Capability interfaces](#capability-interfaces-in-a-duck-typed-language) · [Signals](#signals-up-calls-down) · [Injection](#injection-and-the-autoload-trap) · [Type checks](#type-checks) · [Testing](#testing-without-a-framework) · [Traps](#gdscript-traps-that-bite-during-refactors) · [C#](#c-in-godot) · [Performance](#performance-notes)

## Node vs RefCounted vs Resource

Decide this before anything else; it is where most Godot over-engineering starts.

| Make it a… | When it needs… | Examples |
|---|---|---|
| **Node** | `_process` / `_draw` / `_input`, tree lifecycle, a Timer or AudioStreamPlayer, or to be a z-layer | the thing that draws, a HUD layer, a Window |
| **RefCounted** | nothing from the tree - pure rules, ledgers, pickers, encoders | damage/patina formulas, inventory accounting, loot roller, schedule logic, a save-file codec |
| **Resource** | to be *read-only definition data* with `@export` typing | enemy stats, material tables, loot tables |

Runtime mutable state is **not** a Resource (see the sharing trap below) - keep it on the node or in a RefCounted with a hand-written `clone()`. If a class has no `_ready`, no signals, and never calls `get_node`, it should not extend `Node`: making it one only costs tree traversal and makes it untestable without a scene.

## Nodes are your components

Godot's scene tree is a composition system - use it instead of deep `extends` chains. A player scene composed of focused child nodes:

```
Player (CharacterBody2D)      <- assembly + physics body only
├── InputReader (Node)        <- turns raw input into an intent
├── Movement (Node)           <- consumes intent, moves the body
├── Health (Node)             <- hit points, emits died/damaged
├── Hurtbox (Area2D)          <- detection only, forwards to Health
└── Sprite2D / AnimationPlayer
```

Each child script has one reason to change. The parent wires them and holds almost no logic. The same `Health` node drops into an enemy or a destructible crate unchanged - that reuse is the payoff, and it is why `class_name PlayerHealth extends Health extends Node` chains are the wrong direction.

Scenes are the reuse unit: make `Health` its own `.tscn` if it carries configuration or child nodes, and instance it everywhere.

**This picture assumes entities are scenes.** If the project is one CanvasItem drawing everything procedurally, do not turn every drawn thing into a child node - see the next section.

## Procedural-draw projects (no scenes)

Many small Godot games are one `Node2D` with a big `_draw()` and no `.tscn` content. The component advice above is the wrong shape there. What works:

- **One CanvasItem per z-layer**, not per drawn thing: `World`, `Fx`, `Hud` as sibling `Node2D`s composited by tree order. Screenshot / "clean frame" modes become `hud.visible = false` instead of mode flags threaded through `_draw`.
- **Renderers are RefCounted with static functions that take the `CanvasItem`**: `PixelDraw.bead(ci, pos, r, …)` calls `ci.draw_rect(...)`. `draw_*` is only legal while `ci` is inside its own `NOTIFICATION_DRAW`, so a renderer can only be reached from a `_draw` call chain - never from `_process`.
- **`_draw` must be a pure projection of state.** Mutating simulation state inside `_draw` (spawning particles, decrementing timers with a smuggled `delta`) breaks every offscreen render path (thumbnails, GIF export, icon baking) - each has to back up and restore that state. Move timers to `_process`, emit a signal, let the Fx layer append.
- **Offscreen rendering** for thumbnails / exports: a `SubViewport` (`transparent_bg`, `render_target_update_mode = UPDATE_ONCE`, `await RenderingServer.frame_post_draw`, `get_texture().get_image()`) with a temporary drawer node inside it. This replaces "swap the live state, grab the main viewport, swap back". (4.7 adds `DrawableTexture2D` as a possible lighter alternative - verify its API before relying on it.)
- **Layout and hit-testing live together** with the drawer (both need the same geometry); do not split them into a `Layout` strategy until a second shape exists.

## Resources for data-driven design

Custom `Resource` subclasses are Godot's ScriptableObject, and the main OCP tool *when someone edits data in the inspector*:

```gdscript
class_name EnemyStats extends Resource

@export var max_health: float = 100.0
@export var move_speed: float = 120.0
@export var attack: AttackBehaviour        # a Resource subclass = strategy object
@export var loot_table: Array[LootEntry] = []
```

```gdscript
class_name Enemy extends CharacterBody2D

@export var stats: EnemyStats              # swap the .tres, get a new enemy
```

A new enemy type is then a new `.tres` file. When behaviour genuinely differs, subclass the behaviour `Resource` (`MeleeAttack`, `RangedAttack`) and call `stats.attack.execute(self, target)` - no `match enemy_type:` block anywhere.

**Three graded options, cheapest first:**

1. **`const` tables in a `*_data.gd` script** (`const DROP_TABLE: Array = [{...}, ...]`). Right for a solo project with no designer: one file, readable diffs, and `const` containers are read-only at runtime (an accidental `append` raises instead of corrupting shared data). Still "add content without touching logic".
2. **Resource subclasses built in code** (`static func materials() -> Array[MaterialDef]`). Adds typing - a misspelled field is a parse error instead of a `.get("tol", 1.0)` default. No `.tres` files to maintain.
3. **`.tres` files** when a non-programmer edits content in the editor. `ResourceSaver.save()` can export option 2 into option 3 in one line, so nothing is lost by starting lower.

**Sharing trap:** `ResourceLoader.load()` returns the same cached instance for the same path, so every user of `enemy.tres` shares one object. Mutating `stats.max_health` at runtime changes it for every enemy and, in the editor, can persist into the file. Treat loaded Resources as immutable config; `.new()`-constructed ones are not shared; `duplicate()` copies one level, `duplicate_deep()` (4.5+) copies nested Resources. Keep mutable runtime state off Resources entirely.

**Security trap:** a `.tres`/`.tscn` is *not* inert data - it can carry `[sub_resource type="GDScript"]` with inline source, or `ext_resource` a script, and `ResourceLoader` will instantiate it. `ConfigFile` values can also spell `Object(ClassName, ...)`. If untrusted / community content must be data-only, load **JSON** (`JSON.parse_string` cannot produce an Object), validate the fields against a whitelist, then build the Resource with a `from_dict()`.

**Derived constants:** `const AUTO_RATE := MAX / (DAYS * HOURS * 3600.0) * SPEED` chains are formulas, not tunables. They travel with the rules code, not into a numeric Resource - a "params" Resource cannot express them.

## Capability interfaces in a duck-typed language

GDScript has no `interface` keyword. Approaches, in order of preference:

1. **`@abstract` (4.5+)** - a base class whose required methods are declared without a body. A subclass that forgets one is a **parse-time** error, which beats every runtime check below. Abstract classes cannot be `.new()`-ed or attached to a scene node, so the concrete subclass is what gets instanced.

   ```gdscript
   @abstract
   class_name Curio extends Node2D

   signal changed
   @abstract func serialize() -> Dictionary          # no body - required
   @abstract func deserialize(d: Dictionary) -> void
   func handle_input(_e: InputEvent) -> bool:          # optional capability: default no-op
       return false
   ```

   Optional capabilities are **virtual methods with a no-op default**, not nullable strategy members (`if curio.care != null:` at every call site is type interrogation in disguise).
2. **A base with `push_error()` / `assert(false)` bodies** - the pre-4.5 substitute; same idea, runtime detection only.
3. **Groups** as capability markers: `add_to_group("damageable")`, then `if body.is_in_group("damageable")`. Only works for Nodes (RefCounted cannot join groups); the contract is a string - document it.
4. **`has_method("take_damage")`** duck typing. Least checkable, but the only option for RefCounted objects; acceptable at boundaries like collision callbacks.

Whichever you pick, be consistent across the codebase and keep the "interface" tiny - one or two required methods. In Godot the ISP failure mode is a single 30-method `Entity` base script that every node inherits and half-implements.

```gdscript
func _on_hitbox_body_entered(body: Node) -> void:
    if body.has_method("take_damage"):
        body.take_damage(damage, global_position)
```

## Signals up, calls down

The Godot idiom lines up exactly with DIP: **call down the tree, signal up.** A parent may call methods on its children (it owns them and knows their type); a child must never reach up with `get_parent()` or `$"/root/Game/UI/HealthBar"` - it emits a signal and lets whoever cares connect.

```gdscript
class_name Health extends Node

signal damaged(amount: float)
signal died

func take_damage(amount: float) -> void:
    current -= amount
    damaged.emit(amount)
    if current <= 0.0:
        died.emit()
```

Hard-coded node paths are the single most common source of coupling in Godot projects - they break the moment anyone reorganises the scene. If a node must reference something outside its own subtree, `@export var target: Node` and let the scene author assign it in the inspector: that is dependency injection with an editor UI on top.

An **EventBus autoload is not needed** when every signal has one emitter and one or two listeners wired in one place - that is just signals. A bus earns its place only when many unrelated emitters exist.

## Injection and the autoload trap

Autoloads (singletons) are convenient and heavily used in Godot, but every `GameState.player_health -= 10` inside gameplay code is a hidden global dependency that makes scenes untestable in isolation.

Guidelines:
- Keep autoloads few and boring: a save service, an audio service, scene transitions. Zero is a fine number.
- Access them at the *edges* (a scene's root script, a service adapter), not deep in behaviour scripts.
- For scene-built trees, prefer `@export` references assigned in the inspector. For code-built trees, a `setup(deps)` call **before** `add_child()` - children run `_ready` before the parent does, so anything a child needs in `_ready` must already be injected.
- **The root node is an implicit singleton** in a single-script project: every function shares `self`'s 100 member variables. Break it the same way as an autoload (playbook Recipe 6) - pass the two or three values a function actually reads as parameters, and it stops being glued to the root.
- **Parameter injection before interfaces.** `Time.get_unix_time_from_system()` and `randf()` inside rules code are the usual DIP violations; the fix is a `now: int` parameter and a `RandomNumberGenerator` field, not an `IClock` adapter (playbook Recipe 4, step 0).
- Pure rules (damage formulas, state machines, inventory logic) belong in plain `RefCounted` classes with no tree access - those are the ones you can test without a scene.

## Type checks

```gdscript
# LSP violation - the caller has to know the concrete class
if enemy is FlyingEnemy:
    enemy.descend_then_move(target)
else:
    enemy.move_to(target)

# Polymorphic - FlyingEnemy overrides move_to and still honours the contract
enemy.move_to(target)
```

`is` checks are fine when asking about a capability at a genuine boundary (a collision callback receiving `Node`), and wrong when the code already holds the right base type. The data version of the same smell is `if item.name == "Diamond":` inside draw code when the item table already has `mat`/`tier` fields - add an `fx` field to the row instead.

## Testing without a framework

You do not need gdUnit4/GUT to get the first assertion running; a bare `SceneTree` script is zero-addon:

```gdscript
# tests/run_tests.gd  —  godot --headless --path . --script tests/run_tests.gd
extends SceneTree

func _init() -> void:
    assert(Rules.stage_index(0.0) == 0)
    assert(Rules.stage_index(100.0) == 4)
    var rng := RandomNumberGenerator.new(); rng.seed = 42
    assert(Roller.pick(Data.DROP_TABLE, rng)["name"] != "")
    print("ok"); quit(0)
```

- `RefCounted` rules test with `.new()`; Node classes need `add_child` (or a framework's `auto_free`).
- `godot --headless --path . --script file.gd --check-only` is a parse/type check - run it on every changed file. Through the Windows console wrapper the exit code is always 0; grep the output for `SCRIPT ERROR`.
- Determinism: inject `RandomNumberGenerator` and `now`; never read `Time.*` or global `randf()` in a function you want to assert on.
- A build-blocking invariant (`assert(DEBUG_SPEED == 1.0)`) belongs in this file too.

## GDScript traps that bite during refactors

- **New `class_name` files are invisible to headless runs** until the editor rescans: run `godot --headless --path . --editor --quit` once (or open the editor). Symptom: `Identifier "Foo" not declared`. `.godot/global_script_class_cache.cfg` is the cache; it is not versioned.
- **`class_name` vs `preload`:** scripts referencing each other should use `class_name` (lazy, cycles allowed). `preload` cycles are a parse error. Do not keep a `const Foo = preload(...)` for a script that already has a `class_name` - pick one.
- **`const` containers are read-only**, including rows pulled out of a `const` Array. Copy before mutating.
- **Typed Arrays are invariant:** `Array[Child]` is not an `Array[Base]`, and an untyped Array (e.g. from `ConfigFile.get_value`) cannot be assigned to `Array[float]` - use `arr.assign(src)` or `Array[float](src)`. Typed Dictionaries (`Dictionary[StringName, Def]`) exist from 4.4.
- **Lambdas capture by value** and cannot write outer locals; for accumulators use a small inner class or an Array/Dictionary cell.
- **`static func` cannot touch instance state** - that is the point when extracting pure rules, and the reason a function that reads 20 members has to receive them as parameters first.
- **`duplicate()` is shallow** for nested Arrays/Dictionaries unless `duplicate(true)`; RefCounted has no `duplicate()` at all - write `clone()`.
- **Exported PCKs rename scripts** (`.gd` → `.gdc`/`.remap` when `script_export_mode` compiles them), so `DirAccess`-based "scan `res://plugins/*/` and auto-register" works in the editor and silently misses files in the export. Use an explicit list.
- **`await` inside a helper** that outlives its caller (an offscreen render finishing after the window closed): hold only your own nodes across the `await`, return an `Image`, and let the caller re-check `is_instance_valid`.

## C# in Godot

With the .NET build, real `interface`s are available - use them, and the Unity guidance about small capability interfaces applies directly. Note that C# interfaces do not cross the GDScript boundary: a GDScript node cannot implement a C# interface. In mixed projects, use groups or duck typing at the language boundary and interfaces within the C# side.

## Performance notes

- `_process`/`_physics_process` on hundreds of nodes is expensive in Godot. Disable processing (`set_process(false)`) on idle nodes, and prefer signal-driven updates.
- Avoid `get_node()` in per-frame code; cache in `_ready()` with `@onready`.
- Every node has memory and tree-traversal cost - splitting a bullet into five child nodes for tidiness is a real regression. For bullets, particles, and other high-count objects, keep them as one node (or drop to `Node2D` + servers / `MultiMeshInstance`) and put the abstraction at the manager level.
- Thousands of `draw_rect` calls per frame from one `_draw` are cheap (2D batching); a string compare on a material name inside that loop is fine. Do not replace it with per-pixel strategy objects for tidiness.
