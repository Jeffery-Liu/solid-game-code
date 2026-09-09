# Refactoring Playbook (engine-agnostic)

Contents: [Smell index](#smell-index) · [Recipe 1: extract component](#recipe-1-extract-a-component-from-a-god-class) · [Recipe 2: type check to polymorphism](#recipe-2-replace-a-type-check-with-polymorphism) · [Recipe 3: switch to data table](#recipe-3-replace-a-switch-with-data) · [Recipe 4: introduce a seam](#recipe-4-introduce-a-seam-for-an-engine-dependency) · [Recipe 5: split a fat interface](#recipe-5-split-a-fat-interface) · [Recipe 6: break a singleton dependency](#recipe-6-break-a-singleton-dependency) · [Review report template](#review-report-template) · [Triage](#triage-what-to-fix-first)

Every recipe is behaviour-preserving and small enough to verify by playing the game. Do one at a time; commit between them. If tests exist, run them after each step; if not, the safety net is "the game still plays the same", so keep steps small enough that this is a meaningful check.

## Smell index

| Smell | Principle | Recipe |
|---|---|---|
| 500+ line class, section comments, everyone edits it | SRP | 1 |
| `if (x is Tank)` / `Cast<>` / `has_method` inside logic that holds a base type | LSP | 2 |
| `switch (enemyType)` repeated in several files | OCP | 3 |
| Gameplay code calling engine/platform APIs directly | DIP | 4 |
| Implementers writing empty method bodies | ISP | 5 |
| `Manager.Instance` / `FindObjectOfType` / autoload deep in logic | DIP | 6 |
| Magic numbers designers keep asking you to change | OCP | 3 |
| Subclass override that throws, no-ops, or requires extra setup | LSP | 2 |

## Recipe 1: extract a component from a god class

1. **Pick one responsibility**, ideally the most self-contained (audio, VFX, inventory - not the one tangled with everything).
2. **List the fields it touches.** If it touches fields owned by three other responsibilities, pick a different one first.
3. **Create the new class/component** with those fields, and move the methods over verbatim. Do not improve them yet.
4. **Leave a delegating method on the original** so all call sites still compile: `public void PlayFootstep() => audio.PlayFootstep();`
5. **Play the game / run tests.** Behaviour must be identical.
6. **Update call sites** to talk to the new component directly, then delete the delegating methods.
7. **Only now** clean up the extracted code.

Stop when each remaining class fits on a screen or two and has a name you can state without "and".

## Recipe 2: replace a type check with polymorphism

1. Find every branch that switches on the concrete type of the same base reference.
2. Name the *behaviour* the branches differ in - "how this enemy approaches a target", not "is this a flier".
3. Add a virtual/abstract method with that name to the base type, with the common case as the default implementation.
4. Move each branch's body into the corresponding subclass override.
5. Replace the branching call site with the single polymorphic call.
6. Verify each override honours the base contract: same preconditions or weaker, same postconditions or stronger, no new required setup calls, no silently ignored parameters.

If a subclass cannot honour the contract, it should not be a subclass - convert it to a sibling with a shared interface, or to a component.

## Recipe 3: replace a switch with data

1. Identify the axis of variation (enemy type, weapon type, upgrade tier).
2. Create a config type holding every value the switch produced.
3. Create one config instance per case, filling in the values from the switch arms.
4. Replace the switch with a lookup: the entity holds a reference to its config, or a table maps id → config.
5. If the arms differ in *behaviour*, not just values, put a strategy object on the config (see the engine reference for the idiomatic form) and call through it.
6. Delete the enum if nothing else needs it - often the enum was the coupling.

Result: the 30th case is a new asset, authored by a designer, touching no code.

## Recipe 4: introduce a seam for an engine dependency

Use when rules code calls audio, rendering, input, networking, time, or randomness directly and you want to test or swap it.

1. Define a narrow interface with only the methods this caller uses - `IAudioService { Play(cue) }`, not the whole audio API.
2. Write a thin adapter implementing it by forwarding to the engine API. No logic in the adapter.
3. Change the caller to hold the interface, supplied via constructor or an `Init`.
4. Wire the real adapter in the composition root.
5. For tests, pass a no-op or recording implementation.

Keep the interface at a **system** boundary, one per subsystem. Do not create an interface per class.

## Recipe 5: split a fat interface

1. Group the methods by which clients actually call them - blank-line clusters in the declaration are usually the seams.
2. Create one interface per group, named after the capability (`IDamageable`, `ISaveable`, `ITargetable`).
3. Have the fat interface temporarily inherit all of them so nothing breaks.
4. Migrate call sites to depend on the narrow interface they need.
5. Migrate implementers to implement only the narrow interfaces they honour.
6. Delete the fat interface.

Empty method bodies disappearing is the signal it worked.

## Recipe 6: break a singleton dependency

1. Add a field for the dependency, typed as a narrow interface, to the class using it.
2. Add a constructor parameter or `Init` parameter that sets it.
3. Replace `Singleton.Instance.Foo()` with `dependency.Foo()` inside the class.
4. At the call site that creates this object, pass `Singleton.Instance` - the singleton still exists, but only the composition layer knows about it.
5. Repeat outward. Eventually the singleton is referenced in one place and can become a plain object owned by the bootstrap.

This is incremental on purpose: deleting a singleton in one pass across a real codebase is how weekends get lost.

## Review report template

```markdown
## Summary
<2-3 sentences: overall shape of the code, the one thing worth fixing first.>

## Findings

### 1. [SRP] PlayerController.cs:1-812 - handles input, movement, combat, UI, and saving
**Why it hurts:** every combat tweak risks breaking movement, and two people cannot work on
the player at once without conflicts.
**Fix:** extract `InputReader` and `Health` first (least tangled); leave movement in place for now.
**Effort:** ~1 hour, mechanical.

### 2. [OCP] WeaponSystem.cs:88 - switch over WeaponType, duplicated in UI and audio
**Why it hurts:** adding a weapon means finding all three switches; the audio one is already out of sync.
**Fix:** move damage/cooldown/cue into a WeaponConfig asset, look up by reference.
**Effort:** ~2 hours, touches designer workflow - confirm before doing it.

## Fine as-is
- `DamageType` switch in `DamageRules.cs` - three cases, one file, stable. Leave it.
- Concrete `Rigidbody` use in `Movement` - engine-imposed, no benefit to abstracting.
```

Always include the "fine as-is" section. It tells the reader what was considered and deliberately left alone, and it stops a review from reading as a demand for a rewrite.

## Triage: what to fix first

Order by **change frequency × pain**, not by principle purity:

1. Files with the most merge conflicts or the most recent commits - that is where coupling costs real time.
2. The class implicated in recurring bugs.
3. The system a designer is currently iterating on (data-driving it pays back immediately).
4. Anything blocking a test you actually want to write.

Everything else can wait. Untouched code that is ugly but stable costs nothing.
