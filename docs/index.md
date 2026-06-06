# Layered State Machine — Documentation

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Creating a State](#creating-a-state)
3. [StateEntry](#stateentry)
4. [Subscribing vs Setting States](#subscribing-vs-setting-states)
5. [State Bindings](#state-bindings)
6. [Built-in Bindings](#built-in-bindings)
7. [Built-in Conditions](#built-in-conditions)
8. [Illegal States & Weight System](#illegal-states--weight-system)
9. [Cooldown & Buffer](#cooldown--buffer)
10. [Managing Bindings at Runtime](#managing-bindings-at-runtime)
11. [StateQuery & Events](#statequery--events)
12. [The Inspector](#the-inspector)
13. [API Reference](#api-reference)

---

## Core Concepts

Layered State Machine is built around four core blocks:

| Block | What it is |
|---|---|
| `State` | A behaviour your object can be in |
| `StateEntry` | Configuration for how a state activates and behaves |
| `StateBinding` | A condition that decides whether a state should be active |
| `StateMachine` | The MonoBehaviour component that runs everything |

Multiple states can be active simultaneously — this is not a traditional "one state at a time" machine. A character can be `Focused`, `Crouching`, and `Reloading` at the same time, as long as none of them conflict with each other.

---

## Creating a State

Inherit from `State` and implement the required members:

```csharp
public class FocusedState : State
{
    // priority used in conflict resolution
    public override float stateWeight => 0.5f;

    // states that cannot be active at the same time as this one
    public override List<State> illegalStates => new List<State>
    {
        new SprintingState()
    };

    public override void OnAdd()     => Debug.Log("Focused started.");
    public override void OnRemoval() => Debug.Log("Focused ended.");
    public override void OnUpdate()  => Debug.Log("Focusing...");
}
```

**Lifecycle methods:**

| Method | When it's called |
|---|---|
| `OnAdd()` | Once, when the state becomes active |
| `OnRemoval()` | Once, when the state is deactivated |
| `OnUpdate()` | Every Unity Update while the state is active |

`stateWeight` is required and determines who wins when two conflicting states compete. See [Illegal States & Weight System](#illegal-states--weight-system).

`illegalStates` is optional. Leave it empty if this state has no conflicts.

---

## StateEntry

`StateEntry` defines how a state behaves when subscribed to the machine.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... }, // what triggers this state
    cooldownDuration = 0f,                             // seconds before it can reactivate
    bufferDuration   = 0f,                             // input buffer window in seconds
    stateDuration    = 0f,                             // how long the state stays active (0 = forever)
    singularUse      = false,                          // remove after one Update tick?
    destroyOnRemoval = false,                          // permanently remove on deactivation?
    respectWeight    = true,                           // participate in illegal state weight resolution?
    oneWay           = false,                          // once activated, never removed by bindings?
    overwrite        = false,                          // overwrite if the same state is already subscribed?
}
```

> You don't need to set `state`, `elapsedCooldown`, `elapsedBuffer`, or `elapsedDuration` — the state machine manages those internally.

---

## Subscribing vs Setting States

There are two ways to activate a state:

### `SubscribeState` — event driven

Ties a state to bindings. The machine evaluates them every Update and activates or deactivates the state automatically.

```csharp
stateMachine.SubscribeState(
    new FocusedState(),
    new StateEntry
    {
        bindings = new List<StateBinding>
        {
            new InputBinding(input.actions["RightClick"])
        }
    }
);
```

Use this for states that should toggle automatically based on input or game conditions.

If a state with the same type is already subscribed, the call is discarded by default. Set `overwrite = true` on the `StateEntry` to replace the existing subscription instead.

### `SetState` — manual

Activates a state directly without any binding. The second parameter controls whether the weight system applies.

```csharp
stateMachine.SetState(new FocusedState(), respectWeight: true);
```

Use this for states you want full manual control over, such as a cutscene state or a death state.

---

## State Bindings

A `StateBinding` is a condition evaluated every Update to decide whether a subscribed state should be active. To create a custom binding, inherit from `StateBinding` and implement two methods:

```csharp
public class HealthBinding : StateBinding
{
    public override bool Evaluate()
    {
        return machine.GetComponent<Health>().current < 20f;
    }

    public override string Transcript()
    {
        return "Health below 20.";
    }
}
```

`machine` is automatically available inside any binding — it's the `StateMachine` the binding is attached to.

### AND Logic — BindingFolder

To require multiple conditions at once, wrap them in a `BindingFolder`:

```csharp
new BindingFolder(new List<StateBinding>
{
    new InputBinding(input.actions["RightClick"]),
    new InputBinding(input.actions["Aim"])
})
```

This only returns `true` if **all** bindings inside evaluate to `true`.

---

## Built-in Bindings

### `InputBinding`

Activates when a Unity Input System action is held down.

```csharp
new InputBinding(input.actions["RightClick"])
// Transcript: "RightClick is Pressed."
```

Requires the **Unity Input System** package.

---

### `ConditionBinding`

Delegates evaluation to a `Condition` class you implement. Useful when the logic is complex or needs to be reused across multiple states.

```csharp
public class LowHealthCondition : Condition
{
    public override bool Evaluate(StateMachine machine)
    {
        return machine.GetComponent<Health>().current < 20f;
    }

    public override string Transcript() => "Health below 20.";
}

// usage
new ConditionBinding(new LowHealthCondition())
```

Prefer `ConditionBinding` over a raw `StateBinding` subclass when the same condition is shared across multiple states or projects.

---

### `BindingFolder`

Groups multiple bindings together with AND logic — all must be true for the state to activate.

```csharp
new BindingFolder(new List<StateBinding>
{
    new InputBinding(input.actions["RightClick"]),
    new InputBinding(input.actions["Aim"])
})
// Transcript: "RightClick is Pressed., Aim is Pressed."
```

---

## Built-in Conditions

### `AlwaysCondition`

A built-in `Condition` that always returns `true`. Useful for states that should activate unconditionally as soon as they are subscribed.

```csharp
new ConditionBinding(new AlwaysCondition())
```

---

### `ContainStateCondition`

A built-in `Condition` that checks whether a specific state is currently active on the machine.

```csharp
// activates when FocusedState is active
new ConditionBinding(new ContainStateCondition(new FocusedState()))

// activates when FocusedState is NOT active
new ConditionBinding(new ContainStateCondition(new FocusedState(), desiredResult: false))
```

---

## Illegal States & Weight System

Illegal states are states that **cannot be active at the same time**. When two conflicting states compete, `stateWeight` decides the winner — the higher weight wins.

```csharp
public class FocusedState : State
{
    public override float stateWeight => 0.5f;

    public override List<State> illegalStates => new List<State>
    {
        new SprintingState()
    };
}

public class SprintingState : State
{
    public override float stateWeight => 0.6f;
}
```

If `SprintingState` tries to activate while `FocusedState` is active:
- `SprintingState` (0.6) **>** `FocusedState` (0.5) → Sprinting wins, Focused is removed.
- If the weights were reversed, Sprinting would be blocked entirely.

> You only need to declare the conflict on one side. If either state lists the other as illegal, they can never be active at the same time.

### Bypassing the weight system

Set `respectWeight = false` on a `StateEntry` to make a state activate regardless of conflicts. It will never be blocked or removed by weight resolution.

```csharp
new StateEntry
{
    bindings      = new List<StateBinding> { ... },
    respectWeight = false
}
```

Good for: death states, cutscene states, anything that must always win.

---

## Cooldown & Buffer

### `cooldownDuration`

Sets a minimum number of seconds that must pass after a state deactivates before it can activate again.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... },
    cooldownDuration = 2f
}
```

Good for: abilities, attacks, anything with a recharge time.

### `bufferDuration`

Sets an input buffer window. If a binding evaluates to `true` while the state is on cooldown, the machine stores the input for `bufferDuration` seconds. When the cooldown expires, the state activates immediately without requiring the player to press again.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... },
    cooldownDuration = 0.5f,
    bufferDuration   = 0.15f
}
```

Good for: attacks, rolls, any action where early input should still register.

### `stateDuration`

The state automatically removes itself after this many seconds. Set to `0` (default) for no automatic removal.

```csharp
new StateEntry
{
    bindings      = new List<StateBinding> { ... },
    stateDuration = 0.4f
}
```

Best paired with `cooldownDuration` to prevent immediate reactivation, or `oneWay` for a state that triggers once and runs for a fixed time.

Good for: attacks, rolls, timed buffs, ability windows.

### `singularUse`

When `true`, the state activates, runs `OnUpdate` exactly once, then removes itself from `currentStates`. It stays in `subscribedStates` and can activate again the next time the binding condition becomes true.

```csharp
new StateEntry
{
    bindings    = new List<StateBinding> { ... },
    singularUse = true
}
```

Good for: single-frame impulses, one-tick events, fire-and-forget effects.

### `destroyOnRemoval`

When `true`, the state permanently removes itself from `subscribedStates` the moment it deactivates. It will never activate again even if the binding condition becomes true.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... },
    destroyOnRemoval = true
}
```

Can be combined with `singularUse` — the state activates once, runs one update tick, then is gone permanently.

Good for: intro animations, first-time tutorials, single-use pickups.

### `oneWay`

When `true`, once the state is activated by a binding it will never be deactivated by bindings. The state stays active until explicitly removed via `RemoveState`, `ClearAllStates`, or `stateDuration` expiry.

```csharp
new StateEntry
{
    bindings = new List<StateBinding> { ... },
    oneWay   = true
}
```

Good for: death states, permanent unlocks, mode switches that should never auto-revert.

> `oneWay` only suppresses binding-based removal. `stateDuration`, `singularUse`, and manual `RemoveState` calls still work normally.

### `overwrite`

When `true`, subscribing a state whose type is already registered will replace the existing `StateEntry` instead of being silently discarded.

```csharp
new StateEntry
{
    bindings  = new List<StateBinding> { ... },
    overwrite = true
}
```

Good for: reconfiguring a state at runtime without having to manually unsubscribe it first.

---

## Managing Bindings at Runtime

You can add or remove bindings from a subscribed state while the game is running.

### `AddBindings`

```csharp
stateMachine.AddBindings(
    new FocusedState(),
    new List<StateBinding> { new InputBinding(input.actions["Aim"]) }
);
```

### `RemoveBindings`

```csharp
stateMachine.RemoveBindings(
    new FocusedState(),
    new List<StateBinding> { existingBinding }
);
```

> To remove a specific binding you need a reference to the original binding instance, not a new one.

### `UnsubscribeState`

Removes the state from the machine entirely. If it is currently active, it will also be deactivated.

```csharp
stateMachine.UnsubscribeState(new FocusedState());
```

### `ClearAllStates`

Deactivates all currently active states. Pass `true` to also clear all subscribed states.

```csharp
stateMachine.ClearAllStates(clearSubscribedStates: false); // deactivate all, keep subscriptions
stateMachine.ClearAllStates(clearSubscribedStates: true);  // full reset
```

Good for: scene transitions, player death, respawn.

---

## StateQuery & Events

Every state activation, deactivation, and failure fires an `OnStateChanged` event with a `StateQuery` value explaining what happened.

```csharp
stateMachine.OnStateChanged += (state, query) =>
{
    Debug.Log($"{state.GetType().Name}: {query}");
};
```

### `StateQuery` values

| Value | When it fires |
|---|---|
| `SuccessfullyAdded` | State was activated |
| `SuccessfullyRemoved` | State was deactivated |
| `RemovedByWeight` | State was removed because a higher-weight conflicting state won |
| `AlreadyActive` | `SetState` was called but the state was already active |
| `NotActive` | `RemoveState` was called but the state was not active |
| `OnCooldown` | `SetState` was blocked because the cooldown has not expired |
| `BlockedByWeight` | `SetState` was blocked by a higher-weight conflicting state |

### Subscribe events

`OnSubscribeChange` fires when a state is subscribed or unsubscribed, and includes the full `StateEntry`.

```csharp
stateMachine.OnSubscribeChange += (state, query, entry) =>
{
    Debug.Log($"{state.GetType().Name}: {query}");
};
```

| Value | When it fires |
|---|---|
| `SuccessfullySubscribed` | State was registered with `SubscribeState` |
| `SuccessfullyUnsubscribed` | State was removed with `UnsubscribeState` |

---

## The Inspector

The `StateMachine` component includes a custom inspector visible during Play Mode with four tabs:

| Tab | What it shows |
|---|---|
| `currentStates` | All states active right now, with weight, illegal states, elapsed duration, and max duration |
| `subscribedStates` | All subscribed states, with bindings, cooldown, buffer, duration, and all flags |
| `statesHistory` | A rolling log (last 75 entries) of every state activation and deactivation with timestamps |
| `subscribeHistory` | A rolling log (last 75 entries) of every subscribe and unsubscribe event |

Each entry is expandable and shows the full details of that state or event.

---

## API Reference

### `StateMachine`

| Method | Description |
|---|---|
| `SubscribeState(State, StateEntry)` | Registers a state with bindings, evaluated every Update |
| `UnsubscribeState(State)` | Removes a state and deactivates it if currently active |
| `SetState(State, bool respectWeight)` | Manually activates a state |
| `RemoveState(State)` | Manually deactivates a state |
| `Has(State)` | Returns `true` if the state is currently active |
| `AddBindings(State, List<StateBinding>)` | Adds bindings to an already subscribed state |
| `RemoveBindings(State, List<StateBinding>)` | Removes specific bindings from a subscribed state |
| `ClearAllStates(bool)` | Deactivates all active states; pass `true` to also clear subscribed states |

### `StateEntry` Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `bindings` | `List<StateBinding>` | `[]` | Conditions evaluated every Update to activate or deactivate the state |
| `cooldownDuration` | `float` | `0` | Seconds before the state can activate again after deactivation |
| `bufferDuration` | `float` | `0` | Seconds input is buffered while on cooldown |
| `stateDuration` | `float` | `0` | Seconds the state stays active before auto-removal (0 = no limit) |
| `singularUse` | `bool` | `false` | If `true`, the state runs `OnUpdate` once then removes itself |
| `destroyOnRemoval` | `bool` | `false` | If `true`, permanently removes the state from `subscribedStates` on deactivation |
| `respectWeight` | `bool` | `true` | If `false`, the state ignores illegal state weight resolution entirely |
| `oneWay` | `bool` | `false` | If `true`, bindings can activate the state but never deactivate it |
| `overwrite` | `bool` | `false` | If `true`, re-subscribing a state replaces the existing entry instead of being discarded |

### Events

| Event | Signature | Fires when |
|---|---|---|
| `OnStateChanged` | `Action<State, StateQuery>` | A state is activated, deactivated, or blocked — includes reason via `StateQuery` |
| `OnSubscribeChange` | `Action<State, StateQuery, StateEntry>` | A state is subscribed or unsubscribed |