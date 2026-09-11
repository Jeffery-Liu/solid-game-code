# SOLID in Unity / C#

Written for Unity 2022 LTS / Unity 6 idioms (MonoBehaviour, ScriptableObject, Entities 1.x). Not re-verified against a specific patch release - for API facts, prefer the project's own pinned engine docs.

Contents: [Component split](#splitting-a-god-monobehaviour) · [Data-driven with ScriptableObjects](#data-driven-design-with-scriptableobjects) · [Interfaces & serialization](#interfaces-and-unitys-serializer) · [Dependency injection](#dependency-injection-without-a-framework) · [Events](#events-without-spaghetti) · [Type checks](#replacing-type-checks) · [Testing seams](#testing-seams) · [Performance](#performance-notes) · [DOTS](#dots--ecs)

## Splitting a god MonoBehaviour

Unity already gives you composition - use it. A `PlayerController` that grew to 800 lines becomes several `MonoBehaviour`s on the same GameObject, each with one reason to change:

```csharp
[RequireComponent(typeof(Rigidbody))]
public sealed class Movement : MonoBehaviour
{
    [SerializeField] MovementConfig config;   // designer-tunable data
    Rigidbody body;
    MoveIntent intent;                        // written by the input component

    public void SetIntent(in MoveIntent value) => intent = value;

    void Awake() => body = GetComponent<Rigidbody>();
    void FixedUpdate() => body.AddForce(intent.Direction * config.Acceleration);
}
```

Guidance:
- Wire sibling components in `Awake` via `GetComponent`, and cache the result. Never `GetComponent` in `Update`.
- Use `[RequireComponent]` to make the dependency explicit and impossible to forget in the editor.
- Prefer one component pushing data to another (`SetIntent`) over components polling each other - it keeps the direction of dependency visible.
- `sealed` by default on MonoBehaviours; it documents intent and helps the JIT devirtualize.

Do not split so far that a simple pickup needs five components. A component per *reason to change*, not per method.

## Data-driven design with ScriptableObjects

`ScriptableObject` is the main OCP tool in Unity. Stats, curves, VFX and audio references, spawn tables - all data assets, so a new enemy is a new `.asset` file rather than a code change.

```csharp
[CreateAssetMenu(menuName = "Combat/Weapon")]
public sealed class WeaponConfig : ScriptableObject
{
    public float Damage = 10f;
    public float Cooldown = 0.25f;
    public AnimationCurve SpreadOverHeat = AnimationCurve.Linear(0, 0, 1, 1);
    public FireBehaviour FireBehaviour;   // polymorphic strategy, see below
}
```

For behaviour that genuinely differs (hitscan vs projectile vs beam), make the strategy itself a ScriptableObject subclass and reference it from the config:

```csharp
public abstract class FireBehaviour : ScriptableObject
{
    public abstract void Fire(FireContext context);
}

public sealed class HitscanFire : FireBehaviour { public override void Fire(FireContext c) { /* ... */ } }
```

Now `WeaponSystem` never grows a `switch` - it calls `config.FireBehaviour.Fire(context)`. Adding a beam weapon means one new class and one new asset, and touches nothing existing.

**Caveat:** ScriptableObject state persists in the editor between play sessions and is shared across all instances. Treat them as immutable configuration. Anything mutable per-instance (current ammo, heat) lives on the runtime component, not on the asset.

## Interfaces and Unity's serializer

Unity cannot serialize a plain interface field, which pushes people back toward concrete types. Options, in order of preference:

1. `[SerializeReference]` for plain C# objects implementing an interface (polymorphic, serialized inline).
2. Serialize the concrete `MonoBehaviour`/`ScriptableObject` field and expose it as the interface in code.
3. Serialize a `GameObject`/`Component` and resolve with `GetComponent<IDamageable>()` in `Awake` - Unity's `GetComponent` does work with interfaces.

Small capability interfaces are the point:

```csharp
public interface IDamageable { void TakeDamage(in DamageInfo info); }
public interface ITargetable { Vector3 Position { get; } bool IsAlive { get; } }
```

A projectile then works on anything damageable - the player, a barrel, a destructible wall - with no shared base class:

```csharp
void OnTriggerEnter(Collider other)
{
    if (other.TryGetComponent(out IDamageable damageable))
        damageable.TakeDamage(new DamageInfo(config.Damage, transform.forward));
}
```

`TryGetComponent` avoids the allocation `GetComponent` can cause on failure - use it in collision paths.

## Dependency injection without a framework

Avoid `FindObjectOfType`, `GameObject.Find`, and ambient `static Instance` singletons in gameplay classes. They hide dependencies, break at scene-load time, and make tests impossible.

Plain composition root - a single bootstrap MonoBehaviour that builds services and hands them out:

```csharp
public sealed class GameBootstrap : MonoBehaviour
{
    [SerializeField] AudioService audioService;
    [SerializeField] SaveService saveService;
    [SerializeField] PlayerSpawner spawner;

    void Awake() => spawner.Init(audioService, saveService);
}
```

For objects spawned at runtime, pass dependencies in an explicit `Init(...)` immediately after `Instantiate`, or use a factory that closes over them. Prefer constructor injection for plain C# classes (systems, services, rules) - keep as much logic as possible in plain classes that a test can `new` up without a scene.

If the project already uses VContainer or Zenject, follow its conventions. Do not introduce a container into a project that does not have one just to satisfy DIP; a composition root is enough for most games.

An acceptable middle ground: one static service locator, accessed only in the composition layer and in `Init` calls, never deep inside gameplay logic.

## Events without spaghetti

Use plain C# events for one-to-many notifications across subsystem boundaries:

```csharp
public sealed class Health : MonoBehaviour
{
    public event Action<DamageInfo> Damaged;
    public event Action Died;
    // Health knows nothing about the HUD, ragdolls, VFX, or achievements.
}
```

Always unsubscribe in `OnDisable`/`OnDestroy` - leaked handlers on destroyed objects are one of the top sources of Unity's `MissingReferenceException`. `UnityEvent` is fine for designer-wired connections in the inspector, but prefer C# events for code-to-code because they are refactor-safe and cheaper.

ScriptableObject event channels (a `GameEvent` asset that listeners register with) decouple scenes nicely, but overused they make call flow untraceable. Reserve them for genuinely cross-scene signals.

## Replacing type checks

```csharp
// LSP violation
if (enemy is FlyingEnemy flying) flying.Descend();
else enemy.MoveTo(target);

// Polymorphic
enemy.MoveTo(target);   // FlyingEnemy.MoveTo descends first, and honours the base contract
```

If the caller needs a capability only some enemies have, that is an interface question, not a type-check question: `if (enemy is IStunnable stunnable) stunnable.Stun(2f);` is acceptable - it asks about a capability, not a concrete class.

## Testing seams

The clearest signal that DIP is satisfied in Unity: the rules can be tested in Edit Mode with no scene.

```csharp
[Test]
public void Armour_reduces_incoming_damage()
{
    var rules = new DamageRules(new FixedRandom(0.5f));    // injected, deterministic
    Assert.AreEqual(5, rules.Compute(damage: 10, armour: 0.5f));
}
```

Push arithmetic, state machines, and rules into plain classes; leave MonoBehaviours as thin adapters that read input, call the rules, and apply results to the scene. Inject `Time.deltaTime`, randomness, and clocks rather than reading them statically inside rules.

## Performance notes

- Interface calls prevent inlining and cost a virtual dispatch. Fine for dozens of objects per frame; a problem for thousands. Move the abstraction up to the system level and iterate concrete arrays inside.
- Interface-typed structs box. Never pass a struct through an interface in a per-frame path.
- No LINQ, no `foreach` over interface-typed collections, no closures allocated per frame in `Update`.
- Cache component lookups in `Awake`. `GetComponent` in a loop is a known frame-time killer.
- Events allocate on subscribe, not invoke - safe in the hot path once wired.

## DOTS / ECS

If the project uses Entities, the principles map differently: SRP becomes one system per behaviour, OCP becomes new components/archetypes rather than new subclasses, and DIP is largely handled by systems querying components rather than referencing each other. Do not force interfaces or inheritance into `IJobEntity` code - that is exactly the hot path the main SKILL warns about. Keep abstraction at the authoring/baking layer.
