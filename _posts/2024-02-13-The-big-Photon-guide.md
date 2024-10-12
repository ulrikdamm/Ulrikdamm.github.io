---
layout: page
title: "The Big Photon Fusion Guide"
permalink: /photon-fusion-guide
---

Alright, this is going to be kind of a primer on the principles on the model of networking we’re using in our game. The reason it’s getting documented here is that it’s a kind of complicated setup, and small mistakes can be difficult to debug, so care must be taken in implementing everything correctly. Also, Photon’s own documentation is worthless.

## Overview and definitions

The idea behind host-authoritative networking is that only one client (or a dedicated host server) can modify the shared state of the game. Want to move an object, and have it move for every player? Only the host can do that. If you’re a client, you have to tell the host to move it for you. There are two different ways of telling things to the host (Remote Procedure Calls and networked input).

* Remote Procedure Calls is the easy option, it’s just sending a message to the host, as if it was an HTTP request. The message can be configured to just executing as soon as possible, or to wait to run on the same simulation tick as the client is currently in. You can also attach parameters, such as a target position for an ability, or even reference other networked objects, by just passing in a NetworkObject reference. TODO downsides of RPC.
* If you want it to move immediately on the client, you can choose to simulate it’s position, while waiting for the host. The host will then need to apply the change retroactively, so that it has started on the same tick as on the client tick. The way this is done is with networked input. Photon will be polling the client with Input Authority over an object for input, and this input will be applied on the host as well.
    * In a simple game, an input would be directly tied to mouse/keyboard input, so that Input.GetKey would be passed into an input struct, which will be picked up by Photon, an replicated on the host. Since this is not a simple game, and input is not simple, out input won’t be tied directly to a button, but instead to something like “used the lord action tied to the left mouse button” or “consumed a health potion”.
    * This also means that the game can’t do simulation based directly on input, it has go to through the Photon input. This means that you can’t just use Input.GetKey inside the simulation, because the simulation can be rolled back and re-simulated, and it will need to be able to re-poll old input.

Each object in the world have one of three client-local states:

* Input authoritative means that the client controls input for that.
* State authoritative means that the client controls the shared networked state of an object. For the host-authoritative setup, the host has state authority over all objects. In a setup with shared authority, state authority can be passed to clients, but we’re not using that.
* If an object is neither input or state authoritative, it’s a proxy. Proxies are empty shells of objects, that don’t do any simulation or rewind, but only receives their networked state, and applies that state visually. This is for example another player lord, when you’re not the host.

## Photon Components

These are the basic components that you usually need for a Photon setup

### FusionBootstrap

Component that needs to be in the scene, will setup the connection.

### NetworkRunner

The main object responsible for running the networking simulation. Everything networked will be connected to the Runner. You can access the current simulation tick with the Tick property, and with IsForward and IsResimulation you can access wether the current Tick is the newest simulation step, or if it’s a re-simulation of an old tick.

### NetworkObject

This is a component you attach to a game object to make it networked. Needs to be in the root of the object, and will apply to all child objects also. This contains things like the owner of the object, and wether the local client has state/input authority.

### NetworkTransform

Put on an object to sync its transform (position/rotation/scale/parent). All the transform properties will act as if they’re [Networked].

### NetworkBehaviour

This is a base class you’ll need to subclass for a class to interact with the simulation. It’ll have the following overridable methods:

* Spawned is called when the object appears in the simulation
* FixedUpdateNetwork is where a simulation tick happens. This is only called for objects with state and/or input authority. Called on a regular interval, equivalent to FixedUpdate() for physics. In this method, you can’t use Time.deltaTime, you need to use Runner.deltaTime.
* Render is called each screen-frame, like Update, and contains an interpolated version of the networked state. All visuals responding to networked state should happen here, like animator properties and particle system activations.

Other than that, you get access to some useful methods and properties:

* Runner accesses the NetworkRunner.
* GetInput<> accesses the input struct for the current tick, if available.
* GetChangeDetector returns a change detector, that can be used to track changes in networked state, especially useful for proxies.

And you’re able to use these attributes:

* [Networked] attribute on a property makes that property part of the networked simulation, and automatically updates it based on the current tick. Only the state authority can change these, if a client changes them, that’ll be seen as a prediction, and will be overridden at some point with the real value. Networked properties must have a getter and setter, and these will have their implementation spliced in at compiletime by Photon using IL weaving.
* [Rpc] attribute on a method makes it into a Remote Procedure Call. The name must also be prefixed or suffixed with “rpc”.

### INetworkRunnerCallbacks

TODO

### INetworkInput

TODO

## Simulation principles

All networked state can be seen as a deterministic simulation, being executed potentially in parallel on multiple clients. This simulation has [Networked] properties as its state. Communication into and out of the simulation must happen at certain points, and all methods called from within the simulation must also follow the same rules.
A simple example of just an object moving at a constant velocity:

```c#
class ExampleSimulation : NetworkBehaviour {
    [Networked] Vector3 position { get; set; }
    
    public override void FixedUpdateNetwork() {
        position += Vector3.left * Runner.deltaTime;
    }
}
```

