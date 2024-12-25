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
- The Mathematics package as an alternative to the built in `VectorX` and `Mathf`, which can produce much more optimized math code when Burst-compiled.
- The Entities package is a new way to do ECS as an alternative to GameObjects and MonoBehaviours. It’s doing away with the object oriented thinking, replacing it with a systems based approach, allowing many times more entities to exist.

These packages together creates a whole new way of structuring your projects, and unlocks performance that was not feasible before.
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
using Unity.Jobs;

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

If you have a C# Array that you want to pass in to a job, you can create a `NativeArray` and use its `CopyFrom` method to copy the contents over.

### Burst-compiling jobs

This is the most fun part of the whole guide. If you try to run a job you've created, and inspect it in the profiler, you'll see that it doesn't really run any faster than the non-jobified version. This is because the job is still running normal Mono code. To get the speedup, we need to tell the Burst compiler to optimize it. This is done very simply by putting a marker on the job struct:

```c#
using Unity.Burst;

[BurstCompile]
struct SumJob : IJob {
	...
}
```

That's all you need to do. When you run it now, it should appear in the profiler in a green color instead of the normal blue color, that is, if you can even find it. Burst-compiling jobs can make them so fast that they basically complete in no time, and turn into a tiny sliver of a bar in the profiler.
That's pretty much all there is to it. The only rinkle is that Burst-compiled jobs are harder to debug than non-Bursted jobs. Exception support is limited, stack traces are harder to read, string formatting is more limited, etc. So when you want to debug a job, it makes sense to temporarily turn off Burst. Burst can be turned off completely from the Jobs menu (Jobs > Enable Burst Compilation).

### Job scheduling & dependencies

You might have two jobs that somehow share data, like a job converting a list of data, and then another job sorting it. Running those two jobs in the background at the same time will create a conflict, where they both try to access the same data at the same time, which is not allowed, and the Job System will produce an error for you. Of course you can just run them sequencially on the main thread, but the whole point of creating jobs is to be able to run them off the main thread. You could schedule the first job, wait for it to complete, and then schedule the second job, but this will create a "sync point" in the middle, which will block the main thread anyway. You really want to schedule as much work as possible up front, to run everything without any sync points. To do that, we can schedule jobs with dependencies on other jobs. When a job is `.Schedule`d, you get a `JobHandle` back. This handle can be passed in when scheduling another job to create a dependency on the first job. This will make the second job run automatically when the first job is done.

Say you want to calculate a list of distances to all nearby waypoints, sorted by that distance. You have a `CalculateDistancesJob` job, which calculate the distance to each waypoint, and a `SortWaypointDistancesJob`, which sorts a list of `WaypointDistance`s. You can schedule these two jobs with a dependency, to make them run in the correct order:

```c#
var calculateDistancesJob = new CalculateDistancesJob(origin, waypoints, waypointDistances);
var calculateDistancesJobHandle = calculateDistancesJob.Schedule();

var sortJob = new SortWaypointDistancesJob(waypointDistances);
var sortJobHandle = sortJob.Schedule(dependsOn: calculateDistancesJobHandle);

sortJobHandle.Complete();
```

This way, they will execute in the correct order, and you won't get any errors about them trying to access the same data at the same time, even though both jobs read and write to the `waypointDistances` array.

One thing you might also want to do is schedule a job with multiple dependencies. Say you want to both calculate the distance, but also wether the waypoint is accessible or not, and have the sort job only consider accessible waypoints. You can only pass one `JobHandle` to the `Schedule` method, so how would we do this? `JobHandle` has the static method called `CombineDependencies`, which can take in two or more dependencies, and combine them into a single `JobHandle`. This can then be used to schedule the sort job:

```c#
var calculateDistancesJob = new CalculateDistancesJob(origin, waypoints, waypointDistances);
var calculateDistancesJobHandle = calculateDistancesJob.Schedule();

var calculateAccessiblityJob = new CalculateAccessiblityJob(origin, waypoints, waypointsAccessibility);
var calculateAccessiblityJobHandle = calculateAccessiblityJob.Schedule();

var combinedDependency = JobHandle.CombineDependencies(calculateDistancesJobHandle, calculateAccessiblityJobHandle);

var sortJob = new SortWaypointDistancesJob(waypointDistances, waypointsAccessibility);
var sortJobHandle = sortJob.Schedule(dependsOn: combinedDependency);

sortJobHandle.Complete();
```

