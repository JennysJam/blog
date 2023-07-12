+++
title = "Baby's second garbage collector"
description = "Introduction to copying collectors"
date = 2023-07-10
+++

# Baby's 2nd garbage collector. 

This blog post is meant to an iteration on the tutorial and introduction to garbage collection implementation presented in [this classic blog post](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/). We are still working with simple garbage collectors, but this time another one with a bit more complexity.

I thought this would be cool because the original code already has a tiny but functional mutator and a set of tests meant to stress it; this means we can compare both the performance and implementation of the copying collector as well as the normal collector.

## Copying collectors

A _copying collector_ is a garbage collector design that ties itself intimately in with the backing allocation system: you have two logical (and here, physical) memory regions. One of these is set to be _active_ region, and all new memory is allocated from this region. When the garbage collection subsystem determines that enough memory has been used that garbage needs to be collected, then the active region is swapped to inactive and vice-versa, then the garbage collector determines the set of live objects and _evacuates_ these objects from the now-inactive region to the newly active region. Then, any pointers inside these objects are checked to see if they point to the old region -- if they do, it is checked if an object has been already brought over, and if not, it itself copied over and analyzed.

If an object has already been moved, then the object in the inactive-region can be reused to contain a pointer pointing to it's evacuated counterpart; this helps preserve the topology of the object graph.

## The algorithm

1. Swap active heap
1. initialize a _worklist_ of objects that have been evacuated, but the objects they point to may not be.
2. For each root in the set of roots, perform evacuation and update the old object to point to the new object
3. Service each item in the root list until empty:  
  a. check if a field in the object points to the old heap.  
  b. if yes, evacuate the pointed-to-object, and add it to the worklist

Or, in psuedo code terms:

```py

def collect_garbage():
  swap_heap()
  for root in roots:
    root.pointer = evacuate(root.pointer)
  
  while not worklist.is_empty():
    obj = worklist.pop()
    for field in obj.fields():
      if field.pointer is in old_region:
        field.pointer = evacuate(root.pointer)

def evacuate(ptr):
  new_object = allocate_from_active()
  copy(new_object, ptr)
  worklist.append(new_object)

```

## Show me the code

The code for this listing originally came from [](). For convenience, i'm hosting the update version on my blog here:

- [main.c](main.c)
- [COPYRIGHT](COPYRIGHT)
- [Makefile](Makefile)

## Our Implementation

