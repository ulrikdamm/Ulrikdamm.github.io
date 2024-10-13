---
layout: page
title: "The Unity Entities ECS Guide"
permalink: /unity-entities-guide
---

*Work in progress*

This is going to be a guide on using Unity’s new ECS system. A lot of information online talks about outdated versions of the system, and many guides assume a pure-ECS environment, so this is going to be a quick guide on using ECS to speed up and simplify things in an existing gameobject-based project.

## What is ECS

An ECS system is a very typical way to structure code for game development. A game scene will consist of Entities in the world with Components attached to give them functionality, and Systems that run to modify the state of those Components.

### The current state of ECS in Unity

Unity have from the beginning had an ECS system; the GameObject is the Entity, which is a class on which you can add/get/remove Components (`AddComponent`/`GetComponent`/etc), Components usually being `MonoBehaviour`s, which is a subclass of the Component class. The System part is a bit more hidden, but it is for example the Update system, which calls the `Update` method on each Component, which allows themselves to update their state.

This system works pretty well because it’s simple, and easy to work with for programmers used to object-oriented programming. It is, on the other hand, also broken enough to barely be able to be called an ECS.

First of all, the Entity is usually just an ID, with no other associated state. In Unity’s system, that ID is wrapped in a GameObject class, and your ID is a reference to that class. That class only contains things like the name of the object and whether it’s activated or deactivated. That state should really be in Components, since that’s where the state of the Entity is supposed to be. It’s also impossible to create a GameObject with a Transform Component.

Second, Components are supposed to be pure state. It’s just a collection of data about the Entity it’s connected to. But in Unity’s case, the Systems got implemented not as separate objects, but as methods on the Components, which just blurs the line of what is a Component and what is a System.

Last of all, this way of structuring an ECS is just not performant. Every part of the system is a heap-allocated class, which means that it lives in its own place in memory, and something wants to access another part of the system, it will have to jump to another place in memory, usually jumping through a few places to find it. Instantiating new Entities and adding/removing components is very slow, because it requires allocating and instantiating C# objects. Running the Update System requires going through all the Component objects and calling a method by name. And of course, since all Systems have global access to every GameObject and Component currently instantiated (through `Transform.Find`, `FindObjectByType`, singletons, etc), there’s no way this could run on a background thread, meaning everything will have to run sequentially on the main thread.

Another way it is not only not performant, but also hard to use, are cases where you want to access a list of Component, instead of running a System on one Component at a time. Say you want to find the closest enemy unit. What you want to do is to get a list of all units, filter out the non-enemies, and then find the distance to each one of them, and select the smallest. You can `FindObjectsByType<Unit>` and loop through them, but doing so every frame, possibly many times per frame, is going to be horrendously slow, not even speaking of all the memory allocations it’s going to make. And what if you want to find only the units that have health components? There’s no `FindObjectsByType<Unit, Health>`. So what you do instead is that you create a UnitsManager objects, where units will register themselves, so that you can add them to a list, maybe even a separate list for friendlies and enemies if you know you’re going to be looping through them a lot. And remember to clean out the lists when the objects get destroyed, or you’re going to have NullReferenceExceptions!

Anyway, enough talking about the current state of things. Time to see how it can be done better.

### A new way of doing ECS

Unity’s new ECS system (called Entities) is not a fixup of this old system, but a radical departure from it, and it wants to fix all of the problems listed above. An Entity is not a class anymore; it’s just a struct with and ID. It requires no allocations, and come with no state. No `Transform`, no name. Just an ID.

Components are what we described earlier as the ideal component: it’s just a C# struct with data, that you can attach to Entities. No Update method, no `Awake`/`Start`, it’s just a struct. Want a Component to keep track of health of an Entity? Just make a struct with a float health; and you’re done. Accessing and modifying this health will be done from Systems.

A System in Entities is a struct or a class, with one method that gets invoked at a certain time in each frame. You can’t start Systems from other Systems, you can only schedule them to happen before or after another System. If a System doesn’t access any global state, but instead statically specifies which components it wants to access, it will be able to be run asynchronously in the background. If the System writes new data to Components, other Systems wanting to read from those Components will be scheduled to run after that System has completed. If multiple Systems want read-only access to Component data, they can be scheduled to be run at the same time on different background threads.