This simulation could run on different machines, and is sufficiently deterministic. Note that this wouldn’t work for a lockstep implementation, since float imprecision could cause the exact value to drift differently over time on different machines, leading to the simulation diverging. We don’t have this problem, because we have a host as the final arbiter. The network will be synced regularly, and the host will apply its value of position to a client simulation, if they become out of sync, and re-simulate the ticks since the desync.
Let’s say, for arguments sake, that Vector3.left had been overridden to have a length of 2 on the client. What would happen to the simulation? For the first few ticks, the position would increase at double the velocity. Starting at tick 0 with an X value of 0, for each tick the value would go 2, 4, 6, etc.
At tick 3, with a value of 6, it might get a sync back from the server, and it’ll perform a rollback of the client prediction, insert the server value of position, and then re-run the simulation. So at tick 1, the server would overwrite the 2 it had on the client to a 1, and then re-run the next simulation ticks. Since the local Vector3.left still is at odds with the server value, the local simulation would keep diverging in the prediction, but now at least at tick 1, it would have the correct value, and it would simulate 3, 5, 7, etc. as the next values. The next server sync might apply the value of 4 to tick 4, that used to have the value of 7, and then re-run the simulation up to the current tick. There will always be a tick where all the values match the server values, but all ticks after that are client predicted, and will eventually be re-simulated. This means that if you put a Debug.Log into FixedUpdateNetwork, printing the current Runner.Tick, you will see it print the same tick value multiple times on the client.
A client mis-prediction is no big deal, since a prediction is going to be wrong at some point no matter what, so the system is made to handle them. What is more of a problem is a non-deterministic simulation. This could be relying on non-simulation state. Take this example:

```c#
class ExampleSimulation : NetworkBehaviour {
    [Networked] Vector3 position { get; set; }
    
    float velocity;
    
    public override void FixedUpdateNetwork() {
        velocity += Runner.deltaTime;
        position += Vector3.left * velocity * Runner.deltaTime;
    }
}
```

The velocity field is state that is not part of the simulation, since it’s not marked with [Networked]. This means that, while it’ll exist on both the host and the client, and be increased similarly, it won’t get host changes applied, and it won’t be rolled back when re-simulating. So when the client receives values from the host, and performs a rollback, the value of velocity will not only stay the same, it will have the += Runner.deltaTime re-applied, since those ticks are evaluated multiple times, and the result will be a velocity value of out sync between the host and the client.
Accessing non-mutable values, like Vector3.left or ScriptableObject properties, is no problem, since these will be the same on the host and client, but all mutable state must be done through networked properties.

## Timers

For timers specifically, Photon has a built-in networked TickTimer. For networked state, this should be used instead of a float counting down:

```c#
class ExampleSimulation : NetworkBehaviour {
    [Networked] TickTimer timer { get; set; }
    
    public override void Spawned() {
        timer = TickTimer.CreateFromSeconds(Runner, 1);
    }
    
    public override void FixedUpdateNetwork() {
        if (!timer.ExpiredOrNotRunning(Runner)) {
            // Still in the first second of the simulation
        }
        
        if (timer.Expired(Runner)) {
            timer = default; // Should be reset if you want something to only trigger once
        }
    }
}
```

Under the hood, the TickTimer is just a target tick, and the methods you call will compare the target tick to the current tick.

## Simulation Input

As mentioned in the overview, input into the simulation can be done with either Remote Procedure Calls or networked input.

### Remote Procedure Calls

The remote procedure call will call directly into the host to change values. This could be a player performing an emote, that we want replicated on all clients, and it not time critical, so won’t be needing client-side prediction. We define a Remote Procedure Call by giving it the [Rpc] attribute

```c#
[Networked] public int lastEmotePerformed { get; set; }

public void performEmote(int type) {
    rpcPerformEmote(type);
}

[Rpc]
void rpcPerformEmote(int emoteType) {
    lastEmotePerformed = emoteType;
}
```

Photon will re-write the code at compile time, so that the rpcPerformEmote will be transformed into an RPC call on the client, that will invoke the method on the host. The clients can then set up a change detector on the lastEmotePerformed property, and display the emote on value change (more on that later).

### Networked input

Networked input is what you’ll use when you need input into the simulation to happen on a specific tick. Unlike RPCs, that are just inserted whenever they’re received by the server, network input is polled for every tick, and inserted into the simulation. They will happen immediately on the client, since the client will be running a prediction of the simulation, and it will be applied on the server when it catches up to the client’s simulation tick.
All networked input has to be put into a single struct, marked by the INetworkInput interface:

```c#
struct NetInput : INetworkInput {
    public Vector2 playerMovement;
    public int abilityActivated;
}
```

The struct should be created in the normal Unity Update call, since Unity expects input to be performed here:

```c#
NetInput inputValues;

void Update() {
    inputValues.playerMovement = new Vector2(Input.GetAxis("Horizontal"), Input.GetAxis("Vertical"));
}

public void onAbilitySelected(int id) {
    inputValues.abilityActivated = id;
}
```

The input is then passed on to Photon in the OnInput method from the INetworkRunnerCallbacks interface, that is implemented on a component, sitting on the same game object as the NetworkRunner:

```c#
void OnInput(NetworkRunner runner, NetworkInput input) {
    input.Set(inputValues);
    
    inputValues.abilityActivated = 0;
}
```

Notice that the abilityActivated value only should be set for a single tick, the tick in which the ability is performed, so it’s reset once the input has been sent.
The input values are now registered in the simulation, and can be retrieved in a FixedUpdateNetwork via the GetInput method:

```c#
public override void FixedUpdateNetwork() {
    if (GetInput<NetInput>() is {} input) {
        position += new Vector3(input.playerMovement.x, 0, input.playerMovement.y) * Runner.deltaTime;
        
        if (input.abilityActivated != 0) {
            // Start performing ability
        }
    }
}
```