This way, the `CalculateDistancesJob` and `CalculateAccessiblityJob` will run at the same time in parallel on different threads, and when they're both done, the `SortWaypointDistancesJob` job will run, using the results of the two jobs! Neat!
You might notice something in this piece of code though: Both `CalculateDistancesJob` and `CalculateAccessiblityJob` uses the `waypoints` array without any dependencies! They're going to conflict, and it's not going to work! Well, actually not, because they're both only reading from waypoints, and that can perfectly be done in parallel on different threads without problem. It's only when you start modifying data that you run into data races.
To let the Job System know that you're only going to read from some data, you can put a `[ReadOnly]` marker on the field in the job struct:

```c#
struct CalculateDistancesJob : IJob {
	[ReadOnly] public NativeArray<Vector3> waypoints;
	
	...
}
```

This will let the job scheduler know that you only want to read from the array, and will be able schedule it in parallel with other jobs just reading the same data.
Note that it doesn't prevent you from writing to the array; however doing so will produce a nasty runtime error.
There's also a `[WriteOnly]` marker, for if you are only ever writing to an array, such as having an output array.

### Avoiding main thread blocking

Even though the wayponints code before runs in the background, we still end up waiting for the results on the main thread by calling `.Complete()` on the job handle. This might make sense, if you just have a method you need to run, and you need the results immediately before you can do the next thing. In other cases, it might be a bit of a waste having the main thread sitting around idle, while a background job is running. Sometimes you might be able to do some other useful work on the main thread in the meantime, like if you need to do somework, then calculate something, then do something with that calculation, you can jobify the calcuation and schedule it in the beginning, do the initial work while the job is running, and then complete the job to get the result for the final action. That could look something like this:

```c#
void Update() {
	var waypointDistancesJob = new WaypointDistancesJob {
		waypoints = new NativeArray<Vector3>(waypoints.Length, Allocator.TempJob),
		outDistances = new NativeArray<float>(waypoints.Length, Allocator.TempJob)
	};
	waypointDistancesJob.waypoints.CopyFrom(waypoints);
	
	var waypointDistancesJobHandle = waypointDistancesJob.Schedule();
	
	// Do some other work here
	
	waypointDistancesJobHandle.Complete();
	destination = selectWaypoint(waypoints, waypointDistancesJob.outDistances);
	
	waypointDistancesJob.waypoints.Dispose();
	waypointDistancesJob.outDistances.Dispose();
}
```

In this setup, if the main thread work finishes and calls `Complete()` before the job is done, the main thread will block and wait for it to complete. If the job completes before the main thread has called `Complete()`, it will just free up that thread, and have the result ready.

If you have a very long running job, like doing pathfinding, mesh generation, or something like that, which doesn't require the result to be ready straight away, and can even run over multiple frames if needed, you can instead schedule the job, and then save the job and `JobHandle` in class fields. Then you can check in every frame, and instead of requesting the result immediately with `Complete()` use `IsCompleted` to check for completion:

```c#
public class PathfindingUnit : MonoBehaviour {
	PathfindingJob pathfindingJob;
	JobHandle pathfindingJobHandle;
	
	const int maxPathLength = 512;
	
	public void setDestination(Vector3 destination) {
		pathfindingJob = new PathfindingJob();
		pathfindingJob.destination = destination;
		pathfindingJob.pathLength = new NativeReference<int>(Allocator.TempJob);
		pathfindingJob.pathBuffer = new NativeArray<int>(length: maxPathLength, Allocator.TempJob)
		pathfindingJobHandle = pathfindingJob.Schedule();
	}
	
	void Update() {
		if (pathfindingJobHandle != default && pathfindingJobHandle.IsComplete) {
			path = pathfindingJob.pathBuffer.Slice(0, pathfindingJob.pathLength.Value).ToArray();
			beingPath(path);
			
			pathfindingJob.pathLength.Dispose();
			pathfindingJob.pathBuffer.Dispose();
			pathfindingJob = default;
			pathfindingJobHandle = default;
		}
	}
	
	void beginPath(Vector3[] path) {
		...
	}
}
```

### Jobifying existing code

Let’s take a look at how to convert a piece of code to be jobified. Let's say you have this code in an RTS game to find the nearest enemy to target, that is getting called quite a lot:

```c#
Unit nearestReachableEnemy(Unit forUnit, Unit[] units) {
	var minDistance = float.MaxValue;
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

void Update() {
	targetEnemy = nearestReachableEnemy(forUnit: this, GameManager.instance.allUnits);
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

We're going to have to refactor the code to satisfy all of these points. You can see the resulting job in the end, but let's go over the process.

The first thing we can do is to actually specify the job. The method is getting called from a lot of different places in the codebase, so we'd like to keep the method, but replace the implementation with a job. We'll just in the method, instead of calculating everything directly, create the job, fill it with data, run it, get the result out, do cleanup, and then return the result. We're not going to get any multithreading out of this; if we wanted that, we need to do some bigger refactor of the code, possibly to allow all units in the game to run their method at the same time (this is going to be much easier when we get into Entities). Instead, we'll just the benefit of having the method Burst-compiled.

Instead of looping through a lot of Units, we have to replace this with a list of pure data, so we need to think about, how we can prepare all the data we're going to need before running the job. We need two things for each unit; its transform position, and wether or not it's an enemy, which is queried from some global game manager. For the positions, we can prepare a list of positions in a `NativeArray` ahead of time, and pass that in to our job.
For the enemy check, it's going to depend on the implementation of the method. If the method is simple enough (like `unit.isEnemy != otherUnit.isEnemy`), a job-friedly `static` version of it can be exposed, that takes in pure-data input, which can be called from the job (you can call methods from jobs no problem, you just can't pass in the reference to the game handler, since it's a MonoBehaviour). However let's just say that it's a complicated method that requires access to a lot of global state. In this case, we'll have to pre-calculate it for each unit before starting the job. We'll create another `NativeArray`, this time with a bool for each unit, to signal if they're an enemy or not, and we'll fill the array before starting the job.
The last issue is that jobs can't return values. What we can do instead is to return the value via a write-only, one-length array, where we write the result to the first slot. Or, instead of a one-length array, use `NativeReference` to store the resulting value. We'll mark it with `[WriteOnly]`, just to signal it to the Job System, in case it can do some optimizations, and I like to prefix the name with "out", so that it's clear that it's meant to be a return value.

```c#
// The implementation of the method, in job form
struct NearestEnemyJob : IJob {
	// This doesn't need [ReadOnly], since it's not a collection type, and it's going to be copied no matter what.
	public int unitIndex;
	
	// Our prepared input, marked as read-only
	[ReadOnly] public NativeArray<Vector3> unitPositions;
	[ReadOnly] public NativeArray<bool> isUnitEnemy;
	
	// Our output, marked as WriteOnly
	[WriteOnly] public NativeReference<int> outEnemyIndex;
	
	public void Execute() {
		var unitPosition = unitPositions[unitIndex];
		
		var minDistance = float.MaxValue;
		int? closestUnitIndex = null;
		
		for (var i = 0; i < unitPositions.Length; i++) {
			if (i == unitIndex) { continue; }
			if (!isUnitEnemy[i]) { continue; }
			
			var distance = Vector3.Distance(unitPosition, unitPositions[i]);
			if (distance < minDistance) {
				minDistance = distance;
				closestUnitIndex = i;
			}
		}
		
		outEnemyIndex = closestUnitIndex ?? -1;
	}
}

// Same method as before, but the guts has been switched out with a job-invocation
Unit nearestReachableEnemy(Unit forUnit, Unit[] units) {
	// Creating the job and allocating collections
	var job = new NearestEnemyJob {
		unitIndex = units.IndexOf(forUnit),
		unitPositions = new NativeArray<Vector3>(units.Length, Allocator.TempJob),
		isUnitEnemy = new NativeArray<bool>(units.Length, Allocator.TempJob),
		outEnemyIndex = new NativeReference<int>(Allocator.TempJob)
	};
	
	// Copying all the data we need, that we can't access in a job
	for (var i = 0; i < units.Length; i++) {
		job.unitPositions[i] = units[i].transform.position;
		job.isUnitEnemy = GameManager.instance.areUnitsEnemies(forUnit, units[i]);
	}
	
	// Running the job
	job.Run();
	
	// Getting out the result
	var result = (job.outEnemyIndex < 0 ? null : units[job.outEnemyIndex]);
	
	// Disposing all the collections we allocated earlier
	job.unitPositions.Dispose();
	job.isUnitEnemy.Dispose();
	job.outEnemyIndex.Dispose();
	
	return result;
}