Whereas in the standard Unity system are desentivized from using a lot of `GetComponent` and `FindObjectsByType`, instead wanting to cache the responses to these, in the Entities ECS, this pattern is actually encouraged. Component data is not scattered around in memory in heap-allocated C# objects; instead they are packed tightly in arrays, making looping over them very efficient and cache-friendly. Say that, as in the previous example, you want to find all objects that have a Unit and Health component, this is as easy as making an `EntityQuery<Unit, Health>`, and then looping over it (syntax-wise it’s not as simple as that, but we’ll get to it). Doing it this way is as efficient as for-looping over an array, since that it essentially what it’s doing. There are also ways to optimize storage further, by chucking the data based on specific properties.

This more efficient approach changes the way you want to structure your objects. In the traditional system, you want to chunk things together in bigger components, so you avoid having a lot of references, and potential for nullref-exceptions. Like a Health Component that tracks health, exposes events for health changing, handles unit death, and possibly also defence, blocking, dodging, hit effects, sounds, etc. In Entities ECS, these big Components are very much discouraged. Since splitting things into multiple Component has such a low overhead now, we can instead make a Health Component, a Physical Defence Component, a Component for Magical Defence, Dodge, Block, and so on. Then, the System for dealing damage can find the relevant Components, and if a Component doesn’t exist, use a default value.

It’s even encouraged to have empty Component, as a form of Boolean data. Take the `GameObject.SetActive`/`GetActive`/`activeSelf`. In Entities ECS, this active/deactive state is moved out from the Entity to a Component. Entities are considered Active by default, disabling it is just adding a Disabled Component to it (Along with checking for Components on an Entity, adding/removing Components is also very efficient). The Disabled Component is just an empty struct, since just adding the Component communicates everything it needs to. Empty Components are called Tag Components.

One last thing that Entities adds is the ability to run multiple separate Entity-worlds at the same time. In the Traditional ECS, if you create a GameObject with a Health Component, it will always show up in a `FindObjectsByType`, even if it’s in another scene. In Entities ECS, all queries are performed on a World, and you can always just create a new empty World, and add Entities, Components and Systems to that, and it will be completely cut off from all other Worlds.

## A basic example of Entities in action

Enough of the theory, it's time to show how to actually use this thing.

### Creaing your first Entity

The first thing you need to get the Entities ECS started, other than installing the Entities package, is a World to add your Entities into. You can always just create a new World, but the easiest way is to use the default World, which is accessed statically as `World.DefaultGameObjectInjectionWorld`. From that you can get the EntityManager through the `.entityManager` property. This is what you use to create an Entity:

```c#
using Unity.Entities;

var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;
var entity = entityManager.CreateEntity();
```

As simple as that, you now have a new, blank Entity. If you open the Entities Hierarchy window in Unity (Window > Entities > Hierarchy), you will be able to find it in the bottom with a name like `Entity 0:1`. This is how Entities are identified. Entities can be given a name with the SetName method, which is a shorthand for adding a Name Compoent to it. Remember, the Entity itself have no state. It doesn't keep track of its own name or Components. The Entity is just an ID; what Components it has is controlled by the EntityManager. That's why all the methods for querying/modifiying Entities are on the EntityManager, not the Entity itself.

```c#
entityManager.SetName(entity, "My First Entity");
```

It will now show up in the Entity Hierarchy with that name. If you want to disable it, equvivalent to `gameObject.SetActive(false)`, that's just adding a Disabled component to it:

```c#
entityManager.AddComponent<Disabled>(entity);
```

It will now show up as disabled in the hierarchy. Enabling it again is just removing the Component:

```c#
entityManager.RemoveComponent<Disabled>(entity);
```

### Creating Entity Components

Creating a component to associate some state with an Entity is just making a struct, that conforms to the IComponentData interface.

```c#
struct Health : IComponentData {
    public float current;
    public float max;
}
```

And to attach it to your Entity:

```c#
entityManager.AddComponentData(entity, new Health { current = 100, max = 100 });
```

Notice that if you want to actually provide some starter-data to the component when you add it, you use AddComponentData instead of just AddComponent. The same could be accomplished by:

```c#
entityManager.AddComponent<Health>(entity);
entityManager.SetComponentData(entity, new Health { current = 100, max = 100 });
```

AddComponentData just checks that a Component doesn't already exist, and SetComponentData checks that it does exist.

To get Component values from an Entity, you use GetComponentData, and you can then change the struct, and apply it via SetComponentData

```c#
var health = entityManager.GetComponentData<Health>(entity);
health.current = 50;
entityManager.SetComponentData(entity, health);
```

