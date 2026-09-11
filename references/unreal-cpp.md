# SOLID in Unreal Engine / C++

Written for Unreal Engine 5.x idioms (Actor Components, UInterface, Subsystems, GAS). Not re-verified against a specific point release - for API facts, prefer the project's own pinned engine docs.

Contents: [Actor Components](#actor-components-over-actor-inheritance) · [Data assets](#data-driven-with-data-assets-and-data-tables) · [UInterface](#uinterface-for-capability-interfaces) · [Subsystems as injection](#subsystems-the-engine-blessed-injection-point) · [Casting](#casting-is-the-lsp-smell) · [Delegates](#delegates-for-outward-communication) · [Gameplay Ability System](#gameplay-ability-system) · [Blueprints](#blueprint-boundaries) · [Performance](#performance-notes)

## Actor Components over Actor inheritance

Unreal's default gravity pulls toward deep `AActor` hierarchies: `ACharacter` → `AMyCharacter` → `AEnemyCharacter` → `ABossCharacter`. Every level down makes the classes harder to recombine, and the base accumulates fields only one descendant uses.

Prefer `UActorComponent` per responsibility - each one has a single reason to change and can be attached to a character, a turret, or a destructible crate alike:

```cpp
UCLASS(ClassGroup=(Combat), meta=(BlueprintSpawnableComponent))
class UHealthComponent : public UActorComponent
{
    GENERATED_BODY()
public:
    UPROPERTY(BlueprintAssignable) FOnDeath OnDeath;
    void ApplyDamage(const FDamageInfo& Info);

private:
    UPROPERTY(EditDefaultsOnly, Category="Health")
    TObjectPtr<const UHealthConfig> Config;   // data asset, not hard-coded numbers

    float Current = 0.f;
};
```

Keep the Actor as an assembly point: it owns components and wires them, and holds little logic itself. Two levels of inheritance is a good ceiling; past that, ask what should have been a component.

## Data-driven with Data Assets and Data Tables

OCP in Unreal is mostly asset work:
- `UPrimaryDataAsset` / `UDataAsset` for per-thing configuration (weapon stats, enemy loadouts, ability parameters).
- `UDataTable` with a `FTableRowBase` struct for tabular content designers edit in a spreadsheet.
- `TSoftObjectPtr` / `TSoftClassPtr` for references that should not force-load the whole dependency chain.
- `UCurveFloat` for tuning curves instead of magic constants.

```cpp
UCLASS(BlueprintType)
class UWeaponData : public UPrimaryDataAsset
{
    GENERATED_BODY()
public:
    UPROPERTY(EditDefaultsOnly) float Damage = 10.f;
    UPROPERTY(EditDefaultsOnly) float Cooldown = 0.25f;
    UPROPERTY(EditDefaultsOnly) TSubclassOf<UFireBehaviour> FireBehaviour;  // strategy
};
```

Adding a weapon becomes a new asset. If it needs new behaviour, it is one new `UFireBehaviour` subclass - never a new case in a `switch(WeaponType)`.

## UInterface for capability interfaces

Unreal's interface boilerplate is verbose but it is the right ISP tool - small, capability-shaped, implementable by C++ and Blueprint classes alike:

```cpp
UINTERFACE(MinimalAPI, Blueprintable)
class UDamageable : public UInterface { GENERATED_BODY() };

class IDamageable
{
    GENERATED_BODY()
public:
    UFUNCTION(BlueprintNativeEvent, BlueprintCallable, Category="Combat")
    void TakeDamage(const FDamageInfo& Info);
};
```

Call sites stay open to anything that implements it:

```cpp
if (IDamageable* Damageable = Cast<IDamageable>(HitActor))
{
    IDamageable::Execute_TakeDamage(HitActor, Info);
}
```

Store interface references as `TScriptInterface<IDamageable>` so both the object and the interface pointer are kept valid and GC-visible. Use `BlueprintNativeEvent` when Blueprints may override, `BlueprintImplementableEvent` when only Blueprints implement.

## Subsystems: the engine-blessed injection point

Instead of a hand-rolled singleton or `GetWorld()->GetAuthGameMode()` casts sprinkled through gameplay code, use subsystems - they have engine-managed lifetimes tied to a clear scope:

- `UGameInstanceSubsystem` - persists across level loads (save, telemetry, audio banks).
- `UWorldSubsystem` - per level (spawning director, wave manager).
- `ULocalPlayerSubsystem` - per local player (input mapping, per-player UI state).

```cpp
if (USaveSubsystem* Save = GetGameInstance()->GetSubsystem<USaveSubsystem>()) { ... }
```

That is still a lookup, so keep it at the edges. Better: components and plain classes take what they need as an injected pointer/interface set in `BeginPlay` or via `Initialize(...)`, and only the Actor or subsystem doing the assembling performs the lookup. Business logic that needs no `UObject` should live in plain C++ classes that a test can construct directly.

## Casting is the LSP smell

`Cast<AEnemyTank>(Actor)` inside code that already holds an `AActor*` or a base type is the Unreal spelling of the type-check violation. Fix by:

1. Moving the behaviour onto a virtual on the base, or
2. Asking for a capability interface instead of a concrete class (`Cast<IDamageable>` is fine - it asks *what can this do*, not *what is it*), or
3. Using Gameplay Tags to describe the object, and reacting to tags rather than classes.

Also watch for overrides that break the base contract - a `BeginPlay` override that forgets `Super::BeginPlay()` is a Liskov violation with real, painful symptoms.

## Delegates for outward communication

Use dynamic multicast delegates for events crossing subsystem boundaries so the emitter stays ignorant of listeners:

```cpp
DECLARE_DYNAMIC_MULTICAST_DELEGATE_OneParam(FOnDeath, AActor*, Victim);
```

`HealthComponent` broadcasts `OnDeath`; the ragdoll handler, HUD, quest tracker, and audio each subscribe. None of them appear in `HealthComponent`'s header - that is DIP working. Prefer non-dynamic `TMulticastDelegate` for C++-only paths (cheaper, no reflection); use dynamic when Blueprints must bind.

## Gameplay Ability System

GAS is, in effect, SOLID pre-packaged: abilities are separate objects (SRP), new abilities are new assets and classes (OCP), effects are data (`UGameplayEffect`), and attributes are decoupled from the abilities that modify them. If the project already uses GAS, add abilities and effects the GAS way rather than inventing a parallel system. If it does not, do not introduce GAS just to satisfy an architectural argument - it is a large commitment.

## Blueprint boundaries

A workable split: C++ owns systems, rules, and data structures; Blueprints own tuning, assembly, and presentation. Expose small, intention-revealing `UFUNCTION`s rather than dozens of raw setters - Blueprint graphs are the hardest place to refactor later, so keep logic out of them. A 200-node event graph is the Blueprint form of the god class.

## Performance notes

- Prefer per-component `SetComponentTickEnabled(false)` and event-driven updates over ticking everything; the cheapest virtual call is the one that never runs.
- Interface dispatch via `Execute_` for `BlueprintNativeEvent` goes through the reflection system and is far more expensive than a plain virtual call - do not put it in a per-frame path over many actors.
- Batch work in a subsystem that iterates its own compact array rather than having thousands of actors tick individually.
- `TObjectPtr` in `UPROPERTY` fields, raw pointers in local scope; keep GC-visible references correct before optimising anything.