Notice that the input might not be available. This might be that you don’t have input authority over the object. All player inputs are not sent to all clients, only to the host. If an object is neither input- nor state-authority, it’s a proxy object, and shouldn’t perform input-dependent simulation anyway, but only visually display the object’s state.

## Simulation Output

Outputting results of the simulation seems like a straightforward thing to do. For outputting positions, you just change transform.position. Change to a walk animation when moving by checking the change in position, and apply it to the animator. And some of this is simple enough. However, even in this, there are caveats. Say, if you get input that an ability has been pressed, and you want to perform an animation. If you just use SetTrigger in FixedUpdateNetwork, this is going to lead to problems, since the trigger will end up being set multiple times, for each time that tick is re-simulated. Also, FixedUpdateNetwork doesn’t get executed on proxy objects (objects where the game is neither state nor input authority), so neither the walk nor ability animations will get triggered.
Instead, output of the simulation should happen in the Render method. This method is called more like the Unity Update message, in between ticks, to match the framerate of the game. This will also prevent stutter, since applying values in FixedUpdateNetwork will only happen at the specified tick interval, no matter the refresh rate of the screen.

### The Render method

Let’s start with the simple case of a varying value:

```c#
class PlayerMovement : NetworkBehaviour {
    [Networked] Vector3 position { get; set; }
    
    public override void Render() {
        transform.position = position;
    }
    
    public override void FixedUpdateNetwork() {
        if (GetInput<NetInput>() is {} input) {
            position += new Vector3(input.playerMovement.x, 0, input.playerMovement.y) * Runner.deltaTime;
        }
    }
}
```

Note that you would properly just move an object using the NetworkTransform component, but for demonstrations sake, this object doesn’t have one, and we change transform.position manually instead.
In this case, if we had just applied input.playerMovement directly to transform.position in FixedUpdateNetwork, the movement would end up jittery, since the position wouldn’t be smoothly interpolated, and also proxy objects wouldn’t move at all. The input authority object would also have its transform.position changed back and forth during re-simulation, which is not ideal.
There’s a lot of different techniques you can use in the Render method, so they’ll be covered in its own section later.

### Animator parameters

Next let’s try to apply a walk speed as well:

```c#
class PlayerMovement : NetworkBehaviour {
    [SerializeField] Animator animator;
    
    [Networked] Vector3 position { get; set; }
    Vector3? previousPosition;
    
    public override void Render() {
        transform.position = position;
        
        if (previousPosition is {} previous) {
            animator.Setfloat("WalkSpeed", (position - previous).magnitude * Runner.deltaTime);
        }
        
        previousPosition = position;
    }
    
    public override void FixedUpdateNetwork() {
        if (GetInput<NetInput>() is {} input) {
            position += new Vector3(input.playerMovement.x, 0, input.playerMovement.y) * Runner.deltaTime;
        }
    }
}
```

Here we save the old position in a field, and use that to figure out, how far the object has moved since the last frame. We don’t need a networked property here, because Render is not called in the re-simulation, only going forward from it’s current state.

### Animation triggers and Change Detectors

Our next task is slightly more involved, which is to set an animation trigger. This has to be done only once, and it has to be done in the Render method.
The obvious way to do it would be to have a boolean field, that get’s set by the simulation step, and the next Render call will then trigger the animation, and reset the field. However this won’t work correctly, since the same simulation step might get calculated multiple times, and might then trigger the animation on multiple different frames. Also, it will never trigger on proxies.
Instead, to observe values in the simulation from the outside, a Change Detector can be used instead. This will keep track of networked state, and you can poll it for changes in the Render method. Here’s an example, omitting the FixedUpdateNetwork, as it will be from the point of view of a proxy:

```c#
class PlayerAnimation : NetworkBehaviour {
    [SerializeField] Animator animator;
    
    [Networked] int actionCount { get; set; }
    
    ChangeDetector changeDetector;
    
    public override void Spawned() {
        changeDetector = GetChangeDetector(ChangeDetector.Source.SimulationState);
    }
    
    public override void Render() {
        var changes = changeDetector.DetectChanges(this, out var oldValues, out var newValues);
            foreach (var change in changes) {
            case nameof(actionCount):
                var propertyReader = GetPropertyReader<int>(change);
                var (oldValue, newValue) = propertyReader.Read(oldValues, newValues);
                if (newValue > oldValue) {
                    animator.SetTrigger("Attack");
                }
                break;
        }
    }
}
```

You might be wondering why an actionCount as an integer is used instead of just a bool actionPerformed. The reason for that is that even if the bool was only active for one tick, multiple Renders might get run in between a single tick. The Render method also has no way of resetting the value after the animation has been played, since the value is part of the simulation.
Another option, instead of using a Change Detector, is to keep a non-simulation field in the class, keeping track of the amount of displayed actions, and in the Render method check if actionCount is bigger than displayedActionCount, and update the local value accordingly.

```c#
class PlayerAnimation : NetworkBehaviour {
    [SerializeField] Animator animator;
    
    [Networked] int actionCount { get; set; }
    
    int displayedActionCount;
    
    public override void Render() {
        if (actionCount > displayedActionCount) {
            animator.SetTrigger("Attack");
            displayedActionAcount = actionCount;
        }
    }
}
```

### Non-networked simulation output

