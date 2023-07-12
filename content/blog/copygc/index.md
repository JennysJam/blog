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

## Cheney's algorithm

1. Swap the active heap
2. Walk the set of roots, and evacuate the pointer to the new heap.
3. After evacuating all of the objects, start walking through all of the newly allocated objects and inspecting each of their fields. If the field is a pointer and points into the old region, check if it has a forwarding address. If it does, set the field to the forwarding address. If not evacuate it.

Or, in psuedo code terms:

```py

def collect_garbage():
  swap_heap()
  for root in roots:
    if field in old_region:
      root.pointer = evacuate(root.pointer)
  
  while not worklist.is_empty():
    obj = worklist.pop()
    for field in obj.fields():
      if field.pointer is in old_region:
        field.pointer = evacuate(root.pointer)

def evacuate(ptr):
  if has_forward_address(ptr):
    return ptr.forward
  new_object = allocate_from_active()
  copy(new_object, ptr)
  ptr.forward = new_object
  worklist.append(new_object)

```

## Show me the code!

The code for this listing originally came from. For convenience, i'm hosting the update version on my blog here:

- [main.c](main.c)
- [COPYRIGHT](COPYRIGHT)
- [Makefile](Makefile)


## The world's smallest allocator, twice

Unlike mark and sweep, a copying collector relies on it's allocator to free memory, and thus we cannot use the standard library `malloc()` and `free()` function. Thus, we will need an allocator!

The simplest possible allocator is a _bump_ allocator: just a large contiguous region of memory, and an offset. Whenever an `allocate(size)` call is perform, it returns a pointer to the memory that is `offset` bytes into the memory region. The offset is then incremented by this size.

This allocator doesn't really have a coherent `free()` operation -- even if you logically free memory earlier on in the allocator, the allocator itself can't reuse it. Thus, the only meaningful way to reset memory is to wipe _all_ allocations and reset the `offset` to 0.

This property makes this allocator-pattern genuinely useful for situations like webservers or videogames, where a request or individual frame might generate large amounts of data that is not meaningful useful to keep around afterwards; the `clear()` operation would be much faster than attempting to deallocate each of the objects.

In our case, we actually need 2 bump-allocators. Because one is every being allocated from, we can keep around a single offset, and also keep an extra bit of state describing which pool is being used. The boring names would be Region 1 and 2, or Regions A and B; I want to be cute and name them Regions Kiki and Regions Boba instead.

First, we need a reasonable max size -- I simply choose 2 * 16:

```c
#define HEAP_MAX 65536
```

Next, the data structure. In following the langauge in Cheney's algorithm and other papers, we call this a _heap_.

```c
typedef struct {
  size_t bump_offset;
  Region region;
  unsigned char region_boba[HEAP_MAX];
  unsigned char region_kiki[HEAP_MAX];
} Heap;
```

Next, allocation!