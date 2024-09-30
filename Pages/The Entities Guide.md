---
layout: page
title: "The Unity Entities ECS Guide"
permalink: /unity-entities-guide
---

# The Unity Entities ECS Guide

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

```
using Unity.Entities;

var entityManager = World.DefaultGameObjectInjectionWorld.EntityManager;
var entity = entityManager.CreateEntity();
```

As simple as that, you now have a new, blank Entity. If you open the Entities Hierarchy window in Unity (Window > Entities > Hierarchy), you will be able to find it in the bottom with a name like `Entity 0:1`. This is how Entities are identified. Entities can be given a name with the SetName method, which is a shorthand for adding a Name Compoent to it. Remember, the Entity itself have no state. It doesn't keep track of its own name or Components. The Entity is just an ID; what Components it has is controlled by the EntityManager. That's why all the methods for querying/modifiying Entities are on the EntityManager, not the Entity itself.

```
entityManager.SetName(entity, "My First Entity");
```

It will now show up in the Entity Hierarchy with that name. If you want to disable it, equvivalent to `gameObject.SetActive(false)`, that's just adding a Disabled component to it:

```
entityManager.AddComponent<Disabled>(entity);
```

It will now show up as disabled in the hierarchy. Enabling it again is just removing the Component:

```
entityManager.RemoveComponent<Disabled>(entity);
```

### Creating Entity Components

Creating a component to associate some state with an Entity is just making a struct, that conforms to the IComponentData interface.

```
struct Health : IComponentData {
	public float current;
	public float max;
}
```

And to attach it to your Entity:

```
entityManager.AddComponentData(entity, new Health { current = 100, max = 100 });
```

Notice that if you want to actually provide some starter-data to the component when you add it, you use AddComponentData instead of just AddComponent. The same could be accomplished by:

```
entityManager.AddComponent<Health>(entity);
entityManager.SetComponentData(entity, new Health { current = 100, max = 100 });
```

AddComponentData just checks that a Component doesn't already exist, and SetComponentData checks that it does exist.

To get Component values from an Entity, you use GetComponentData, and you can then change the struct, and apply it via SetComponentData

```
var health = entityManager.GetComponentData<Health>(entity);
health.current = 50;
entityManager.SetComponentData(entity, health);
```

And to remove the Health Component from the Entity:

```
entityManager.RemoveComponent<Health>(entity);
```

### Finding Entity Components

To find components, you can create `EntityQuery`s. Queries can be configued with groups of Components, exclusion of components, read-only/read-write, and more complicated things like filtering based on Shared Components (a later topic), but for now we'll just create a query to find all our `Health`s.

```
var query = entityManager.CreateEntityQuery(typeof(Health));
```

If you want to loop over all the entities in a query, and perform some logic, you'll have to create a System to do that. If you want to access the Components outside of a System, you'll have to instead put them into an array. This doesn't mean a C# array though; everything in Entities uses `NativeArray`s. They're like a C# array, but with more low-level control over allocations. To create a NativeArray, you have to define which Allocator you want to use. For this, we'll use an temporary allocator. One thing to keep in mind with native collections is that they are not garbage-collected; you'll have to `Dispose` of them yourself.

```
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

```
var healths = query.ToComponentDataArray<Health>(Allocator.Temp);

foreach (var health in healths) {
	Debug.Log($"Entity has {health.current} / {health.max} health");
}

healths.Dispose();

```

Notice that since Health is just a pure-data struct, there's no way to get the Entity from a Component.

If we want to create a combined query, like for all entities that both have a Health and a Defence component, we can add this to our query:

```
var query = entities.CreateEntityQuery(typeof(Health), typeof(Defence));
```

And then get an array for each Component. To get the number of results, we can use `.CalculateEntityCount()`.

```
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

```
partial class HealthLogger : SystemBase {
	protected override void OnUpdate() {
		
	}
}
```

SystemBase has two requirements of its subclasses: it must implement the `OnUpdate` method, which is the method that will get called when the System runs, and the class must be a `partial` class (this is just an implementation detail in how the ECS is implemented).

There's no need to register or start the System, it will automatically be detected, and added to the update loop. So just adding this class to your project will start executing OnUpdate once each frame.

To actually run some code per Component, you can use an EntityQuery like before, but SystemBase comes with a `Entities.ForEach` method, which can be used to loop over Components, and can also potentially take care of splitting the work onto multiple threads.

```
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

```
partial class HealthLogger : SystemBase {
	protected override void OnUpdate() {
		Entities
			.ForEach((ref Health health) => {
				health.current -= Time.deltaTime;
				Debug.Log($"Entity now has {health.current} / {health.max} health");
			})
			.Schedule();
	}
}
```

The ForEach syntax uses the Unity Job System to create jobs for the lambda function provided to ForEach. The Job System is what allows code to be both split up to be run parallel in the background, and also to be Burst-optimized. This is where there's huge performance improvements to be gained over regular C# code. In this case, however, we're using a Debug.Log inside our job, which is a method that can only run on the main thread. This is allowed, but it's going to prevent us from gaining a lot of the performance benefits. If you're trying to squeeze the maximum amount of performance out of your code, you want to prevent calling main-thread only methods or accessing global/GameObject state in your jobs. However, for interoperating with an existing GameObject codebase, it might sometimes be required. What you want to do in these cases is to, as far as possible, split your jobs into pure data processing jobs and slower GameObject-interaction jobs. This can be done by making multiple different Systems:

```
partial class HealthSubtractor : SystemBase {
	protected override void OnUpdate() {
		Entities
			.ForEach((ref Health health) => {
				health.current -= Time.deltaTime;
			})
			.ScheduleParallel();
	}
}

partial class HealthLogger : SystemBase {
	protected override void OnUpdate() {
		Entities
			.ForEach((in Health health) => {
				Debug.Log($"Entity now has {health.current} / {health.max} health");
			})
			.Schedule();
	}
}
```

Now the pure data-processing job can be run in the background, and we can even swap out .Schedule for .ScheduleParallel, which will split up the job to run on multiple threads (trying to do this on the HealthLogger job will results in errors, since Debug.Log will be called from a background thread, which is illegal).