// This is completely unchanged
void Update() {
	targetEnemy = nearestReachableEnemy(forUnit: this, GameManager.instance.allUnits);
}
```

If you think that this is a lot more code than before, well, yea, it kind of is. Using the Job System can be a bit verbose, and requires a lot of allocation, copying and cleanup. When you see the result of getting code Burst compiled, and see it reduced from multiple milliseconds to almost nothing, it's going to feel worth it. Another side-benefit as opposed to traditional code is that jobs are completely self-contained. There's no way that deep within a method, it suddenly calls out to methods that start messing with things it shouldn't, because that literally not going to compile.
Not all code is well-suited to be jobified. What is traditionally called "glue" code, meaning code that doesn't do much calculation, but just bind different system together, like AI decision-making code, that needs access to a bunch of different systems, is going to be very difficult to jobify, since you'll have to provide all the data it's going to need up-front, and it's not going to have a lot of benefit, since it's probably not what's tanking your framerate. Use it instead for long-running methods, that you can see in the profiler is taking up a big chunk of your frametime. 

### Parallel job execution

So far we've only looked at jobs that run a single method in the background, and to get any parallelization, you'd have to schedule multiple jobs at the same time. Well, there is actually a way for a single job to be able to run on all the worker threads available with very little change! For this, we have to upgrade from `IJob` to `IJobParallelFor`. This is like a normal `IJob`, except that it's going to run for every index in an array, like a for-loop. When you schedule it, you specify the length of the loop, and the `Execute` method will now have an `int index` parameter, which tells you the index into the array that is being executed:

```c#
struct CalculatePairSumsJob : IJobParallelFor {
	[ReadOnly] NativeArray<int> valuesA;
	[ReadOnly] NativeArray<int> valuesB;
	[WriteOnly] NativeArray<int> sums;
	
	public void Execute(int index) {
		sums[index] = valuesA[index] + valuesB[index];
	}
}

int[] calculateSums(int[] valuesA, int[] valuesB) {
	Debug.Assert(valuesA.Length == valuesB.Length);
	
	var job = new CalculatePairSumsJob();
	
	CalculatePairSumsJob.valuesA = new NativeArray<int>(length: valuesA.Length, Allocator.TempJob);
	CalculatePairSumsJob.valuesA.CopyFrom(valuesA);
	
	CalculatePairSumsJob.valuesB = new NativeArray<int>(length: valuesB.Length, Allocator.TempJob);
	CalculatePairSumsJob.valuesB.CopyFrom(valuesB);
	
	CalculatePairSumsJob.sums = new NativeArray<int>(length: valuesA.Length, Allocator.TempJob);
	
	job.Schedule(valuesA.Length, innerLoopBatchCount: 32).Complete();
	
	var sums = CalculatePairSumsJob.sums.ToArray();
	
	CalculatePairSumsJob.valuesA.Dispose();
	CalculatePairSumsJob.valuesB.Dispose();
	CalculatePairSumsJob.sums.Dispose();
	
	return sums;
}
```

The only differences from a regular `IJob` would be that usually you'd just do a for-loop inside of the job, but now the Execute method only contains what would be inside of the loop body. Also when you schedule the job, you have to specify the amount of times it should loop, along with an inner loop batch count. It's going to take the entire array, and split it up into batches, with the size of each batch defined by that count, and run each of them independently. If you schedule a job with a long enough array, you should be able to see it taking up all the job workers threads! If you have a job that's looping over data, turning it into a `IJobParallel` For is a pretty easy way to parallelize it.
You might notice a possible race condition here though: what if you access the array outside of the index given? What happens if you write to sum 10 when invoked as index 1? Well, the Job System is going to prevent you. By default, you can only access array indices the same as the index you get passed in. If you really need access outside of that, and you're sure you know what you're doing, you can put `[NativeDisableParallelForRestriction]` on the array declaration to avoid this error.

### Job System conclusion

This should be everything you need to know to start using the Job System! By converting long-running calculations into Burst-optimized jobs, you can achieve great speedups. Of course, this requires that you have your calculations fairly separate from your glue-code, that requires access to global state, and if you really want the full potential of job realized, you might have to restructure your code away from the typical MonoBehaviour structure. Like, if you want all units to run their pathfinding code in parallel, either you need to schedule the jobs to have them ready for the next frame, or you need to have a planner object, that calculates the paths for all units at the same time during the frame. When we get to Entities, you'll see that this kind of functionality is exactly what it unlocks. For now, if jobs fulfill your performance needs, feel free to stop here, and just use jobs in a GameObject environment! It can be perfectly fine! But if you need a lot of objects in your game, or if you're just curious, then read on!

## Entities