If you need some output from the simulation, that is only visible to the local player (this could be a green UI flash when using a health potion), there is an easier way, that doesn’t require the Render method. This can be done from within the simulation tick, by checking if we’re currently simulating the local player, and only if the simulation is going forward (meaning it’s not a re-simulation). We need to check for forward, since the tick is going to be re-simulated on the client when the next host data arrives, and we don’t want the effect playing twice.

```c#
class PlayerHealth : NetworkBehaviour {
    [SerializeField] GameUI ui;
    
    [Networked] int health { get; set; }
    
    // This method is a simulation method, should only be called from within FixedUpdateNetwork
    public void pickUpPotion() {
        health += 100;
        
        if (HasInputAuthority && Runner.IsForward) {
            ui.showFlash(Color.green);
        }
    }
}
```

Of course, if it ends up that the host decides that you actually didn’t pick up the health potion, but some other player took it first, the flash will still play, even though you get no health from it. This is a tradeoff you’ll have to make: display the flash immediately, even when it might be wrong, or delay the flash until you get a response from the server.

### Output through RPC

Another way to output an effect is by invoking a Remote Procedure Call from the host to all clients. This is optimal for state-events that are not predicted by any clients, but that should be visible to all players. An example is dealing damage to an enemy; if you’re ok with a slight delay between attack animation and hit reaction on the enemy unit, you can have the client only predict the attack animation itself, and leave dealing damage exclusively to the host. The host will simulate the player attack along with the client prediction, but the hit detection, damage dealing, and hit reaction will only be done in one place, and any results of the action will be sent to players in a Remote Procedure Call.

```c#
class PlayerAttack : NetworkBehaviour {
    [Networked] int attackCount { get; set; }
    
    public override void FixedUpdateNetwork() {
        if (GetInput<NetInput>() is not {} input) { return; }
        
        if (input.attackTriggered) {
            attackCount += 1; // The Render method will pick this up to display the attack
            
            if (HasStateAuthority) { // This will only happen on the host
                if (findEnemyInRange(transform.position) is {} enemy) {
                    enemy.health -= 10;
                    rpcEnemyDamaged(enemy.GetComponent<NetworkObject>());
                }
            }
        }
    }
    
    // This RPC will be invoked on all clients, including the host
    [Rpc(sources: RpcSoures.StateAuthority, targets: RpcTargets.All)]
    void rpcEnemyDamaged(NetworkObject unit) {
        unit.GetComponent<EnemyRenderer>().showHitReaction();
    }
}
```

### More Render method techniques

The Render method has to observe the [Networked] state of an object, and apply that state to the visual display of it. This sounds simple enough, but you’ll quickly run into a lot of tricky cases. Let’s go through a few of these.

#### Observing timers

Timers are very widely used, and it’s common to have something happen visually when a timer expires. The first thought would maybe be to use a ChangeDetector to track the timer, and find out if it has changed from running to expired, but that’s not directly possible, since the timer itself doesn’t have any changing state; it’s just a wrapper around a final Tick, and the methods you call on it just compares that to the current Tick. If you reset the timer after it’s done, it’s possible to use a change detector to check if it has gone from set to unset, and play the effect then. However, if it’s remaining in the expired state, you can instead just check if it’s .Expired, and use a local non-networked field to check if the effect has already been displayed. Even though the timer is a networked property, it’s still possible to read its value from the non-networked Render method, it will just have the value of the latest simulation tick (and, remember, might get rolled back. But if it is, the simulation will re-play all ticks up to the current simulation tick before the Render method is called again).

```c#
class ShowEffectOnEnd : NetworkBehaviour {
    [Networked] TickTimer timer { get; set; }
    
    bool showingEffect;
    
    // Simulated
    public void showEffect() {
        timer = TickTimer.CreateFromSeconds(Runner, 1);
    }
    
    public override void Render() {
        if (timer.ExpiredOrNotRunning(Runner)) {
            if (showingEffect) {
                // End effect
                showingEffect = false;
            }
        } else {
            if (!showingEffect) {
                // Show effect
                showingEffect = true;
            }
        }
    }
}
```

An effect of this technique is that if a lag spike happens, the effect might be skipped completely, since the simulation will both start and end the timer before the Render method happens. This might be desired or undesired depending on the effect.
This can also be written using an observer pattern:

```c#
class ShowEffectOnEnd : NetworkBehaviour {
    [Networked] TickTimer timer { get; set; }
    
    bool _showingEffect;
    bool showingEffect {
        get => _showingEffect;
        set {
            if (_showingEffect == value) { return; }
            _showingEffect = value;
            
            if (value) {
                // Show effect
            } else {
                // End effect
            }
        }
    }
    
    public override void Render() {
        showingEffect = !timer.ExpiredOrNotRunning(Runner);
    }
}
```

#### Local-player-only effects

If you need an effect to show up only for the local player, like a stamina meter displayed on screen, you just use HasInputAuthority as a check

```c#
class StaminaHandler : NetworkBehaviour {
    [SerializeField] UIProgressBar staminaBar;
    
    [Networked] float stamina { get; set; }
    
    public override void Render() {
        if (HasInputAuthority) {
            staminaBar.showFilled(stamina);
        }
    }
}
```

## Spawning and despawning

### Spawned callback

