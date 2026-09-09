# solid-game-code

A Claude skill that applies the SOLID principles to game code the way experienced
game programmers actually do — decoupled, data-driven, and testable, without
choking the hot path with needless abstraction.

Inspired by [SOLID Principles for Game Developers](https://www.gamedeveloper.com/programming/solid-principles-for-game-developers),
extended into concrete, engine-specific guidance and refactoring recipes.

## What it does

Triggers when you write, review, refactor, or plan game code — even when you
never say "SOLID". A request like *"add a dash ability"*, *"make an enemy that
shoots"*, or *"clean up my PlayerController"* is exactly when it kicks in. It
favors well-placed seams over ceremony and knows where **not** to abstract
(hot paths, jam code, stable three-case switches).

## Layout

```
SKILL.md                              main guidance (loaded on trigger)
references/
  unity-csharp.md                     MonoBehaviour, ScriptableObject, DOTS
  unreal-cpp.md                       Actor Components, UInterface, Subsystems, GAS
  godot.md                            nodes, Resources, signals, autoload traps
  refactoring-playbook.md             engine-agnostic recipes + review template
solid-game-code.skill                 packaged skill (zip of the above)
```

The engine references load on demand — only when the code at hand is for that
engine — so a trigger doesn't pull all of it into context at once.

## Install

Unpack `solid-game-code.skill` (a zip) into your skills directory, or point your
skills loader at this folder directly.
