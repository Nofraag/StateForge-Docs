# Hybrid State Machine — Documentation

## Table of Contents
1. [Core Concepts](#core-concepts)
2. [Creating a State](#creating-a-state)
3. [StateEntry](#stateentry)
4. [Subscribing vs Setting States](#subscribing-vs-setting-states)
5. [State Bindings](#state-bindings)
6. [Built-in Bindings](#built-in-bindings)
7. [Illegal States & Weight System](#illegal-states--weight-system)
8. [OneShot & Cooldown](#oneshot--cooldown)
9. [Managing Bindings at Runtime](#managing-bindings-at-runtime)
10. [The Inspector](#the-inspector)
11. [API Reference](#api-reference)

---

## Core Concepts

Hybrid State Machine works around four building blocks:

| Block | What it is |
|---|---|
| `State` | A behaviour your object can be in |
| `StateEntry` | Settings for how a state activates |
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
    bindings = new List<StateBinding> { ... }, // what triggers this state
    oneShot  = false,                          // remove permanently after first use?
    cooldown = 0f,                             // seconds before it can activate again
}
```

> You don't need to set `state` or `elapsedCooldown` — the state machine handles those internally.

---

## Subscribing vs Setting States

There are two ways to activate a state:

### `subscribeState` — event driven
Ties a state to bindings. The machine checks every Update and activates or deactivates the state automatically based on the binding conditions.

```csharp
stateMachine.subscribeState(
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

### `setState` — manual
Activates a state directly without any binding. The state stays active until you manually call `removeState`.

```csharp
stateMachine.setState(new FocusedState());
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

### `TimedBinding`
Activates for a set duration, then deactivates and resets automatically.

```csharp
new TimedBinding(3f)
// active for 3 seconds, then resets
// Transcript: "Active for 3s, elapsed 1.2s"
```

You can also reset the timer manually:

```csharp
TimedBinding timed = new TimedBinding(3f);
timed.Reset();
```

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

> **Important:** only the state that is trying to activate needs to list the conflict in its `illegalStates`. You don't need to add it on both sides.

---

## OneShot & Cooldown

These are set inside `StateEntry` and control reactivation behaviour.

### `oneShot`
When `true`, the state is completely removed from the machine after its first deactivation. It won't activate again even if the binding condition becomes true.

```csharp
new StateEntry
{
    bindings = new List<StateBinding> { ... },
    oneShot = true
}
```

Good for: intro animations, first-time tutorials, single-use pickups.

### `cooldown`
Sets a minimum number of seconds that must pass before the state can activate again.

```csharp
new StateEntry
{
    bindings = new List<StateBinding> { ... },
    cooldown = 2f // can't reactivate for 2 seconds after deactivation
}
```

Good for: abilities, attacks, anything with a recharge time.

---

## Managing Bindings at Runtime

You can add or remove bindings from a subscribed state while the game is running.

### `addBinding`
```csharp
stateMachine.addBinding(
    new FocusedState(),
    new List<StateBinding> { new InputBinding(input.actions["Aim"]) }
);
```

### `removeBindings`
```csharp
stateMachine.removeBindings(
    new FocusedState(),
    new List<StateBinding> { existingBinding }
);
```

> Note: to remove a specific binding you need a reference to the original binding instance, not a new one.

### `unsubscribeState`
Removes the state from the machine entirely. If it's currently active it will also be deactivated.

```csharp
stateMachine.unsubscribeState(new FocusedState());
```

---

## The Inspector

The `StateMachine` component comes with a custom inspector visible during Play Mode with four tabs:

| Tab | What it shows |
|---|---|
| `currentStates` | All states active right now, with their weight and illegal states |
| `signedState` | All subscribed states, with their bindings and conditions |
| `statesHistory` | A log of every state activation and deactivation |
| `subscribeHistory` | A log of every subscribe and unsubscribe event |

Each entry is expandable and shows the full details of that state or event including timestamps.

---

## API Reference

### `StateMachine`

| Method | Description |
|---|---|
| `subscribeState(State, StateEntry)` | Registers a state with bindings, evaluated every Update |
| `unsubscribeState(State)` | Removes a state and deactivates it if currently active |
| `setState(State)` | Manually activates a state, respects illegal states and weight |
| `removeState(State)` | Manually deactivates a state |
| `has(State)` | Returns true if the state is currently active |
| `addBinding(State, List<StateBinding>)` | Adds bindings to an already subscribed state |
| `removeBindings(State, List<StateBinding>)` | Removes specific bindings from a subscribed state |

### Events

| Event | Signature | Fires when |
|---|---|---|
| `OnStateChanged` | `Action<State, bool>` | A state is activated (`true`) or deactivated (`false`) |
| `OnSubscribeChange` | `Action<State, bool, StateEntry>` | A state is subscribed (`true`) or unsubscribed (`false`) |