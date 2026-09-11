---
name: solid-game-code
description: >-
  Apply the SOLID principles to game code the way experienced game programmers
  actually apply them - decoupled, data-driven, testable, and still fast in the
  hot path. Use this skill whenever writing, reviewing, refactoring, or planning
  game code in any engine (Unity C#, Unreal C++, Godot, Bevy, custom engines) -
  gameplay systems, entities and components, managers, state machines, abilities,
  damage and health, inventory, save systems, UI wiring, AI behaviour, or
  engine-facing services. Trigger it even when the user never says "SOLID",
  "architecture", or "refactor" - a request like "add a dash ability", "make an
  enemy that shoots", "why is this class such a mess", or "clean up my
  PlayerController" is exactly when this skill is needed.
---

# SOLID for Game Code

Game codebases rot in a specific way: one `Player` or `GameObject` class swallows every feature, `switch` statements on enemy type sprout in twelve files, and systems reach for each other through singletons until nothing can be changed or tested in isolation. SOLID is the standard antidote, but applied dogmatically it produces the *other* failure mode - a maze of one-implementation interfaces, factories, and virtual calls in the middle of a 10,000-entity update loop.

This skill is about hitting the middle: reduce coupling where change actually happens, stay concrete where the profiler cares.

## The core question

Before adding an abstraction, answer: **what change is this protecting me from, and is that change likely?**

If the answer is "adding a new enemy/weapon/ability/upgrade", the abstraction is almost always worth it - designers will add fifty of those. If the answer is "we might swap physics engines someday", it usually is not. Write the concrete thing, and extract the seam when the second case arrives.

## The five principles, in game terms

### S - Single Responsibility

*One reason to change per class.* The classic violation is the thousand-line `PlayerController` that reads input, moves, plays audio, updates the HUD, applies damage, and saves the game. When rendering changes break jumping, this is why.

Split by **reason to change**, not by noun. Input handling changes when the control scheme changes; movement changes when the designer retunes feel; health changes when combat rules change. Those are three owners, three classes:

- `InputReader` produces intent (a struct or event), touching no gameplay state.
- `Movement` consumes intent and writes to the transform/rigidbody.
- `Health` owns hit points and raises events; the HUD and audio *listen*, they are not called by `Health`.

Rule of thumb for gameplay code: if a class needs a section-comment header to be navigable, it is at least two classes. Prefer composition (components/nodes/actors-with-components) over adding one more field to the god object.

### O - Open/Closed

*Extendable without editing.* In games the cheapest and most effective form of OCP is **data-driven design**, not inheritance. Every constant a designer might want to tweak - damage, cooldown, projectile count, curve, VFX reference - belongs in data, not in the class body. "Data" is a ladder: a `const` table in a script (solo project, readable diffs) → typed data objects built in code → editor-authored asset files (Unity `ScriptableObject`, Unreal `UDataAsset`/`UDataTable`, Godot `Resource`) when a non-programmer edits them. Start at the lowest rung that fits. Derived constants (`rate = max / (days * hours)`) are formulas, not tunables - they stay with the rules code.

The test: **adding the 30th enemy type should touch zero existing files** - one new data asset, and at most one small new class for genuinely new *behaviour*. If it means editing a `switch (enemyType)`, that switch is the bug. Replace it with a lookup table, a strategy object stored on the data asset, or polymorphic dispatch.

Watch for the "everyone always has this file open" symptom - `GameManager.cs`, `EnemyFactory.cpp`, the merge-conflict magnet. That file is failing OCP.

### L - Liskov Substitution

*A subclass must be usable through the base type without the caller knowing.* Two concrete smells:

1. **Type interrogation** - `if (entity is Tank)`, `Cast<ABoss>(Actor)`, `GetClass() == ...`, `node.is_in_group("flying")` inside code that already holds a base reference. If the base class asks what it is, the behaviour belongs on the base as a virtual/interface method.
2. **Broken contracts** - a `FlyingEnemy` that overrides `MoveTo` and quietly ignores the Y axis; a subclass that throws `NotSupportedException`; an override that requires the caller to call `Init()` first when the base does not.

Practical form: an override may accept *more* than the base and must promise *at least* as much. If a subclass cannot honour the base contract, it is not a subclass - use composition or a separate interface. Prefer shallow hierarchies (two levels at most); when the tree gets deeper than that, convert to components.

### I - Interface Segregation

*Small interfaces, one per capability.* Not `IEntity` with twenty methods that every enemy half-implements, but:

```
IDamageable   { TakeDamage(DamageInfo) }
IHealable     { Heal(amount) }
IInteractable { Interact(Actor instigator), CanInteract(Actor) }
ISaveable     { CaptureState(), RestoreState(state) }
ITargetable   { Position, IsAlive }
```

The payoff is readability at the call site: a turret that needs `IDamageable & ITargetable` declares exactly that, and you know at a glance what it touches. A door, a crate, and a boss can all be `IDamageable` without inheriting anything.

Heuristic: if implementers keep writing empty method bodies, the interface is too fat. If an interface's methods cluster into groups separated by blank lines, split it along those groups.

### D - Dependency Inversion

*Depend on abstractions; let the owner supply them.* Instead of a system constructing or hunting down its collaborators (`new AudioManager()`, `FindObjectOfType<SaveSystem>()`, `GetNode("/root/Game/Audio")`, a singleton `Instance` reference), it receives them - through a constructor, an installer, an `Init()` at spawn time, or an engine subsystem.

```
// Reaches out - hard to test, hard to swap, order-of-initialisation bugs
class CombatSystem { void Fire() { AudioManager.Instance.Play("shot"); } }

// Is handed what it needs - swap in a null/mock audio service for tests
class CombatSystem {
    readonly IAudioService audio;
    public CombatSystem(IAudioService audio) { this.audio = audio; }
    void Fire() { audio.Play(ShotCue); }
}
```

Wire everything once in a **composition root** (bootstrap scene, `GameInstance`, autoload, `main()`), not scattered at point of use. High-level gameplay rules should never `#include`/`using` a rendering, audio, or platform API directly.

The cheapest inversion is a **parameter**, not an interface: `now`, `dt`, and an injected random generator make time- and chance-dependent rules deterministic with zero new types. Reach for an interface + adapter only when a whole subsystem (audio, save, network) needs swapping. In a single-script project the root object *is* the hidden singleton - every function reading its shared fields is the same coupling as `Manager.Instance`, and the fix is the same: pass what the function reads, return what it writes.

## Applying this while writing new code

1. **Name the responsibilities first.** State them in one line each before writing the class. If the list has more than one entry, that is the file split.
2. **Push tunables into data.** Any number, curve, or asset reference a designer will touch goes in a data asset with sensible defaults.
3. **Depend on the smallest interface that does the job**, and take it as a parameter rather than fetching it.
4. **Communicate outward with events**, inward with calls. `Health` raises `Died`; it does not know about the HUD, the ragdoll, or the achievement system. This is what stops the dependency graph from becoming a mesh.
5. **Skip the abstraction when there is one implementation and no test seam.** Concrete first; extract when a second case appears (or when the class is genuinely hard to test without it).

Then say briefly *which* principle drove each non-obvious structural choice - one clause, e.g. "`Health` raises an event instead of calling the HUD so combat rules and UI change independently." Do not lecture.

## Applying this while reviewing or refactoring

Work in **small, behaviour-preserving steps**, highest pain first. Do not rewrite an entire subsystem in one pass unless asked. For each issue found, report it in this shape:

```
[Principle] Location - what is wrong
Why it hurts: <the concrete future bug or friction, in game terms>
Fix: <smallest change that removes the coupling>
```

Order findings by cost of leaving them, and be explicit when something is fine as-is - "this switch has three cases and lives in one file; leave it" is a valid finding. Name the *event* that would justify each deferred abstraction ("when the second enemy type is added"), not a date.

On a god file (thousands of lines, every function reading the same fields) do not guess at the first cut: build the section → fields-written and function → purity tables first (playbook Recipe 0). **If the project has no tests, the first refactor step is extracting the pure formulas into static functions and landing one assertion file** (Recipe 7) - components come after there is something that fails when they break.

`references/refactoring-playbook.md` has step-by-step recipes (read the god file, extract component, replace type check with polymorphism, replace switch with data table, introduce a seam, split a fat interface, break a singleton dependency, first test with no framework, split save/load without changing the format) and a fuller report template.

## Where NOT to apply this

These are not exceptions to be apologised for; they are the craft.

- **Hot paths.** Thousands of entities updated per frame want data-oriented layout and tight loops, not per-entity virtual dispatch or interface calls. Put the abstraction at the *system* boundary (one `IParticleSimulator`), and keep the inner loop concrete and branch-predictable. Never introduce per-frame allocation, LINQ, boxing of interface-typed structs, or reflection in an update loop for the sake of tidiness.
- **Jam / prototype / throwaway code.** Coupling is cheap when the code has a two-week lifespan. Say so and move on.
- **Engine-imposed shapes.** `MonoBehaviour`, `AActor`, and `Node` lifecycles are given. Work with them (see the engine references) rather than building a parallel framework to escape them.
- **Small, stable things.** A three-case `switch` on a damage type that has not changed in a year does not need a strategy pattern.
- **Long but linear builder and draw routines.** A 130-line function that declares a settings window or paints a chest pixel by pixel is not a god class; splitting it into five 25-line functions does not read better. Line-count rules apply to logic, not to declarative construction.
- **Solo-authored code with no test suite and no designer iteration.** Dial the ceremony down to: pure rules in plain classes, parameters instead of interfaces, one assertion file, signals only across real ownership boundaries. That is the whole list.

If a request explicitly asks for the quick hack, give the quick hack, and add at most one line noting the debt.

## Failure modes to actively avoid

AI-written game code drifts toward over-abstraction. Guard against:

- Interfaces with exactly one implementation, created reflexively.
- `AbstractEnemyFactoryProvider`-style naming; layers that only forward calls.
- A DI container hauled in for a project with six systems - a composition root and constructor parameters are enough.
- Inheritance chains three or more deep where components would do.
- Events everywhere, so no one can trace what happens when the player dies. Events are for crossing subsystem boundaries, not for talking to yourself.
- Refactoring past what was asked. If the fix is one extracted class, do not deliver a new architecture.
- Designing for the imagined 15th variant. An abstraction is forced into shape by the *second* real case; before that it is a guess. Two implementations validate a seam, three are not needed.
- Rendering code that mutates simulation state (spawning particles or ticking timers inside a draw callback). It makes every offscreen render path back up and restore state by hand.

Fewer, well-placed seams beat many shallow ones.

## Engine specifics

Read the file matching the project's engine before writing engine-facing code - each covers the idiomatic way to get these properties in that engine, the traps (Unity's inability to serialize plain interfaces, Unreal's `UInterface` boilerplate and subsystem-based injection, Godot's autoload / class-cache / `.tres`-can-contain-scripts pitfalls), and how to run a first test without a framework. Each file states the engine version it was verified against; if the project pins a newer one, prefer the project's own engine reference docs for API facts.

- `references/unity-csharp.md` - Unity / C# (MonoBehaviour, ScriptableObject, DOTS note)
- `references/unreal-cpp.md` - Unreal / C++ and Blueprints (Actor Components, UInterface, Subsystems, GAS)
- `references/godot.md` - Godot 4.5+ (Node vs RefCounted vs Resource, `@abstract`, procedural-draw projects, signals, testing, GDScript traps)
- `references/refactoring-playbook.md` - engine-agnostic refactoring recipes and the review report template

For an engine not listed, apply the principles directly; the playbook is engine-agnostic.