And to remove the Health Component from the Entity:

```c#
entityManager.RemoveComponent<Health>(entity);
```

One thing to note with IComponentData structs is that they have to be _blittable_. That means that it has some restrictions on what types you can use, notably no reference-types, meaning no strings, arrays or object references. The alternatives to these are `FixedString#Bytes`, `NativeArray` and `UnityObjectRef`.

### Finding Entity Components

To find components, you can create `EntityQuery`s. Queries can be configued with groups of Components, exclusion of components, read-only/read-write, and more complicated things like filtering based on Shared Components (a later topic), but for now we'll just create a query to find all our `Health`s.

```c#
var query = entityManager.CreateEntityQuery(typeof(Health));
```

If you want to loop over all the entities in a query, and perform some logic, you'll have to create a System to do that. If you want to access the Components outside of a System, you'll have to instead put them into an array. This doesn't mean a C# array though; everything in Entities uses `NativeArray`s. They're like a C# array, but with more low-level control over allocations. To create a NativeArray, you have to define which Allocator you want to use. For this, we'll use an temporary allocator. One thing to keep in mind with native collections is that they are not garbage-collected; you'll have to `Dispose` of them yourself.

```c#
using Unity.Collections;

var entities = query.ToEntityArray(Allocator.Temp);

foreach (var entity in entities) {
    var health = entityManager.GetComponentData<Health>(entity);
    Debug.Log($"Entity {entity} has {health.current} / {health.max} health");
}

entities.Dispose();
```

In this case, we ask the `EntityQuery` to create for us a NativeArray of all the Entities that matches the query. We then know that all of those entities will have a Health component, and we can get each component via GetComponentData.

Another option is to get arrays of the Component values:

```c#
var healths = query.ToComponentDataArray<Health>(Allocator.Temp);

foreach (var health in healths) {
    Debug.Log($"Entity has {health.current} / {health.max} health");
}

healths.Dispose();

```

Notice that since Health is just a pure-data struct, there's no way to get the Entity from a Component.

If we want to create a combined query, like for all entities that both have a Health and a Defence component, we can add this to our query:

```c#
var query = entities.CreateEntityQuery(typeof(Health), typeof(Defence));
```

And then get an array for each Component. To get the number of results, we can use `.CalculateEntityCount()`.

```c#
var count = query.CalculateEntityCount();
var healths = query.ToComponentDataArray<Health>(Allocator.Temp);
var defences = query.ToComponentDataArray<Defence>(Allocator.Temp);

for (var i = 0; i < count; i++)
    var health = healths[i];
    var defence = defences[i];
    
    Debug.Log($"Entity has {health.current} / {health.max} health and {defence.physical} physical defence");
}

healths.Dispose();
defences.Dispose();
```

Even though this is very possible, for the most part, you won't actually be executing queries manually. Instead, you'll be creating Systems that runs once per frame. The systems will have some handy methods for more easily creating queries.

### Creating Systems

To actually add some logic to this, like `Debug.log`ing the health every frame, we create a System that works on `Health`s. The easiest way to create a System is to make a class that subclasses SystemBase.

```c#
partial class HealthLogger : SystemBase {
    protected override void OnUpdate() {
        
    }
}
```

SystemBase has two requirements of its subclasses: it must implement the `OnUpdate` method, which is the method that will get called when the System runs, and the class must be a `partial` class (this is just an implementation detail in how the ECS is implemented).

There's no need to register or start the System, it will automatically be detected, and added to the update loop. So just adding this class to your project will start executing OnUpdate once each frame.

To actually run some code per Component, you can use an EntityQuery like before, but SystemBase comes with a `Entities.ForEach` method, which can be used to loop over Components, and can also potentially take care of splitting the work onto multiple threads.

```c#
partial class HealthLogger : SystemBase {
    protected override void OnUpdate() {
        Entities
            .ForEach((in Health health) => {
                Debug.Log($"Entity has {health.current} / {health.max} health");
            })
            .Schedule();
    }
}
```

Notice the `in` keywork here for the Component. This means that any changes you make to the Component will be discarded, meaning that it'll be read-only access. If you want to actually change the Component, you would change `in` to `ref`:

```c#
partial class HealthLogger : SystemBase {
    protected override void OnUpdate() {
        Entities
            .ForEach((ref Health health) => {
                health.current -= SystemAPI.Time.DeltaTime;
                Debug.Log($"Entity now has {health.current} / {health.max} health");
            })
            .Schedule();
    }
}
```

