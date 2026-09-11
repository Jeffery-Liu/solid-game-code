# Refactoring Playbook (engine-agnostic)

Contents: [Smell index](#smell-index) · [Recipe 0: read the god file](#recipe-0-read-a-god-file-before-cutting-it) · [Recipe 1: extract component](#recipe-1-extract-a-component-from-a-god-class) · [Recipe 2: type check to polymorphism](#recipe-2-replace-a-type-check-with-polymorphism) · [Recipe 3: switch to data table](#recipe-3-replace-a-switch-with-data) · [Recipe 4: introduce a seam](#recipe-4-introduce-a-seam-for-an-engine-dependency) · [Recipe 5: split a fat interface](#recipe-5-split-a-fat-interface) · [Recipe 6: break a singleton dependency](#recipe-6-break-a-singleton-dependency) · [Recipe 7: first test with no framework](#recipe-7-the-first-test-in-a-project-that-has-none) · [Recipe 8: split save/load](#recipe-8-split-a-monolithic-saveload-without-changing-the-file-format) · [Review report template](#review-report-template) · [Triage](#triage-what-to-fix-first)

Every recipe is behaviour-preserving and small enough to verify by playing the game. Do one at a time; commit between them. If tests exist, run them after each step. If none exist, **Recipe 7 comes first**: extract the pure functions and land one assertion file before extracting components - the assertions are the safety net for everything after.

## Smell index

| Smell | Principle | Recipe |
|---|---|---|
| 500+ line class, section comments, everyone edits it | SRP | 0 then 1 |
| `if (x is Tank)` / `Cast<>` / `has_method` inside logic that holds a base type | LSP | 2 |
| `if item.name == "Diamond"` in draw/logic code when the item already has data fields | OCP | 3 |
| `switch (enemyType)` repeated in several files | OCP | 3 |
| Gameplay code calling engine/platform APIs directly (time, random, audio, input) | DIP | 4 |
| Implementers writing empty method bodies | ISP | 5 |
| `Manager.Instance` / `FindObjectOfType` / autoload deep in logic | DIP | 6 |
| Every function reads 20 fields of the root/main script | DIP | 6 (root-as-singleton) |
| Magic numbers designers keep asking you to change | OCP | 3 |
| Subclass override that throws, no-ops, or requires extra setup | LSP | 2 |
| Rendering code mutates simulation state (`_draw` / `OnGUI` spawns particles) | SRP | 1 (move to update) |
| One 100-line load function with inline migrations and backfills | SRP | 8 |
| No test can run without opening a window | — | 7 |
| A formula, a "switch", or a rule you cannot verify without playing for an hour | — | 7 |

## Recipe 0: read a god file before cutting it

Recipe 1 says "pick the most self-contained responsibility". In a 3,000-line file where every function reads 20 shared fields, you cannot see which one that is. Build three tables first; they take an hour and every later step reads from them.

1. **Sections → fields.** For each comment-section (or cluster of functions), list the fields it *writes*. A field written by two sections is a coupling point; a field written by one is that section's property.
2. **Functions → purity.** Mark each function as: pure already (only parameters in, value out) · one parameter from pure (reads one or two fields) · tangled (reads many fields, has side effects). The first two groups are Recipe 7 material and go first.
3. **Cross-cutting calls.** Count call sites of `save()`, `play_sound()`, `show_toast()`, `Time.now()`, `random()` inside rules code. These become parameters, injected references, or signals in Recipes 4 and 6.

Now pick: the responsibility with the fewest shared-field writes and the most "pure already" functions. That is the first extraction. Write the tables into the review report; they are the evidence for the sequencing.

## Recipe 1: extract a component from a god class

1. **Pick one responsibility**, ideally the most self-contained (audio, VFX, inventory - not the one tangled with everything). Use Recipe 0's tables.
2. **List the fields it touches.** If it touches fields owned by three other responsibilities, pick a different one first.
3. **Create the new class/component** with those fields, and move the methods over verbatim. Do not improve them yet.
4. **Leave a delegating method on the original** so all call sites still compile: `public void PlayFootstep() => audio.PlayFootstep();` Constants can be aliased the same way (`const MAX := Rules.MAX`) so the remaining 3,000 lines do not change in the same commit.
5. **Play the game / run tests.** Behaviour must be identical.
6. **Update call sites** to talk to the new component directly, then delete the delegating methods.
7. **Only now** clean up the extracted code.

Stop when each remaining class fits on a screen or two and has a name you can state without "and".

**In a zero-test project the first extraction is not a component.** It is the pure formulas → `static` functions on a plain class (Recipe 7). Components come after there is something that fails when they break.

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

**Cheapest form:** when the table already exists and code still branches on a *name* from it (`if row.name == "Diamond"`), add one field to the row (`fx = "fire"`) and branch on the field. Three lines, no new type.

**Where the data lives** is a graded choice - a `const` table in a script (solo project, readable diffs, read-only at runtime) → typed data objects built in code → editor-authored asset files (when a non-programmer edits them). Start at the lowest rung that fits; each can be exported into the next. **Do not move derived constants** (`RATE = MAX / (DAYS * HOURS)`) into a numeric table - they are formulas and travel with the rules code.

## Recipe 4: introduce a seam for an engine dependency

Use when rules code calls audio, rendering, input, networking, time, or randomness directly and you want to test or swap it.

0. **Try parameter injection first.** `now`, `dt`, `today`, `hours_offline` become parameters; randomness becomes an injected `RandomNumberGenerator` / `System.Random` field seeded by the owner. No interface, no adapter, and the function is now deterministic in a test. This covers time and random in almost every project - stop here if it does.
1. Define a narrow interface with only the methods this caller uses - `IAudioService { Play(cue) }`, not the whole audio API.
2. Write a thin adapter implementing it by forwarding to the engine API. No logic in the adapter.
3. Change the caller to hold the interface, supplied via constructor or an `Init`.
4. Wire the real adapter in the composition root.
5. For tests, pass a no-op or recording implementation.

Keep the interface at a **system** boundary, one per subsystem. Do not create an interface per class. **Do not wrap engine-imposed shapes** (window setup, tray icon, process launching) that have one implementation and nothing to test - leave those calls where they are.

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

**The root-as-singleton variant.** A single-script project has no `Instance`, but the root/main object *is* the global: every function reads and writes its shared fields through `self`. Same recipe, smaller steps - for one function at a time, replace the fields it reads with parameters and the fields it writes with a return value. When a function no longer touches `self`, it can become `static` and move out.

## Recipe 7: the first test in a project that has none

Goal: one file, one command, runs in under a second, fails when a formula breaks. No framework.

1. **Pick the formulas** (Recipe 0, "pure already" and "one parameter from pure"): progression rates, damage/wear, daily caps, offline catch-up, loot weights, pity counters, time-window rules, save-format migrations.
2. **Make them static** on a plain class with explicit parameters: `now`, `rng`, and the two or three state values they read. Leave a one-line delegating method behind (Recipe 1 step 4).
3. **Write the assertions that already matter:** boundaries of every threshold; "the cap is really a cap" (`gain(13h) == gain(12h)`); a seeded RNG over 10,000 draws lands within tolerance of the table weights; pity fires exactly at N; the migration function on old / new / null input; and any build-blocking invariant (`assert(DEBUG_SPEED == 1.0)`).
4. **One runner script** the CI or a pre-commit hook can call; the engine references show the zero-addon shape for each engine. Exit non-zero on failure.
5. Now Recipes 1–6 have a net.

Do not write a per-function suite, do not add fixtures, do not install a framework yet. Upgrade to gdUnit4 / NUnit / Automation Spec when the assertion file passes ~30 asserts or needs parameterised cases.

## Recipe 8: split a monolithic save/load without changing the file format

Player saves are the highest-risk refactor area: a mistake deletes weeks of progress. Keep the on-disk format frozen while you split the code.

1. **Capture a fixture**: copy a real save file (old version too, if migrations exist) into `tests/fixtures/`.
2. **Split by owner, not by section**: `load_shell()` / `load_progress()` / `load_<entity>()` each reading the keys their owner writes; the old `load()` becomes three calls. Same for `save()`. Function boundaries move; key names and file layout do not.
3. **Pull side effects out of `save()`** (`state = capture()` hidden inside the writer) into an explicit call at the top - a save must never mutate what it saves.
4. **Make capture/restore a full round-trip**: every field that any screenshot/thumbnail/undo path backs up by hand belongs in the snapshot. When `restore(capture())` is the identity, the hand-rolled backup dances can be deleted.
5. **Diff test**: load the fixture, save it, `diff` against the fixture - byte-identical except timestamps. Add this to Recipe 7's runner.
6. **Only then** version the format: add `schema`, keep the old loader path, default missing keys, and migrate forward in one function per version.

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
- `BuildSettingsWindow()` - 130 lines, but linear declarative UI construction; splitting it into
  five 25-line functions would not read better. Builder and draw routines are exempt from the
  line-count rule.
- Per-pixel material string compares inside `_draw` - hot path, eight values, one file. Leave it.

## Deferred until <the second case exists>
- `Curio` base class / `AgingModel` strategy - one implementation today; cut it when the walnut
  arrives and the seam has a real shape.
```

Always include "Fine as-is" and "Deferred until". Together they tell the reader what was considered and deliberately left alone, and they stop a review from reading as a demand for a rewrite. Name the *event* that would trigger a deferred abstraction ("when the second enemy type is added"), not a date.

## Triage: what to fix first

Order by **change frequency × pain**, not by principle purity:

1. Files with the most merge conflicts or the most recent commits - that is where coupling costs real time. On a solo project there are no merge conflicts; use `git log --oneline -- <file>` over the last 20 commits and count which section each touched.
2. The class implicated in recurring bugs.
3. The system a designer is currently iterating on (data-driving it pays back immediately).
4. Anything blocking a test you actually want to write - on a zero-test project this outranks everything above.

A good yardstick for "is this seam worth cutting now": **can I write one assertion today because of it?** If yes, cut it. If the only beneficiary is a hypothetical second implementation, defer it and say what event would change the answer.

Everything else can wait. Untouched code that is ugly but stable costs nothing.
