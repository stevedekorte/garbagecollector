# garbagecollector

## Overview

libgarbagecollector is an incremental tricolor tracing collector using a Baker Treadmill. libgarbagecollector is not an allocator; it only keeps track of the objects you tell it about and determines when it is safe to free them. So you'll allocate objects using malloc() (or the allocator of your choice), and register them with the collector. When the collector finds that an object can be freed, it will call a function you give it to do the actual deallocation.

## Getting Started

First you'll need to create a collector instance:

'''Collector *collector = Collector_new();
'''

Your values must contain the CollectorMarker struct as the first part of it's structure. So their struct declarations will look something like this:

'''struct MyObjectType
{
	CollectorMarker marker;
	...
};
'''

The collector manages the CollectorMarker and your code shouldn't touch it.

To tell the collector about your root value(s), call:

'''Collector_retain_(collector, aRootValue) 
'''

## Adding Values

When you allocate an object, you add the value to the collector.

'''Collector_addValue_(collector, aValue)
'''

## Marking

You provide the means for the collector to trace the reference graph via the mark callback function.

'''Collector_setMarkFunc_(collector, MyObjectType_mark);
'''

In your object's mark function, you'll need to call:

'''Collector_shouldMark_(collector, referencedValue);
'''

on each of the values it references.

You also need to tell the collector when a reference is added from one object to another (this is typically called the write barrier) and is required to support incremental collection.

'''Collector_value_addingRefTo_(collector, value, referencedValue);
'''

## Freeing

Every so many Collector_addValue_() calls, the collector will do a bit of marking, and every so many marks it will do a sweep. A sweep will result in the free callback being called for each value that was found to be unreachable.

You'll need to set the free callback to tell the collector which function to call when a value is found to be no longer reachable:

'''Collector_setFreeFunc_(collector, MyObjectType_free)
'''

## Atomic Operations

When you're doing an atomic operation, like initializing a new object, it's important to call:

'''Collector_pushPause(collector);
'''

To resume the collector, call:

'''Collector_popPause(collector);
'''

These increment and decrement a pause count and the collector will delay any marking until the pause count is zero.

## Stacks

Since the structure of the C stack is unknown, there is no way to trace it. By "stack" here I mean value stacks which hold the values being referenced by the C stack (or referenced by your language locals, if your language's stack frames aren't first class).

The simplest way to deal with stacks is to call:

'''Collector_value_addingRefTo_(collector, stackOwnerValue, newStackValue);
'''

whenever a new value is added to the stack. This will ensure that things referenced by the stack get marked. The next 2 sections describe how to ensure the stacks themselves get marked.

### Cooperative Multitasking Stacks

In the case of my programming language, Io, coroutines are used for concurrency and are first class objects in the language.

So if a coroutine is reachable via the root node, it will get marked and if not, no one will be able to tell it to resume, so it's safe to collect it. This works for all coroutines except the main coroutine (the one that started the program) and the current coroutine. So on startup I call:

'''Collector_retain_(collector, mainCoroutineValue);
'''

to ensure the main coroutine won't be collected. And everytime Io resumes a coroutine, I call:

'''Collector_setMarkBeforeSweepValue_(collector, currentCoroutineValue);
'''

Which will cause currentCoroutineValue to get marked immediately before the collector enters a sweep phase.

### Premptive Multitasking Stacks

If your language uses premtive threads which cannot be collected until they are explicitly exited, then you'll need to call:

'''Collector_retain_(collector, threadObjectValue);
'''

when a thread begins and:

'''Collector_stopRetaining_(collector, threadObjectValue);
'''

when it ends. The retained values are stored in an array, so this is bit inefficient if you're dealing with thousands of threads that are frequently created and destroyed, but for the typical case (10s of threads with greater than one second lifetimes) it should work well.

## Globals

Globals such as language primitives that should not be collected until shutdown can avoid being collected by calling:

'''Collector_retain_(collector, someGlobalValue);
'''

on each global.

## Options

The collection parameters can be adjusted with these methods:

'''Collector_setAllocsPerMark_(collector, apm);
Collector_setMarksPerSweep_(collector, mps);
Collector_setSweepsPerGeneration_(collector, spg);
'''

## Cleaning Up

To free all values beign monitored by the collector, call:

'''
Collector_freeAllValues(collector);
'''

To free your collector instance, call:

'''Collector_free(collector);
'''

## Notes

No attempt has been made to make this code safe for use when it's functions are called from multiple prememtive threads simultaniously.</td>