The ForEach syntax uses the Unity Job System to create jobs for the lambda function provided to ForEach. The Job System is what allows code to be both split up to be run parallel in the background, and also to be Burst-optimized, both of which it will do by default. This is where there's huge performance improvements to be gained over regular C# code.

Notice that SystemAPI.Time.DeltaTime is used instead of Time.deltaTime. SystemAPI is a way of getting various information inside Entities, we'll see more of that later.

In the case where you want to interact with the GameObject-world, however, you're going to have to disable some of these optimizations. If you're trying to squeeze the maximum amount of performance out of your code, you want to prevent calling main-thread only methods or accessing global/GameObject state in your jobs. For interoperating with an existing GameObject codebase, it might sometimes be required though. What you want to do in these cases is to, as far as possible, split your jobs into pure data processing jobs and slower GameObject-interaction jobs. This can be done by making multiple different Systems:

```c#
partial class HealthSubtractor : SystemBase {
    protected override void OnUpdate() {
        Entities
            .ForEach((ref Health health) => {
                health.current -= SystemAPI.Time.DeltaTime;
            })
            .ScheduleParallel();
    }
}

partial class HealthBarsUpdater : SystemBase {
    protected override void OnUpdate() {
        var healthBarsUI = FindObjectByType<UIHealthBars>();
        healthBarsUI.clear();
        
        Entities
            .ForEach((in Health health) => {
                var healthBar = Object.Instantiate(healthBarsUI.prefab); // error!
                healthBar.show(health);
                healthBarsUI.addHealthBar(healthBar);
            })
            .Schedule();
    }
}
```

Now the pure data-processing job can be run in the background, and we can even swap out .Schedule for .ScheduleParallel, which will split up the job to run on multiple threads. The second System, however, is not going to work just yet. Unity is by default running all Entity jobs on a background thread, and in this case, and our method is interacting with the main-thread only GameObject world. This is possible, but we'll have to tell the System to disable some optimizations.

```c#
partial class HealthBarsUpdater : SystemBase {
    protected override void OnUpdate() {
        var healthBarsUI = FindObjectByType<UIHealthBars>();
        healthBarsUI.clear();
        
        Entities
            .WithoutBurst()
            .ForEach((in Health health) => {
                var healthBar = Object.Instantiate(healthBarsUI.prefab);
                healthBar.show(health);
                healthBarsUI.addHealthBar(healthBar);
            })
            .Run();
    }
}
```

This should now work correctly. The two differences are 1) the `.WithoutBurst()` marker disables Burst-optimizations 2) The invocation is ended with `.Run()` instead of `.Schedule()`. `Schedule()` is going to schedule the job to run at a time where the system thinks that it's approprite (taking into account dependencies on other jobs, read/write access to data, thread occupancy, etc), whereas `Run()` is just going to execute it as soon as possible, and block while waiting for the result, ensuring that no other GameObject-touching code can run at the same time.

Notice at _as soon as possible_ is not _immediately_. All jobs will check their dependencies before running, and make sure that they're safe to access. In this case, if the `HealthSubtractor` job is running long, and is not done executing in the background when `HealthBarsUpdater` starts running, `HealthBarsUpdater` will stall, and wait for the `HealthSubtractor` background job to finish, before executing. It knows to wait, since `HealthSubtractor` has a `ref Health` argument (meaning it wants write-access to all `Health`s) and HealthBarsUpdater has an `in Health` (which means it wants read-access to all `Health`s). Since executing `HealthBarsUpdater` immediately could mean that the health subtractor job was only half-finished, and access to `Health`s could result in race conditions, the reader job will patiently have to wait. If the `HealthSubtractor` job, on the other hand, only had read-access to Health (`in Health`), there would be no problem running the two at the same time, since just reading data is not going to result in race conditions.

### Scheduling and races

In general, the Entities system is very good at ensuring that you're not accessing your data at time where you shouldn't. Usually the only downside is that it might put in syncronization points, which can block threads for a while, which is not amazing for performance. You should be entirely free from data races however, the worst case scenario is an exception thrown, in case you're doing something it can't guarantee is safe to do.

As an example of blocking threads, if you have the `HealthSubtractor` system running in the background, but in the Update method of a MonoBehaviour you try to restore all healths like this:

```c#
foreach (var entity in myListOfStoredEntities) {
    var health = entityManager.GetComponentData<Health>(entity);
    health.current = health.max;
    entityManager.SetComponentData(entity, health);
}
```

This won't cause any problems, but if you take a look in the profiler, you will be able to see that the code is actually blocking the main thread if the `HealthSubtractor` job is running. This means no data races, but if `HealthSubtractor` is a heavy, long-running job, it might cause frame stutter if you're waiting for it on the main thread at an unfortunate time (one quick fix could be to move it from Update to LateUpdate, so that at least all other Update methods can run before blocking, but ideally, you want to create a System to do this).

## ECS transforms

To use ECS transforms, remember to add `using Unity.Transforms;`

In GameObject-land, position/rotation/scale of objects are done with the Transform component, which is default added to all GameObjects. Other than that, the Transform component also handles the hierarchy of objects with the `.parent` property, so that when you add a parent to a Transform, it might have a different `.position` and `.localPosition`. The final transformation matrix after applying all properties can be retrieved with the `.localToWorldMatrix` property.

In Entities ECS, this functionality has been split up. The base component for Entities with position is the `LocalToWorld` Component. Its only value is a single local-to-world transformation matrix, and it has a few properties for more easily accessing position and rotation. No transform hierarchies in this, for that you need to use the separate `Parent`, `Child` and `LocalTransform` Components. _TODO: Transform hierarchies_

## Entity Authoring

Until now, we've been looking at creating Entities from code, but usually GameObjects are not made in code, but put into scenes and prefabs. Current, Unity has no way of creating Entity scenes in the editor (though it sounds like it's coming with Unity 7, but that's still years away as of writing). It does have something else to compensate for it though, with is Entity Authoring components. You can create MonoBehaviour components, that describe how to create corresponding Entities, and Unity will create these at build-time through a process called Baking. A scene made of Entities will be much faster to load and initialize than a GameObject-scene.

Too create an Entity Authoring Component, you add a normal MonoBehaviour, except that it's one that's only going to exist in the editor, never in the actual build. It just need to look like you're creating a MonoBehaviour equivalent of your Entity Component. You can also add all the usual editor-features like gizmo drawing. Here's an authoring component for a Health Entity:

```c#
public class HealthAuthoring : MonoBehaviour {
    public int initial;
    public int max;
}
```

That's all you need for the authoring component. A converter from this to an Entity Component still needs to be created manually though, so we need to also create a Health Baker. A Baker takes in an Autoring Component, creates an Entity automatically, and then allows you to add Components to that Entity.

```c#
public class HealthBaker : Baker<HealthAuthoring> {
    public override void Bake(HealthAuthoring authoring) {
        var entity = GetEntity(TransformUsageFlags.None);
        
        entity.AddComponentData(entity, new Health {
            current = authoring.initial,
            max = authoring.max
        });
    }
}
```

The Baker subclass takes the Authoring Component as a generic parameter, and will call the Bake method whenever an Entity scene is baked. An Entity is created automatically before the Bake method (called the Primary Entity), and we can access it via the GetEntity method.

Now, since a GameObject always have an associated Transform component, but for Entities transforms are both optional and more flexible, you might or might not want to copy over the values of the Transform Component on the GameObject to the new Entity. The TransformUsageFlags parameter on the GetEntity method allow you to specify that. If you don't care about the position/rotation/scale of the object, you can specify `.None`, and no components will be created for you on the Entity, and all Transform component values will be ignored. If you want to keep the position, but it's not going to be moving around, you can specify `.Renderable`, which will give the Entity a LocalToWorld Component, and apply the Transform position and rotation to it. If you want to copy over the position, and you want to move the Entity around, and have it stay under its parent Transform, you can specify `.Dynamic`, and it will create `LocalToWorld`/`LocalTransform`/`Child`/`Parent` components as needed.

Other than the components created from the transform flags, it's also going to add a few Shared Components to tag it with the scene it was created from, along with some other Components.

Now, to actually use this in a scene, you need to create a SubScene for all the Entities. Inside your normal scene, you can right click > New Sub Scene > Empty Scene..., and then choose where to save it. Then you can start putting your Authoring Components under the subscene. When you start play mode in the editor, the baking process will run, the Authoring GameObjects will be discarded, and the Entities and Components will be added to the World.

## Associating Entities with GameObjects