Spawned is called when a NetworkBehaviour has been instantiated and registered into the Fusion simulation. Before Spawned is called, accessing (even just reading) Networked properties will result in an error.
When registering multiple objects at the same time, like when loading a scene, all the objects are registered before the first Spawned is called, which means that you can access networked properties on other objects before their Spawned callback, as long as they’re valid (IsValid is true).
TODO
NetworkObject.IsValid to check if the object has been spawned?

### Common patterns

#### Sending commands

A common pattern is for the client to ask the host to do something, and get a response to that action. This could be a player wanting to buy an item for their character. If the player is the host, they have the authority to just do it, so they can just perform the action, as if it was a single-player game (just remember that any changes, like resources spent, has to be reflected on all clients). If you’re not the host, you’ll have to ask the host to do it for you. A few things to keep in mind:

* You should check as much as you can on the client before sending any commands. For example, don’t request the host to buy an item that you locally can see that you can’t afford.
* The host should check that the request is valid, don’t just trust the client.
* If the request fails on the host, the client should get an error message, as if it failed locally.
* Errors sent over the network can’t be strings; the host and client might use different languages.

To put this into practise, this is how this pattern could be implemented:

```c#
public void purchaseItem(Item item) {
    if (!canPurchaseItem(item, out var error)) {
        ui.showError(error.ToString());
        return;
    }
    
    if (!HasStateAuthority) {
        rpcPurchaseItem(item.id);
        return;
    }
    
    performPurchase(item, for: Runner.localPlayer);
}

enum ItemPurchaseError { none, unknown, missingResources, unavailable }

bool canPurchaseItem(Item item, out ItemPurchaseError error) {
    if (!resourceManager.contains(item.purchaseCost)) {
        error = ItemPurchaseError.missingResources;
        return false;
    }
    
    if (!item.availableForPurchase) {
        error = ItemPurchaseError.unavailable;
        return false;
    }
    
    return true;
}

[Rpc(sources: RpcSources.Proxies, targets: RpcTargets.StateAuthority, InvokeLocal = false)]
void rpcPurchaseItem(int itemId, RpcInfo info = default) {
    var item = itemManager.itemById(itemId);
    if (item == null) { rpcPurchaseItemError(info.Source, ItemPurchaseError.unknown); }
    if (!canPurchaseItem(item)) { rpcPurchaseItemError(info.Source, ItemPurchaseError.missingResources); }
    if (!item.availableForPurchase) { rpcPurchaseItemError(info.Source, ItemPurchaseError.unavailable); }
    
    performPurchase(item, for: info.Source);
}

[Rpc(sources: RpcSources.StateAuthority, targets: RpcTargest.Proxies, InvokeLocal = false)]
void rpcPurchaseItemError([RpcTarget] PlayerRef player, ItemPurchaseError error) {
    ui.showError(error.ToString());
}
```

A few things to note here:

* canPurchaseItem is run both on the client and the server.
* Even though the item would probably not be visible in the ui, or be disabled, if it’s unavailable, a check is still made to see if it is, since a modded client might enable the button, or send RPC to try to buy any item.
* The client to server RPC has an RpcInfo parameter, this is how you get which client sent the request (.Source).
* The server to client RPC has an [RpcTarget] parameter, this is how to target an RPC to go to one specific player (you only want the error to show up for the player that tried to perform the purchase, not all players).
* Fusion is limited in what types can be sent as parameters to an RPC: plain enums are fine (they’re just ints), ScriptableObjects are not (you’ll have to serialize/deserialize them, for example to an int identifier).
* The error enum has an “unknown” case, used for when something unexpected goes wrong (like the item deserialization failing). There’s not really any good error to show for this case, but just choosing a random one or using “none” is problematic.

One final detail is that here the error is just a plain enum, which simplifies the example, but that’s missing to things that might be critical:

* No way to attach parameters to each case. If you want an error like “requires character level 10”, you need to attach the level value to the case somehow.
* No way to do custom error messages. You can of course create a itemPurchaseErrorToString method, but it would be nice if this could be in the error type itself.

We can solve that by creating an error struct instead. Structs can be sent as RPC parameters, as long as they implement the INetworkStruct interface, and only contains RPC-compatible types.

```c#
struct ItemPurchaseError : INetworkStruct {
    public enum Type { none, unknown, missingResources, unavailable, minLevelRequired }
    
    public Type type;
    public int minLevel;
    
    public static ItemPurchaseError none => new ItemPurchaseError { type = Type.none };
    public static ItemPurchaseError unknown => new ItemPurchaseError { type = Type.unknown };
    public static ItemPurchaseError missingResources => new ItemPurchaseError { type = Type.missingResources };
    public static ItemPurchaseError unavailable => new ItemPurchaseError { type = Type.unavailable };
    public static ItemPurchaseError minLevelRequired(int level) => new ItemPurchaseError { type = Type.minLevelRequired, minLevel = level };
    
    public string description {
        get {
            return this switch {
                Type.missingResources => "Not enough resources to buy this",
                Type.unavailable => "This item is currently unavailable",
                Type.minLevelRequired => $"Requires character level {minLevel}",
                _ => $"Unknown error ({(int)type})"
            };
        }
    }
}
```

The static fields are for convenience to make it easier to create error than new ItemPurchaseError { type = ... }, but they also exist for correctness purposes, since you can’t forget to pass in the level for the minLevelRequired case. However, if you find them annoying to create, and care less about potentially missing parameters, you can omit these.
This is quite a lot of code, but it gets worse. What if you want a ScriptableObject as an error case parameter? Like if you want to buy an item that is an upgrade of another item, and in the case you don’t have the item to upgrade, the error should say “Requires name-of-item”? You can’t have ScriptableObject references in an INetworkStruct. In this case, you’ll have to make a “normal” error, and serialize it into a “network” error, and deserialize it again when receiving it:

