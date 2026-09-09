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
```

The engine references load on demand — only when the code at hand is for that
engine — so a trigger doesn't pull all of it into context at once.

## Supported engines

Works with any engine — the principles and the refactoring playbook are
engine-agnostic. Dedicated, idiomatic guidance ships for:

- **Unity** (C#) — MonoBehaviour, ScriptableObject, DOTS
- **Unreal** (C++ / Blueprints) — Actor Components, UInterface, Subsystems, GAS
- **Godot** (4.x, GDScript / C#) — nodes, Resources, signals, autoload traps

For any other engine (Bevy, custom, etc.) it applies the principles directly.

## Install

Requires [Claude Code](https://claude.com/claude-code) (CLI, desktop app, or IDE
extension).

1. Clone this repo (or download it via **Code → Download ZIP**, or from any
   release's *Source code* archive).

2. Copy `SKILL.md` and the `references/` folder into your skills directory —
   those two are all the skill needs:
   - Project-level: `.claude/skills/solid-game-code/`
   - User-level (all projects): `~/.claude/skills/solid-game-code/`

3. That's the whole install — no build step, no dependencies. Claude Code
   auto-discovers `SKILL.md` on the next session.

## Try it

Start Claude Code in a project and type any of these — the skill triggers on
intent, you never have to name it:

```
add a dash ability to my player
make an enemy that shoots at the player
this PlayerController is a mess, clean it up
review this class for coupling problems
```

You should see it split responsibilities, push tunables into data assets, and
reach for the matching engine reference — while deliberately *not* abstracting
hot paths or throwaway code.
