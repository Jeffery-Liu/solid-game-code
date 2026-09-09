# SOLID in Godot (4.x)

Contents: [Nodes as components](#nodes-are-your-components) · [Resources](#resources-for-data-driven-design) · [Capability interfaces](#capability-interfaces-in-a-duck-typed-language) · [Signals](#signals-up-calls-down) · [Injection](#injection-and-the-autoload-trap) · [Type checks](#type-checks) · [C#](#c-in-godot) · [Performance](#performance-notes)

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

## Resources for data-driven design

Custom `Resource` subclasses are Godot's ScriptableObject, and the main OCP tool:

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

**Caveat:** resources are shared between all users by default. Mutating `stats.max_health` at runtime changes it for every enemy and can persist into the saved `.tres` in the editor. Treat resources as immutable config; copy with `duplicate()` if an instance genuinely needs its own, and keep mutable runtime state on the node.

## Capability interfaces in a duck-typed language

GDScript has no `interface` keyword. Three workable approaches, in order of preference:

1. **An abstract base `Node` with `class_name`** and methods that `push_error()` (or `assert(false)`) if not overridden. Cheap and explicit; use for one capability at a time, keeping it small - `Damageable`, not `Entity`.
2. **Groups** as capability markers: `add_to_group("damageable")`, then `if body.is_in_group("damageable")`. Fast, but the contract is a string - document it.
3. **`has_method("take_damage")`** duck typing. Most flexible, least checkable; acceptable at boundaries like collision callbacks.

Whichever you pick, be consistent across the codebase and keep the "interface" tiny - one or two methods. In Godot the ISP failure mode is a single 30-method `Entity` base script that every node inherits and half-implements.

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

## Injection and the autoload trap

Autoloads (singletons) are convenient and heavily used in Godot, but every `GameState.player_health -= 10` inside gameplay code is a hidden global dependency that makes scenes untestable in isolation.

Guidelines:
- Keep autoloads few and boring: an event bus, a save service, an audio service, scene transitions.
- Access them at the *edges* (a scene's root script, a service adapter), not deep in behaviour scripts.
- For everything else, prefer `@export` references assigned in the inspector, or an `initialize(deps)` call right after `instantiate()`.
- Pure rules (damage formulas, state machines, inventory logic) belong in plain `RefCounted` classes with no tree access - those are the ones you can unit test with GUT/gdUnit without a scene.

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

`is` checks are fine when asking about a capability at a genuine boundary (a collision callback receiving `Node`), and wrong when the code already holds the right base type.

## C# in Godot

With the .NET build, real `interface`s are available - use them, and the Unity guidance about small capability interfaces applies directly. Note that C# interfaces do not cross the GDScript boundary: a GDScript node cannot implement a C# interface. In mixed projects, use groups or duck typing at the language boundary and interfaces within the C# side.

## Performance notes

- `_process`/`_physics_process` on hundreds of nodes is expensive in Godot. Disable processing (`set_process(false)`) on idle nodes, and prefer signal-driven updates.
- Avoid `get_node()` in per-frame code; cache in `_ready()` with `@onready`.
- Every node has memory and tree-traversal cost - splitting a bullet into five child nodes for tidiness is a real regression. For bullets, particles, and other high-count objects, keep them as one node (or drop to `Node2D` + servers / `MultiMeshInstance`) and put the abstraction at the manager level.