```c#
struct ItemPurchaseError {
    public enum Type { none, unknown, missingResources, unavailable, minLevelRequired, itemRequired }
    
    public Type type;
    public int minLevel;
    public Item requiredItem;
    
    public static ItemPurchaseError none => new ItemPurchaseError { type = Type.none };
    public static ItemPurchaseError unknown => new ItemPurchaseError { type = Type.unknown };
    public static ItemPurchaseError missingResources => new ItemPurchaseError { type = Type.missingResources };
    public static ItemPurchaseError unavailable => new ItemPurchaseError { type = Type.unavailable };
    public static ItemPurchaseError minLevelRequired(int level) => new ItemPurchaseError { type = Type.minLevelRequired, minLevel = level };
    public static ItemPurchaseError itemRequired(Item item) => new ItemPurchaseError { type = Type.itemRequired, requiredItem = item };
    
    public string description {
        get {
            return this switch {
                Type.missingResources => "Not enough resources to buy this",
                Type.unavailable => "This item is currently unavailable",
                Type.minLevelRequired => $"Requires character level {minLevel}",
                _ => $"Unknown error ({(int)type})"
            };
        }
    }
}

struct NetworkItemPurchaseError : INetworkStruct {
    public int type;
    public int parameter;
    
    public NetworkItemPurchaseError(ItemPurchaseError error) {
        type = (int)error.type;
        parameter =
            error.type == ItemPurchaseError.Type.minLevelRequired ? error.minLevel :
            error.type == ItemPurchaseError.Type.itemRequired ? error.requiredItem.id :
            0;
    }
    
    public ItemPurchaseError toNormal(ItemManager itemManager) {
        if (!System.Enum.IsDefined(typeof(ItemPurchaseError.Type), this.type)) {
            return ItemPurchaseError.unknown;
        }
        
        var type = (ItemPurchaseError.Type)this.type;
        switch (type) {
            case ItemPurchaseError.Type.minLevelRequired:
                return ItemPurchaseError.minLevelRequired(parameter);
            case ItemPurchaseError.Type.requiredItem:
                var item = itemManager.itemById(parameter);
                if (item == null) { return ItemPurchaseError.unknown; }
                return ItemPurhcaseError.requiredItem(item);
            default:
                return new ItemPurchaseError { type = type };
        }
    }
}
```

And that’s it, now just use `NetworkItemPurchaseError` in your RPC parameters, and convert them between normal and networked errors as needed. Now you got client/host commands with proper checking and error handling.

#### Invoking command that transparently is delegated to the host

Often when a player inputs a command, like moving a group of units to a specific position, you just want to be able to call a method that makes that action happen, wether the client is the host or not. This is the typical way to set that up:

```c#
public void moveUnits(Unit[] units, Vector3 target) {
    if (HasStateAuthority) {
        hostMoveUnits(units, target, Runner.LocalPlayer);
    } else {
        rpcMoveUnits(units, target);
    }
}

void hostMoveUnits(Unit[] units, Vector3 target, PlayerRef player) {
    foreach (var unit in units) {
        unit.moveTo(target);
    }
    
    rpcDidMoveUnits(player, units, target);
}

[Rpc(RpcSources.All, RpcTargets.StateAuthority, InvokeLocal = false)]
void rpcMoveUnits(Unit[] units, Vector3 target, RpcInfo info = default) {
    hostMoveUnits(units, target, info.Source);
}

    [Rpc(RpcSources.StateAuthority, RpcTargets.All, InvokeLocal = true)]
void rpcDidMoveUnits([RpcTarget] PlayerRef player, Unit[] units, Vector3 target) {
    foreach (var unit in units) {
        unit.playActionConfirmSound();
    }
}
```

`hostMoveUnits` is the method that actually performs the action, moveUnits is the public facing method that will either just call the internal move method directly if possible, otherwise send an RPC that will call the method on the host.
When the action has been performed on the host, it will call an RPC back to the client that initiated the command, to give them a confirmation. That client might be the host itself, that’s why InvokeLocal is set to true.
Remember that this example doesn’t include validation (is that player allowed to move those units?), failure states (those units can’t go there, show an error to the player) or race conditions (the units are already dead). Also you might have to do some type conversion, if the input arguments to moveUnits has types that can’t be sent over the network (non-INetworkStruct structs, interface references, references to non-NetworkObject objects), so in that case you’ll have to do conversions before call into and out of the RPCs (see the Sending commands section above for examples of how to handle that)

## Debugging

Fusion 2 can have some obscure, hard-to-debug failure states, so here’s a handy lookup for the ones I’ve encountered so far:

#### AssertException Not in list

Error example:

`AssertException: arg0:Not in list arg1:Brute arg2:[Type: NetworkTransform, List: Lord.Man Nav(Clone)]`

This happens when you despawn an already despawned object, like something calling Runner.Despawn from the Despawned callback. Can also happen when despawning objects while Fusion is in shutdown mode, for this either check Runner.IsShutdown or hasState in the Despawned callback.
There doesn’t seem to be any official way to check if a NetworkObject has been spawned/despawned, but I think you might be able to use .Runner == null.

#### No-context AssertException in NetworkObjectHeaderSnapshotRef