Many people are probably dreaming of wiping the slate completely clean, and starting their project from the beginning as a pure-ECS project, leaving the GameObject world completely behind. For almost exactly as many people, however, this dream is pretty much unattainable, since rewriting a multi-year project from scratch is just not feasible. This means that we'll have to have some kind of interaction between the Entity world and the GameObject world.

Unity doesn't provide some easy built-in way to associate an Entity and a GameObject, so it's something we'll have to do yourselves. There's also a few different ways to approach it, and one Entity might not always == one GameObject. As discussed before, GameObjects are generally heavier to handle than Entities, especially being more costly to create and destroy. Entities on the other hand can enter and exit reality like some sort of subatomic particle. But the most common case in Entity/GameObject shared codebases is having an Entity and a GameObject refer to the same thing, or actually more common, an Entity and a MonoBehaviour refer to the same thing. A unit GameObject that have a Unit and Health might not translate to an Entity with a UnitData and HealthData component, but instead an Entity with a UnitData component and another Entity with a HealthData component. Each can work, it's up to what makes the most sense in the specific scenario.

When you have an Entity/GameObject pair, you might want to choose one of them as being the main object, and the other to be a "shadow" object, following the main object. We want to have things like creating the other when one is created, and destroying both when one is destroyed. When you have an existing GameObject-based project that you want to add Entities into, what you probably want to do is have the GameObject/MonoBehaviour be the main object, and it will handle creating/destroying it's associated Entity.

Say we start with a component like this:

```c#
public class Unit : MonoBehaviour {
    [SerializeField] bool _stunned;
    
    public float attackDamage;
    public float attackCooldown;
    
    float? currentCooldown;
    
    public bool stunned {
        get => _stunned;
        set {
            if (_stunned == value) { return; }
            _stunned = value;
        }
    }
    
    public void dealDamage(Health target) {
        if (stunned || currentCooldown.HasValue) { return; }
        
        target.takeDamage(attackDamage);
        currentCooldown = attackCooldown;
    }
    
    void Update() {
        if (currentCooldown is {} cooldown) {
            cooldown -= Time.deltaTime;
            
            if (cooldown <= 0) {
                currentCooldown = null;
            } else {
                currentCooldown = cooldown;
            }
        }
    }
}
```

To able able to access its data in the Entities system, we can have it create an manage an associated Entity with some component data:

```c#
public class Unit : MonoBehaviour {
    [SerializeField] bool _stunned;
    
    public float attackDamage;
    public float attackCooldown;
    
    float? currentCooldown;
    
    Entity entity;
    
    EntityManager entityManager => World.DefaultGameObjectInjectionWorld.EntityManager;
    
    void Awake() {
        entity = entityManager.CreateEntity();
        entityManager.SetName(entity, name);
        
        entityManager.AddComponentData(entity, new UnitAttackData {
            damage = attackDamage,
            cooldownDuration = attackCooldown
        });
    }
    
    public bool stunned {
        get => _stunned;
        set {
            if (_stunned == value) { return; }
            _stunned = value;
            
            if (stunned) {
                entityManager.AddComponent<UnitStunnedData>(entity);
            } else {
                entityManager.RemoveComponent<UnitStunnedData>(entity);
            }
        }
    }
    
    public void dealDamage(Health target) {
        if (currentCooldown.HasValue) { return; }
        
        target.takeDamage(attackDamage);
        currentCooldown = attackCooldown;
    }
    
    void Update() {
        if (currentCooldown is {} cooldown) {
            cooldown -= Time.deltaTime;
            
            if (cooldown <= 0) {
                currentCooldown = null;
            } else {
                currentCooldown = cooldown;
            }
        }
    }
    
    void OnDestroy() {
        entityManager.DestroyEntity(entity);
    }
}

struct UnitStunnedData : IComponentData {}

struct UnitAttackData : IComponentData {
    public float damage;
    public float cooldownDuration;
    
    public float? cooldown;
}
```

Now we can start acessing data about the unit from entity systems, like a system for displaying icons above all units that have been stunned:

```c#
partial class StunnedIconsUpdater : SystemBase {
    protected override void OnUpdate() {
        var ui = FindObjectByType<UI>();
        ui.stunIcons.clear();
        
        Entities
            .WithoutBurst()
            .ForEach((in UnitStunnedData stunned) => {
                ui.stunIcons.show(); // position?
            })
            .Run();
    }
}
```

Here we just run into one thing that we haven't exposed yet to the Entities system, which is the position of the unit. For this, there is a special way to efficiently access it.

*TODO finish article*
