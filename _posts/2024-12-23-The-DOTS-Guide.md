---
layout: page
title: "The Unity DOTS Guide"
permalink: /unity-dots-guide
exclude: true
---

# The DOTS guide

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

The Native Collections package comes with replacements for the collection classes built in to C#. Array, List and Dictionary’s equivalent native collections are NativeArray, NativeList and NativeHashMap. These works 

## The Job System

The Job System is a package that allows you to create jobs, which are pieces of code that can’t access global state, but are instead confined to using only the input and output you’ve specified. This constraint allows the job to be run in the background, off of the main thread where all your usual logic happens, unlocking the multicore potential of the machine.
Normally if you’d want to run code in the background, you’d have to create a new thread, and then manually keep track of what is being read and written on that and all other threads, to avoid data races. For jobified code, this is not a concern. The system will take care of all those problems for you. If you schedule two jobs to write to the same data, the system will produce an error, and tell you to schedule them to run sequentially.

### IJob

Let’s take a look at how to convert a piece of code to be jobified. 