Error example:

```
AssertException: Exception of type 'Fusion.AssertException' was thrown.
at Fusion.Assert.Check (System.Boolean condition) [0x00008] in <601d71dc7c0c4885ab33fc6f50c22b95>:0
at Fusion.NetworkObjectHeaderSnapshotRef.GetChangedTickForBehaviour (System.Int32 index) [0x00001] in Fusion\Fusion.Runtime\Object\NetworkObjectHeaderSnapshot.cs:19
```

This can happen if the networked objects in the scene on the host and client don’t match.

#### No-context AssertException in networked collections (NetworkDictionary, NetworkArray et. al.)

Error example:

```
AssertException: Exception of type 'Fusion.AssertException' was thrown.
Fusion.Assert.Check (System.Boolean condition) (at <601d71dc7c0c4885ab33fc6f50c22b95>:0)
Fusion.NetworkDictionary`2[K,V].Find (K key) (at Fusion/Fusion.Runtime/Misc/NetworkDictionary.cs:439)
Fusion.NetworkDictionary`2[K,V].ContainsKey (K key) (at Fusion/Fusion.Runtime/Misc/NetworkDictionary.cs:267)
```

This can happen if you have a networked collection field in a class that isn’t a NetworkBehaviour (even if it has a [Networked] attribute, it won’t complain).

#### ILWeaverException

Error example:

`
error Exception thrown when weaving Assembly-CSharp
error Fusion.CodeGen.ILWeaverException: Failed to weave behaviour SomeClass
error ---> Fusion.CodeGen.ILWeaverException: System.Void SomeClass::rpcMethod(Fusion.NetworkRunner): Instance RPCs not allowed for this type
`

This happens after compilation, and is the Fusion IL-weaver that’s trying to generate code for things like networked properties and RPCs. The weaving will fail if you’re doing something unsupported, like using a non-basic type in an RPC/Networked property/INetworkStruct, or, like in the example above, trying to use a non-static RPC in a non-network-instanced class (meaning not a NetworkBehaviour). This print many more, very unreadable lines of errors, the information you need is usually in the third line (the one containing Instance RPCs not allowed for this type in the example), that’ll tell you where the error is and what is wrong.

#### ContinueWith weirdness

Some of Fusion’s methods will return a Task, that can be chained to other tasks. You need to use this to get a completion callback on certain tasks, like Runner.StartGame. For this, you use .ContinueWith to chain a closure onto the end of the task. You just need to keep in mind that this closure will not be invoked on the main Unity thread, but on a background thread, which can lead to unsuspected outcomes. In general, you can’t use or touch any Unity-managed objects in this closure, meaning no Debug.Log, no changing properties or fields on objects, etc. Pretty much the only thing you should do is to change a local variable (not class field) to signal that the task is done. Anything else will lead to pain and confusion. Example:

```c#
IEnumerator startGameRoutine() {
    var done = false;
    
    runner.StartGame(...).ContinueWith(task => done = true);
    
    while (!done) { yield return null; }
    
    Debug.Log("Game started");
}
```

#### NullRefException on Spawn

Error example:

```
NullReferenceException: Object reference not set to an instance of an object
Fusion.NetworkRunner.Spawn(Fusion.NetworkObject prefab, System.Nullable1[T] position, System.Nullable1[T] rotation, System.Nullable`1[T]
```

This can happen if you try to spawn an object from a runner, but the runner is not running. Were you expecting a nice error message? Well if there was, this document wouldn’t be necessary.

#### Network object disabled and/or invalid

All NetworkObjects have to be attached to the runner. When this happens on a client, it will try to find the object on the host, and if it doesn’t exist, I’ll be disabled. A NetworkObject is valid when it has been attached and connected to the object on the host. You can check if the NetworkObject is attached and valid by checking the .IsValid property, or the Is Valid checkbox in the inspector.
If the NetworkObject is not valid, but not disabled, it means that it hasn’t yet been attached. Generally you use the RegisterSceneObjects method on the NetworkRunner to attach all NetworkObjects in a scene. Alternatively you can use the Attach method directly.
If the NetworkObject is not valid and disabled, it means that an attach has been attempted, but the object wasn’t found on the host. The object is found by its NetworkTypeID, which is a combination of scene number, object index and load index, and is accessible by the .NetworkTypeID property, and also visible in the inspector. One reason that it couldn’t get attached could be that the two clients disagree on which scene it’s in, since the NetworkTypeID won’t match if the scene numbers are different. In that case, check the SceneRef sent to RegisterSceneObjects. It might also be the object index that’s mismatched; in that case, remember to sort the list of NetworkObjects sent to RegisterSceneObjects with the NetworkObjectSortKeyComparer.Instance sorter.
In general, use the SceneRef.FromPath(scene.path) instead of SceneRef.FromIndex(scene.buildIndex), since buildIndex might vary between editor and builds, or different builds, if different scenes are enabled.

#### Object not set error

Error example:
`InvalidOperationException: Behaviour not initialized: Object not set.`

This happens if you call an RPC from a NetworkBehaviour without a NetworkObject, or a NetworkObject that haven’t been spawned/attached yet.
One case this can happen is if you’ve changed a class from a MonoBehaviour to a NetworkBehaviour, but haven’t reimported the prefab/scene it’s in.

#### Operation is not valid due to the current state of the object

Error example:
`InvalidOperationException: Operation is not valid due to the current state of the object.`

This happens when you call a method on a networked collection (NetworkArray, NetworkDictionary, etc), that have had its memory corrupted. Good luck finding out why.

#### GetChangedTickForBehaviour AssertException

Error example:

```
AssertException: Exception of type 'Fusion.AssertException' was thrown.
  at Fusion.Assert.Check (System.Boolean condition) [0x00008] in <b591dcdcca1c4ecaad8796465a118533>:0 
  at Fusion.NetworkObjectHeaderSnapshotRef.GetChangedTickForBehaviour (System.Int32 index) [0x00001] in Fusion\Fusion.Runtime\Object\NetworkObjectHeaderSnapshot.cs:19 
  at Fusion.Timelines.UpdateObject (Fusion.NetworkObjectMeta obj) [0x0002d] in Fusion\Fusion.Runtime\Simulation\Render\Timelines.cs:135 
  at Fusion.Timelines.Update () [0x00034] in Fusion\Fusion.Runtime\Simulation\Render\Timelines.cs:116 
  at Fusion.Simulation+Client.RecvPacket () [0x000b3] in Fusion\Fusion.Runtime\Simulation\Simulation.Client.cs:295 
  at Fusion.Simulation.Fusion.Sockets.INetPeerGroupCallbacks.OnNotifyData (Fusion.Sockets.NetConnection* c, Fusion.Sockets.NetBitBuffer* buffer) [0x0005f] in 
