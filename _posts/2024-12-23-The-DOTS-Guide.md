---
layout: page
title: "The Unity DOTS Guide"
permalink: /unity-dots-guide
exclude: true
---

So, a lot of Unity developers have heard of DOTS, or the Data Oriented Tech Stack, but few have actually taken the plunge and started learning it. And that’s not really surprising, since it’s actually not too easy to get started with. There’s a lot of different parts to it, and none of them are particularly straightforward or intuitive. Documentation is pretty spread out, and focuses a lot of performance, rather than “what do I do with this thing?” Well, this is the guide to get started with DOTS, written from a practical point of view to actually get started using it in your existing Unity projects. 

## What is DOTS

The term DOTS cover a few different projects, the primary ones being the Job System, the Burst Compiler and the Entities package. These are only loosely coupled, so you can use any of them that fits your project. Here’s a few quick overview of these:

- The Job System can take pieces of code and turn them into self-contained jobs, which can be scheduled to be run in the background, allowing easy multithreaded code without worrying about things like locks and data races.
- The Burst Compiler can take pieces of code and, instead of running them on Unity’s Mono .NET runtime, compile it down to speedy native code, and even do some clever optimizations to make code run much faster (it’s not an exaggeration to expect 10-100x speedup!)
- The Native Collections package gives you collections like arrays, lists and dictionaries, like the ones that ships with C#, but in *unmanaged* versions. This means that they don’t use the garbage collector at all: instead they’re manually allocated and disposed off. They also come with some handy safety checks for use in multithreaded code. 
- The Entities package is a new way to do ECS as an alternative to GameObjects and MonoBehaviours. It’s doing away with the object oriented thinking, replacing it with a systems based approach, allowing many times more entities to exist.

These four packages together creates a whole new way of structuring your projects, and unlocks performance that was not feasible before.
Of course, you can just go and delete your game project you’ve been working on for the past three years and start all over in a pure DOTS codebase, but in reality, most of us would not like to have to completely rewrite all our code. The good news is that all this can be added to your project incrementally. Usually, the reason you want to start using DOTS is because parts of your game is slow, and you want to use DOTS to speed it up. Maybe you have just one method that is running painfully slow. For that, you can turn it into a Job, Burst-optimize it, and that’s it. You don’t have to completely convert all your code, you can just use it in this one place, and the rest of your project can be good old normal Unity code. Or you might be struggling with a lot of Update methods taking up precious frame time, even though they don’t do much; you can convert them to entities, and turn all those Update calls into a a single Job System.
While you *can* use each of these packages by themselves, in reality the Burst Compiler is mainly made for optimizing Jobs, the Native Collections are mostly for passing data in and out of jobs, and the Entities systems are very much made for running Jobs, so the Job System/Native Collections/Burst Compiler trio is the most fundamental part of DOTS, so this is where our guide will begin. After this, we’ll finally get to creating entities.
Alright, enough preamble, let’s start writing some fast code!

## Native Collections

The Native Collections package comes with replacements for the collection classes built in to C#. `Array`, `List` and `Dictionary`’s equivalent native collections are `NativeArray`, `NativeList` and `NativeHashMap`. These works pretty much the same way, with one big exception: they're not garbage collected, so you have to manually dispose them. If you allocate a list, and just forget about it, it will live on in memory until the process is terminated.

To use a `NativeArray`, which is an array with a fixed size, it looks like this:

```c#
using Unity.Collections;

var myArray = new NativeArray<int>(length: 10, Allocator.Temp);
myArray[0] = 10;
Debug.Log(myArray[0]);
myArray.Dispose();
```

As you can see, it works pretty much as a normal array, except that you have to call `.Dispose()` on it when you're done with it. If you forget to dispose it, Unity will let you know with a message in the console, but it won't clean it up for you.
Another thing to note is the `Allocator.Temp` that's passed in with the constructor. All native collections need to be associated with an allocator. There's three different ones, and it's fairly simple to know which one to use:

`Allocator.Temp` is for short-lived allocations (you're going to get a warning if they still haven't been `.Dispose`d after four frames)
`Allocator.TempJob` is like `Temp`, but can be used in jobs
`Allocator.Persistent` is for long-lived allocations

Another thing to note is that all these are structs, not classes, which means you can't null-check them, to see if they have been created. Instead you use the `.IsCreated` property.

That's more or less everything you need to know about Native Collections. There are many more details, and you can also create your own custom native collections using scary unsafe pointers, but we'll leave that for now.

### Disposable tricks

One handy thing I use to avoid forgetting to dispose collections is to (ab-)use C#'s `using` keyword. The normal way you use it is to create a block, that will automatically dispose of something at the end of it:

```c#
using (var array = myArray) {
	
} // NativeArray.Dispose is automatically called here
```

Another slightly newer thing you can do is to have it be disposed at the end of the current scope:

```c#
void doSomething() {
	var myArray = new NativeArray<int>(length: 10, Allocator.Temp);
	using var autoDisposeArray = myArray;
	
	...
} // NativeArray.Dispose is automatically called here
```

This way the array is always disposed when the method returns, no matter where in the method it happened.

## The Job System

