# Hybrid State Machine — Documentation

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Creating a State](#creating-a-state)
3. [StateEntry](#stateentry)
4. [Subscribing vs Setting States](#subscribing-vs-setting-states)
5. [State Bindings](#state-bindings)
6. [Built-in Bindings](#built-in-bindings)
7. [Illegal States & Weight System](#illegal-states--weight-system)
8. [Cooldown & Buffer](#cooldown--buffer)
9. [Managing Bindings at Runtime](#managing-bindings-at-runtime)
10. [StateQuery & Events](#statequery--events)
11. [The Inspector](#the-inspector)
12. [API Reference](#api-reference)

---

## Core Concepts

Hybrid State Machine works around four building blocks:

| Block | What it is |
|---|---|
| `State` | A behaviour your object can be in |
| `StateEntry` | Settings for how a state activates and behaves |
| `StateBinding` | A condition that decides if a state should be active |
| `StateMachine` | The MonoBehaviour component that runs everything |

States can be active at the same time — this is not a traditional "one state at a time" machine. For example a character can be `Focused`, `Crouching`, and `Reloading` simultaneously, as long as none of them conflict with each other.

---

## Creating a State

Inherit from `State` and implement the required members:

```csharp
public class FocusedState : State
{
    // how much priority this state has in conflict resolution
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

**`stateWeight`** is required. It determines who wins when two conflicting states compete. See [Illegal States & Weight System](#illegal-states--weight-system).

**`illegalStates`** is optional. Leave it empty if this state has no conflicts.

---

## StateEntry

`StateEntry` defines the settings for how a state should behave when subscribed to the machine.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... }, // what triggers this state
    cooldownDuration = 0f,                             // seconds before it can activate again
    bufferDuration   = 0f,                             // input buffer window in seconds
    stateDuration    = 0f,                             // how long the state stays active (0 = forever)
    singularUse      = false,                          // remove after one Update tick?
    destroyOnRemoval = false,                          // remove permanently from subscribed on deactivation?
    respectWeight    = true,                           // participate in illegal state weight battles?
    oneWay           = false,                          // once activated, never removed by bindings?
}
```

> You don't need to set `state`, `elapsedCooldown`, `elapsedBuffer`, or `elapsedDuration` — the state machine handles those internally.

---

## Subscribing vs Setting States

There are two ways to activate a state:

### `SubscribeState` — event driven
Ties a state to bindings. The machine checks every Update and activates or deactivates the state automatically based on the binding conditions.

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

Use this for states that should turn on and off automatically based on input or conditions.

### `SetState` — manual
Activates a state directly without any binding. The second parameter controls whether the weight system applies.

```csharp
stateMachine.SetState(new FocusedState(), respectWeight: true);
```

Use this for states you want full manual control over, like a cutscene state or a death state.

---

## State Bindings

A `StateBinding` is a condition that the machine evaluates every Update to decide if a subscribed state should be active or not.

To create a custom binding, inherit from `StateBinding` and implement two methods:

```csharp
public class HealthBinding : StateBinding
{
    // the condition — return true to activate the state
    public override bool Evaluate()
    {
        return machine.GetComponent<Health>().current < 20f;
    }

    // what the inspector shows for this binding
    public override string Transcript()
    {
        return "Health below 20.";
    }
}
```

`machine` is automatically available inside any binding — it's the `StateMachine` component the binding is attached to.

### AND Logic — BindingFolder

To require multiple conditions at once, wrap them in a `BindingFolder`:

```csharp
new BindingFolder(new List<StateBinding>
{
    new InputBinding(input.actions["RightClick"]),
    new InputBinding(input.actions["Aim"])
})
```

This only returns true if **all** bindings inside evaluate to true.

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
A flexible binding that delegates to a `Condition` class you implement yourself. Useful when the logic is complex or reusable.

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

Use this over a raw custom `StateBinding` when you want to reuse the same condition across multiple states or projects.

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

In this example, if `SprintingState` tries to activate while `FocusedState` is active:
- `SprintingState` (0.6) **>** `FocusedState` (0.5) → Sprinting wins, Focused is removed
- If the weights were reversed, Sprinting would be blocked entirely

> **Important:** you only need to declare the conflict on one side. If either state lists the other as illegal, they can never be active together.

### Bypassing the weight system

Set `respectWeight = false` on a `StateEntry` to make a state activate regardless of conflicts. It will never be blocked or removed by illegal state logic.

```csharp
new StateEntry
{
    bindings      = new List<StateBinding> { ... },
    respectWeight = false // ignores illegal state rules entirely
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
    cooldownDuration = 2f // can't reactivate for 2 seconds after deactivation
}
```

Good for: abilities, attacks, anything with a recharge time.

### `bufferDuration`
Sets an input buffer window. If a binding evaluates true while the state is on cooldown, the machine stores the input for `bufferDuration` seconds. When the cooldown expires, the state activates immediately without requiring the player to press again.

```csharp
new StateEntry
{
    bindings         = new List<StateBinding> { ... },
    cooldownDuration = 0.5f,
    bufferDuration   = 0.15f // input is buffered for 150ms
}
```

Good for: attacks, rolls, any action where early input should still register.

### `stateDuration`
The state automatically removes itself after this many seconds of being active. Set to `0` (default) for no automatic removal.

```csharp
new StateEntry
{
    bindings      = new List<StateBinding> { ... },
    stateDuration = 0.4f // state lasts 0.4 seconds then removes itself
}
```

Best paired with `cooldownDuration` to prevent immediate reactivation, or `oneWay` for a state that triggers once and runs for a fixed time.

Good for: attacks, rolls, timed buffs, ability windows.

### `singularUse`
When `true`, the state activates, runs `OnUpdate` exactly once, then removes itself from `currentStates`. It stays in `subscribedStates` so it can activate again the next time the binding condition becomes true.

```csharp
new StateEntry
{
    bindings    = new List<StateBinding> { ... },
    singularUse = true
}
```

Good for: single frame impulses, one-tick events, fire-and-forget effects.

### `destroyOnRemoval`
When `true`, the state permanently removes itself from `subscribedStates` the moment it is deactivated. It will never activate again even if the binding condition becomes true.

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

> **Note:** `oneWay` only suppresses binding-based removal. `stateDuration`, `singularUse`, and manual `RemoveState` calls still work normally.

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

> Note: to remove a specific binding you need a reference to the original binding instance, not a new one.

### `UnsubscribeState`
Removes the state from the machine entirely. If it's currently active it will also be deactivated.

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

Every state activation, deactivation, and failure fires an `OnStateChanged` event with a `StateQuery` value explaining exactly what happened. Subscribe to it to react to state changes or debug your setup.

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
| `RemovedByWeight` | State was removed because a higher-weight state won |
| `AlreadyActive` | `SetState` was called but the state was already active |
| `NotActive` | `RemoveState` was called but the state wasn't active |
| `OnCooldown` | `SetState` was blocked because the cooldown hasn't expired |
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

The `StateMachine` component comes with a custom inspector visible during Play Mode with four tabs:

| Tab | What it shows |
|---|---|
| `currentStates` | All states active right now, with weight, illegal states, elapsed duration, and max duration |
| `signedState` | All subscribed states, with bindings, cooldown, buffer, duration, and all flags |
| `statesHistory` | A rolling log (last 50 entries) of every state activation and deactivation with timestamps |
| `subscribeHistory` | A rolling log (last 50 entries) of every subscribe and unsubscribe event |

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
| `Has(State)` | Returns true if the state is currently active |
| `AddBindings(State, List<StateBinding>)` | Adds bindings to an already subscribed state |
| `RemoveBindings(State, List<StateBinding>)` | Removes specific bindings from a subscribed state |
| `ClearAllStates(bool)` | Deactivates all active states; pass true to also clear subscribed states |

### `StateEntry` Fields

| Field | Type | Default | Description |
|---|---|---|---|
| `bindings` | `List<StateBinding>` | `[]` | Conditions evaluated every Update to activate or deactivate the state |
| `cooldownDuration` | `float` | `0` | Seconds before the state can activate again after deactivation |
| `bufferDuration` | `float` | `0` | Seconds input is buffered while on cooldown |
| `stateDuration` | `float` | `0` | Seconds the state stays active before auto-removal (0 = no limit) |
| `singularUse` | `bool` | `false` | If true, state runs `OnUpdate` once then removes itself |
| `destroyOnRemoval` | `bool` | `false` | If true, permanently removes the state from `subscribedStates` on deactivation |
| `respectWeight` | `bool` | `true` | If false, state ignores illegal state weight battles entirely |
| `oneWay` | `bool` | `false` | If true, bindings can activate the state but never deactivate it |

### Events

| Event | Signature | Fires when |
|---|---|---|
| `OnStateChanged` | `Action<State, StateQuery>` | A state is activated, deactivated, or blocked — includes reason via `StateQuery` |
| `OnSubscribeChange` | `Action<State, StateQuery, StateEntry>` | A state is subscribed or unsubscribed |