```

Something’s broken, disable objects until it stops happening.
I’ve had this happen when I used `RegisterSceneObjects` without sorting the array by `NetworkObjectSortKeyComparer.Instance`.

#### RPC called as normal method instead of being sent, ignoring target and InvokeLocal

Check that

* The class containing the method is a NetworkBehaviour (SimulationBehaviour doesn’t work)
* All the types are supported to be sent over the network
* The method name starts or ends with “rpc” (case is ignored)

#### ChangeDetector doesn’t detect changes to a networked property

Check that

* If the concrete instance of the class is a subclass of the class that defines the property, a change detector is not going to detect changes if the property is private, even if the change detector is located in the parent class. Make it public or protected.
* If copyChanges is set to false, it will only detect changes compared to its initial value.

#### Buffer should be null error

Error example:

```
AssertException: Exception of type 'Fusion.AssertException' was thrown.
AssertException: Buffer should be null
```

This happens when a client joins, and there are too many NetworkObjects in the scene. The limit is around 1000 network objects. Either try to reduce the number of objects, or set up area of interest, so that clients only get send networked objects close to them.

#### AssertException: GetTypeKey for ...

IL Weaver issue, try running the IL Weaver manually.

NetworkObjects with Area of Interest enabled not appearing within the Area of Interest

Check that

* The object has a NetworkTransform
* If it’s parented under an object without a NetworkTransform, make sure that object is in position 0, 0, 0!

### When all else fails...

Try to

* Change your code slightly to force a recompile and re-IL weave. Maybe Photon messed up your code, just give it a chance to correct its mistake.

## Testing

Generally the way to test the game is to make a build, and have one client running in the editor, and one running a build. It’s highly recommended to set up a build script that will build and launch the game to increase iteration times.

### Multi-peer mode

Fusion has a multi-peer mode, that can be used to run multiple clients in the same Unity editor. It’s enabled by changing the mode setting from single to multipeer in the Fusion config.
It’s generally a big bag of hurt and if you want to try it out, beware that this feature is basically undocumented. You can look at their “animations” sample code, that has a working multi-peer, but that’s pretty much all you’re going to get, and making it work properly for just slightly more complex setups is an exercise in futility.

## Security

This last topic might not seem important at first, but there’s a few good things to keep in mind, that might save you from having big problems later. Security refers to a broad array of issues your networking implementation might have, like cheating, intrusion, denial of service, etc., and have levels of severity depending on your setup (peer to peer between friends is probably not going to have big problems with security issues, but if you’re maybe going to be hosting games on your own server in the future, lots of interesting problems might appear). So while this is far from exhaustive, here’s a few things to keep in mind:

### Always sanitise client input

All input coming from a client, wether it’s from the player or from game systems, has to be treated as a potential attack vector, since a mod might put anything it wants in the data stream. Generally input from the client is networked input and RPCs. Here’s an example of an unprotected RPC:

```c#
enum DamageType { none, physical, magical }

[Rpc]
void rpcDealDamage(NetworkObject to, int amount,  int type) {
    dealDamageInternal(to.GetComponent<Unit>(), amount: amount, type: (DamageType)type);
}
```

Read the code and think for a while about what potential issues this contains. Here we go, categorized into different kind of security issues:

* Game rules: The client can freely deal damage to any unit they pass in. Is the unit in range? Is it an enemy unit? These things should be checked.
* Game rules: The client decides how much damage they’re going. A mod could be installed that made all the players attacks do int.MaxValue damage.
* Stability: Does the to unit even exist? Does it have a Unit component? These things should be checked, or you’re going to end up with NullReferenceExceptions.
* Correctness: Is type a valid DamageType? What happens if the value is 5? The cast to an enum is not going to fail, it’s just going to be an invalid value, and you can’t expect that dealDamageInternal is going to handle that correctly. Use Enum.IsDefined to check the value.
* Denial of service: This is not specific to this example, but if dealDamageInternal is a computation-heavy method, if the client starts spamming this message as fast as it can, it can overload the host. Have fast-paths out of the method for cases like 0 damage dealt.
