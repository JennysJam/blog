+++
title = "Baby's second garbage collector"
description = "Introduction to copying collectors"
date = 2023-07-10
+++

# Baby's 2nd garbage collector. 

This blog post is meant to an iteration on the tutorial and introduction to garbage collection in [this blog post](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/). It takes most of the same structure and code, but replaces the Mark and Sweep collector with what is variably called a _Semispace_ or _Copying_ collector, along with the necessary allocator and infrastructure.

I thought this would be cool because the original code already has a functional if tiny mutator in parts of a Lisp implementation, so you can see what differences slotting in a new collector have (and what other features we can take advantage of).

## Overview of copy collectoin

The semispace garbage collector is tied in closely with it's _memory allocator_, the system that handles giving out memory. The copying collector has two regions of memory, and only allocates from the currently active region.

When the garbage collector detects that the system is close to running out of memory, it swaps the inactive heap to be active, and then walks the roots looking for reachable objects. When an object is determined to be reachable, memory from the now-active region is allocated for the object, and the now-old object is then set to have a _forwarding pointer_ pointing to the new object. This is so that then it's trivial to update anything that was pointing to the old object to point to it's new location.

## Cheney's Algorithm.


## The worlds tiniest allocator, twice over.

In order to use the semispace allocator, we can't rely on the normal memory allocator given my `malloc()` and friends -- instead, we have to implement one ourself. Production implementations of garbage collectors that utilize custom allocators are much more complicated, but for this we are going to use the simplest allocator: the _bump allocator_. A good write up of some of the mechanics and pro/cons can be found [here](https://www.gingerbill.org/article/2019/02/08/memory-allocation-strategies-002/).

Of course, because we have two heaps, this means we have to handle which region we are allocating from. In the spirit of making things just a bit cuter, our region names are [Boba (region 1) and Kiki (region 2)](https://en.wikipedia.org/wiki/Bouba/kiki_effect).

First, let's define a reasonable size for our heap -- I chose 2^16 for this, but any relatively large size can work.

```c
#define HEAP_MAX 65536
```

Next, let's implement a value that holds both regions. (a more flexible codebase make the regions pointers to heap allocated regions).

```c
/* used for the two regions of the collector.*/
typedef enum {
  RGN_BOBA,
  RGN_KIKI
} Region;

typedef struct {
  size_t bump_offset;
  Region region;
  unsigned char region_boba[HEAP_MAX];
  unsigned char region_kiki[HEAP_MAX];
} Heap;
```

Finally, let's add the function that will handle the actual allocation:

```c
void* heap_alloc(Heap* heap, size_t size) {
  assert(heap->bump_offset + size < HEAP_MAX, "Attempted to allocate more items that can be in heap");
  void* pointer = NULL;

  if (heap->region == RGN_BOBA) {
    pointer = heap->region_boba + heap->bump_offset;
  }
  else {
    pointer = heap->region_kiki + heap->bump_offset;
  }
  heap->bump_offset += size;

  return pointer;
}
```

Because we're relying on the copy collection to clean up all of our old objects, we don't implement any sort of `heap_free()`.