The Job System is a package that allows you to create jobs, which are pieces of code that can’t access global state, but are instead confined to using only the input and output you’ve specified. This constraint allows the job to be run in the background, off of the main thread where all your usual logic happens, unlocking the multicore potential of the machine.
Normally if you’d want to run code in the background, you’d have to create a new thread, and then manually keep track of what is being read and written on that and all other threads, to avoid data races. For jobified code, this is not a concern. The system will take care of all those problems for you. If you schedule two jobs to write to the same data, the system will produce an error, and tell you to schedule them to run sequentially.

### Creating a simple Job

Creating a simple job is fairly easy. All jobs are structs that implements the `IJob` interface, and will get a method called when it's time to run the code:

```c#
struct MyJob : IJob {
	public void Execute() {
		Debug.Log("Running in a job!");
	}
}
```

To actually run the job, you create an instance of it, and call the `Run` method:

```c#
var job = new MyJob();
job.Run();
```

And that's all it takes. Running a job with the `Run` method is only going to run it on the main thread, however, and that's not why we're doing all this. Luckily, running a job in the background is just replacing `Run` with `Schedule`. Then it's going to return immediately with a `JobHandle`, which you can use to track the state of the job:

```c#
var job = new MyJob();
var handle = job.Schedule();
Debug.Log($"Job completed: {handle.IsCompleted}");
```

If you want to wait for a background job to complete, you can either keep checking its `IsCompleted` property, or you can call `.Complete()` on it, which will block until the job is done.

To actually pass some data in to the job, and have it do something useful, we add fields to the job struct, that we can then access in the `Execute` method:

```c#
struct SumJob : IJob {
	public int valueA;
	public int valueB;
	
	public void Execute() {
		var sum = valueA + valueB;
		Debug.Log($"Sum: {sum}");
	}
}

var job = new SumJob { valueA = 10, valueB = 20 };
```

What you can't do is modify these fields, and expect them to keep the changes when running the job:

```c#
struct SumJob : IJob {
	public int valueA;
	public int valueB;
	public int outSum;
	
	public void Execute() {
		outSum = valueA + valueB;
	}
}

var job = new SumJob { valueA = 10, valueB = 20 };
job.Run();
Debug.Log($"Sum: {job.outSum}"); // Prints: `Sum: 0`
```

This is because what you're working on inside the Execute method is a copy of the job struct, so all values you write to fields are not going to be reflected in the original.

To pass data out, you must use a native collection. Passing native collections along with the job allows you to read and write to them from within the job. For this small example, there's a very handy native collection: `NativeReference` is like a native array, except that it stores just one value. We can use this to pass our sum result out from the job:

```c#
struct SumJob : IJob {
	public int valueA;
	public int valueB;
	public NativeReference<int> outSum;
	
	public void Execute() {
		outSum.Value = valueA + valueB;
	}
}

var job = new SumJob { valueA = 10, valueB = 20, outSum = new NativeReference<int>(Allocator.TempJob) };
job.Run();
Debug.Log($"Sum: {job.outSum.Value}"); // Prints: `Sum: 30`
job.outSum.Dispose();
```

Now that the native collection is used in a job, we have to use the `TempJob` allocator, and we also have to remember to dispose it after the job is done!

If we want to sum an arbitrary list of numbers, we also can't pass in a normal C# Array, but have to use a NativeArray instead:

```c#
struct SumJob : IJob {
	public NativeArray<int> values;
	public NativeReference<int> outSum;
	
	public void Execute() {
		outSum.Value = 0;
		
		for (var i = 0; i < values.Length; i++) {
			outSum.Value += values[i];
		}
	}
}

var job = new SumJob {
	values = new NativeArray<int>(length: 3, Allocator.TempJob),
	outSum = new NativeReference<int>(Allocator.TempJob)
};
job.values[0] = 10;
job.values[1] = 20;
job.values[2] = 30;
job.Run();
Debug.Log($"Sum: {job.outSum.Value}"); // Prints: `Sum: 60`
job.values.Dispose();
job.outSum.Dispose();
```

### Job Requirements

Let’s take a look at how to convert a piece of code to be jobified. Let's say you have this code in an RTS game to find the nearest enemy to target, that is getting called quite a lot:

```c#
Unit nearestReachableEnemy(Unit forUnit, Unit[] units) {
	var minDistace = float.MaxValue;
	Unit closestUnit = null;
	
	foreach (var unit in units) {
		if (unit == forUnit) { continue; }
		if (!GameManager.instance.areUnitsEnemies(forUnit, unit)) { continue; }
		
		var distance = Vector3.Distance(position, unit.transform.position);
		if (distance < minDistance) {
			minDistance = distance;
			closestUnit = unit;
		}
	}
	
	return closestUnit;
}
```

This is a candidate for a method that can be turned into a job, and if we do, maybe we can even run it for all unit at the same time in parallel!

To get started with this, first there's a few requirements for code to be able to be jobified:

- The job must work on pure data. Touching GameObjects or MonoBehaviours is a big no-no.
- All collections must be native collections.
- You can't call out to global state, everything you need must be available to the job before running it.
- Jobs don't return data; it can only write into native collections.

In the example above, we violate... all of these:

- Unit is a MonoBehaviour, and we access its Transform.
- We pass in and loop over a C# Array.
- We call out to a singleton method from a global game manager.
- We return the result.

We're going to have to refactor the code to satisfy all of these points
