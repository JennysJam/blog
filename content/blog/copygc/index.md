+++
title = "Baby's second garbage collector"
description = "Introduction to copying collectors"
date = 2023-07-10
+++

# Baby's 2nd garbage collector. 

This blog post is meant to an iteration on the tutorial and introduction to garbage collection implementation presented in [this classic blog post](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/). We are still working with simple garbage collectors, but this time another one with a bit more complexity.

I thought this would be cool because the original code already has a tiny but functional mutator and a set of tests meant to stress it; this means we can compare both the performance and implementation of the copying collector as well as the normal collector.

## Source code for this article

The source code can be found at [This github repo](https://github.com/JennysJam/copygc).

It is also hosted locally on this site, and you can download it as well with:\

```bash
wget https://jennyjams.net/copygc/src.zip
```

## Copying collectors

A _copying collector_ is a garbage collector design that ties itself intimately in with the backing allocation system: you have two logical (and here, physical) memory regions. One of these is set to be _active_ region, and all new memory is allocated from this region. When the garbage collection subsystem determines that enough memory has been used that garbage needs to be collected, then the active region is swapped to inactive and vice-versa, then the garbage collector determines the set of live objects and _evacuates_ these objects from the now-inactive region to the newly active region. Then, we take any pointers in the now moved object, and check if the pointers they pointed to in the old region have already forwarded. If so, use the forwarded pointer, if not, perform evacuation.

If an object has already been moved, then the object in the inactive-region can be reused to contain a pointer pointing to it's evacuated counterpart; this helps preserve the topology of the object graph.

## Cheney's algorithm

The particular algorithm used here is [Cheney's Algorithm](https://dl.acm.org/doi/10.1145/362790.362798), and it comes to us from the venerable 1970s.

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

However, I feel like a lot of GC stuff can feel extremely abstract, especially when you start dealing with the quasi-recursive stuff, so I'm laying it out in pictures instead.

## Cheney's Algorithm, this time in pictures.


![](./img/copy-1.svg)

First, our initial state shows how one of the heaps is full of memory, and the collector has decided to begin garbage collecting. The set of boxes on the left is the Root Set (represented as a stack with 3 active members), and the two boxes on the right are the 2 heap regions.

![](./img/copy-2.svg)

We begin by walking through each of the roots. The blue color represents objects that have yet to be processed.

![](./img/copy-3.svg)

The first root is pointing to an un-forwarded object in the from-heap, so we forward it to the to-heap and update the from-heap object to point to it (represented by the purple and curved, dotted line).

![](./img/copy-4.svg)

Now that we've forward what the first root was pointing at, we set it to point at the evacuated object in the to-heap, and remove it from the set of objects we care about, rendering it again in brown.

![](./img/copy-5.svg)

Next, we handle the second root. Because it is pointing to an object that has already been forwarded because we handled it previously, we simply set the root to point at the evacuated object in the to-heap and remove the 2nd root from my concern.

![](./img/copy-6.svg)

Next, we handle the final root. This is pointing to an object that itself has a pointer, so first we evacuate it and set the from-heap object to forward to it. However, because we just copied it, the evacuated object is itself still pointing to a from-heap object.

![](./img/copy-7.svg)

Next, we set the final root to point toward the evacuated pointer. We are now finished with all of the root set, and next must begin handling the objects in the to-heap to move any objects that were not directly pointed to from the roots.

![](./img/copy-8.svg)

We handle the first object in the from-heap. Because it simply holds an integer, there is no need to do any more work and we move to the next object.

![](./img/copy-9.svg)

The second item in the to-heap is an object that contains a pointer pointing towards an object in the from heap. We must evacuate and forward that object, and add it to the set of object's we care about. 

![](./img/copy-10.svg)

Now that the second item in the to-heap is pointing to a forwarded pointer, we can fix it's pointer to point to the evacuated object in the to-heap, and thus remove it from our concerns.

![](./img/copy-11.svg)

Next, we handle the final object. Because it contains no pointers, we simply remove it from our concerns.

![](./img/copy-12.svg)

Finally, we have no more items to work through in either the roots or the heap. We can then clear the old heap and blow all previous objects.

Garbage collection is now Complete.

## The world smallest allocator

Unlike mark and sweep, a copying collector relies on it's allocator to free memory, and thus we cannot use the standard library `malloc()` and `free()` function. Thus, we will need an allocator!

The simplest possible allocator is a _bump_ allocator: just a large contiguous region of memory, and an offset. Whenever an `allocate(size)` call is perform, it returns a pointer to the memory that is `offset` bytes into the memory region. The offset is then incremented by this size.

This allocator doesn't really have a coherent `free()` operation -- even if you logically free memory earlier on in the allocator, the allocator itself can't reuse it. Thus, the only meaningful way to reset memory is to wipe _all_ allocations and reset the `offset` to 0.

This property makes this allocator-pattern genuinely useful for situations like webservers or videogames, where a request or individual frame might generate large amounts of data that is not meaningful useful to keep around afterwards; the `clear()` operation would be much faster than attempting to deallocate each of the objects.

![](./img/bump-1.svg)

This is a diagram showing a set of objects that will be populated by pointers to objects allocated by the bump allocator, and the bump allocator itself. The bump allocator consists of a offset (or pointer) into the backing allocation, along with a large chunk of memory itself.

![](./img/bump-2.svg)

Allocation is performed -- the object's pointer is set to point at the memory chunk plus the offset. Then the offset is moved down to point to the next free area of memory.

![](./img/bump-3.svg)

And again, with a different size.

![](./img/bump-4.svg)

And again. 

![](./img/bump-5.svg)

Next, we call `clear()` and reset the offset to point to the start of the memory chunk, effectively freeing the memory; on more security minded systems we might ``memset()`` the allocated areas to zero to prevent possibly leaking any information. All of the previous pointers into the memory are now dangling pointers -- even though they are pointing into valid memory that is mapped, the pointers to that memory are likely of the wrong type or value, and will be pointing into the middle of another object.

![](./img/bump-6.svg)

Next, we allocate memory again, and we begin allocating from the beginning of the memory chunk.

## Fixin' up our unified object model

Since we're updating an existing codebase to swap the GC strategy out, we need to perform some code to change things.

First, we update the object model for our VM to remove the intrusive link list pointer, and to add another case to our union for the forwarding pointer. This is one of the canonical benefits of copying over more traditional mark-sweep, mark-compact, and reference-counting schemes: we don't need any extra metadata inside of the header.

```c
typedef struct sObject {
  ObjectType type;
  // remove ``struct sObject* next;``
  union {
    /* OBJ_INT */
    int value;

    /* OBJ_PAIR */
    struct {
      struct sObject* head;
      struct sObject* tail;
    };
    struct sObject* forward;
  };
} Object;
```

Next, we update the VM model to store the code for the begining and end of the heap work-list

```c
typedef struct {
  Object* stack[STACK_MAX];
  int stackSize;

  /* First and last object in heap */
  Object* firstObject;
  Object* lastObject;

  /* The total number of currently allocated objects. */
  int numObjects;

  /* The number of objects required to trigger a GC. */
  int maxObjects;


} VM;
```

## The world's smallest allocator, twice

In our case, we actually need 2 bump-allocators. Because only one one is every being allocated from, we can keep around a single offset, and also keep an extra bit of state describing which pool is being used. The boring names would be Region 1 and 2, or Regions A and B; I want to be cute and name them Regions [Boba and Kiki instead](https://en.wikipedia.org/wiki/Bouba/kiki_effect) instead. Because I cannot spell and cannot be asked to double check things, all of the code uses Boba instead of Bouba. Consider this your daily reminder to go buy Boba.

First, we need a reasonable max size -- I simply choose 2 * 16:

```c
#define HEAP_MAX 65536
```

Next, the data structure. In following the language in Cheney's algorithm and other papers, we call this a _heap_.

```c
typedef struct {
  size_t bump_offset;
  Region region;
  unsigned char region_boba[HEAP_MAX];
  unsigned char region_kiki[HEAP_MAX];
} Heap;
```

First, we need some code to generate a new Heap object. We memset the memory to zero to ensure no previous data remains.

```c
// Allocates and creates new Heap
Heap* newHeap() {
  Heap* heap = malloc(sizeof(Heap));
  memset(heap->region_boba, 0, sizeof heap->region_boba);
  memset(heap->region_kiki, 0, sizeof heap->region_kiki);
  heap->region = RGN_BOBA;
  heap->bump_offset = 0;
  return heap;
}
```

It seem a little strange that, after writing a custom allocator, we would just ``malloc()`` to get the memory for the allocator, but this is pretty common -- unless you are yourself implementing `malloc()` or the lowest-level memory allocator for a system, normally you have to rely on _some_ function call to get you backing memory. And `malloc()` is conveniently right here!

Next, allocation!

```c
void* heapAlloc(Heap* heap, size_t size) {
  assert(heap->bump_offset + size < HEAP_MAX, 
      "Attempted to allocate more items that can be in heap");
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

We can't utilize this allocator unless we have a way of clearing out the dump allocator. Instead of the `clear()` interface discussed above, we swap the active heap over, and then reset the offset to 0. This also updates some data in the VM to allow for our later step of pointer fixup, so we pass a pointer to the VM in rather than to the Heap.

```c
void swapHeap(VM* vm) {
  if (vm->heap->region == RGN_BOBA) {
    vm->heap->region = RGN_KIKI;
    vm->firstObject = vm->lastObject = (Object*) vm->heap->region_kiki;
  }
  else {
    vm->heap->region = RGN_BOBA;
    vm->firstObject = vm->lastObject = (Object*) vm->heap->region_boba;
  }
  vm->heap->bump_offset = 0;
}
```

Also, since we're no longer using `malloc()` to allocate objects, we have to update our `newObject()` function.

We also update how we allocate objects to remove the code for the intrusive 

```c
Object* newObject(VM* vm, ObjectType type) {
  if (vm->numObjects == vm->maxObjects) gc(vm);

  Object* object = heapAlloc(vm->heap, sizeof(Object));
  object->type = type;
  vm->numObjects++;

  return object;
}
```

Finally, we want to update our VM object to contain a Heap pointer. We also add the `lastObject` pointer field that we use during the collection step.

```c
VM* newVM() {
  VM* vm = malloc(sizeof(VM));
  vm->stackSize = 0;
  vm->firstObject = NULL;
  vm->lastObject = NULL;
  vm->numObjects = 0;
  vm->maxObjects = INIT_OBJ_NUM_MAX;
  vm->heap = newHeap();
  return vm;
}
```

Finally, as good responsible C citizens we free the heap object we've malloced when we free it's own VM. 

```c
void freeVM(VM *vm) {
  vm->stackSize = 0;
  gc(vm);
  free(vm->heap);
  free(vm);
}
```

## No more marking, only forwarding.

Since we neither engage in marking nor sweeping, we can rip out almost all code related to mark and sweep code (which would no longer compile correctly anyone because we updated our VM and Object types).

We need to first handle all of our roots. To do this, we simply walk over each of the root, and perform evacuation and forwarding on each of the pointed to objects:

```c
void processRoots(VM* vm) {
  for (int i = 0; i < vm->stackSize; i++) {
    vm->stack[i] = forward(vm, vm->stack[i]);
  }
}
```

When we forward an object, we first check to see if it has already been forwarded.

If it has, we just return the forwarded pointer, which we know is in the to-heap.

If

```c
Object* forward(VM* vm, Object* object) {
  assert(inFromHeap(vm->heap, object), "Object must be in from-heap.");
  
  if (object->type == OBJ_FRWD) return object->forward;
  //...
}
```

If it hasn't, then we allocate a new object in the to-heap, copy the contents of object in the from-heap over.
We then update the `vm->lastObject` field to point one past the currently allocated object. This is one of the times in this codebase where i'm taking advantage of the fact all objects have the same overall size; if they were of a different size it would look more like `vm->lastObject  = (Object*) ((char*)copied + getObjectSize(copied)`.

```c
Object* forward(VM* vm, Object* object) {
  assert(inFromHeap(vm->heap, object), "Object must be in from-heap.");
  
  if (object->type == OBJ_FRWD) return object->forward;

  Object* copied = heapAlloc(vm->heap, sizeof(Object));
  vm->numObjects++;
  memcpy(copied, object, sizeof(Object));

  vm->lastObject = copied + 1;
  return copied;
}
```

Finally, we set our old object to hold a forwarding pointer:

```c
Object* forward(VM* vm, Object* object) {
  assert(inFromHeap(vm->heap, object), "Object must be in from-heap.");
  
  if (object->type == OBJ_FRWD) return object->forward;

  Object* copied = heapAlloc(vm->heap, sizeof(Object));
  vm->numObjects++;
  memcpy(copied, object, sizeof(Object));

  vm->lastObject = copied + 1;
  
  //set forwarding pointer
  object->type = OBJ_FRWD;
  object->forward = copied;
 
  return copied;
}
```

## To all of the indirectly traversable objects that I've left behind

Now that we've built the mechanism to forward our pointers, all managed objects pointed to by the roots have been handled. Next, we have to handle the objects that those moved objects still point to.

In the original mark-sweep scheme, reachability had been solved before any memory management had occured -- we recursively walk the object graph, bail if an object has already been marked to prevent loops, and then walk the linked list to purge all of the unmarked objects.

Here, the reachability and marking works intermingled: we iterate through each of the objects in the new heap, and when we see it has a field pointing into the from-heap, we moved it and stick it onto the end of our heap, meaning, and once we get to scanning it we fix up any of it's pointers, until we hit the end.
cls
Taking advantage that all objects in our VM are the same size, we can just increment a pointer. If we didn't, we'd need to do work to increment via actual current size.

```c
void processWorklist(VM* vm) {
  while (vm->firstObject != vm->lastObject) {
    //forward sub-pointers of Pair object
    if (vm->firstObject->type == OBJ_PAIR) {
      vm->firstObject->head = forward(vm, vm->firstObject->head);
      
      vm->firstObject->tail = forward(vm, vm->firstObject->tail);
    }

    vm->firstObject++;
  }
}
```


## Tying the garbage bag together

Finally, we change the `gc()` function so that we can.

```c
void gc(VM* vm) {
  int numObjects = vm->numObjects;
  vm->numObjects = 0;

  swapHeap(vm);
  processRoots(vm);
  processWorklist(vm);
  vm->lastObject = NULL;
  vm->firstObject = NULL;

  vm->maxObjects = vm->numObjects == 0 ? INIT_OBJ_NUM_MAX : vm->numObjects * 2;

  printf("Collected %d objects, %d remaining.\n", numObjects - vm->numObjects,
         vm->numObjects);
}
```


## Pros and Cons of a copy-collector

### Pro: Removed object list overhead.

Directly, we've removed the overhead of an additional pointer that the VM only uses during garbage collection, which does save us 8 bytes per object, and that sort of cost per-item can turn out to be quite large.

But in addition to that, we've also removed a temporal overhead in addition to a spatial overhead -- we no longer traversing a linked list, and that can be a huge change due to [Cache coherency](https://en.wikipedia.org/wiki/Cache_coherence). Because we're only ever stepping over an array, we won't clear the cache out, and we (potentially) might have no more issues with blowing out our memory cache and repopulating it.

We also have a _much_ simpler allocator than your systems `malloc()`/`free()`, but we do so at the cost of that systems flexibility, as giving up any effects of hardening against possible exploitation or preventing data races. That isn't a big deal with our tiny mutator in a system that'll never go into production, but is something worth considering for more serious projects.

### Pro: Active work depends on the number of live objects

A mark and sweep collector needs to traverse the entire object graphs while cleaning up objects. In contrast, the copy-collector only needs to work for each of the roots, and each of the objects that has been moved into the to-heap. If we have a very large number of dead objects and a very small number of live objects, this can be a massive gain on the mark and sweep implementation.

### Con: Halves the effective free space

All of this talk about the utility and benifets we get from cache coherency is nice and all, but we're also overlooking a bit of a big one:

We are only ever using half of our allocation space. The rest of it is always just sitting there, without anything to do, just dead.

### Con: We can't do extra work per deleted object?

Also a bit of a weird one, but one of the biggest benefits of bump allocators is also a weakness -- we blow away all of our unused memory with one simple command. This is perfect for the small mutator we work with, but if we were using C++ and those objects had non-trivial destructors, or we were implementing a language with [finalizers](https://en.wikipedia.org/wiki/Finalizer), then we lose access to all of those objects that might need to have additional code run, and we'd need to add some sort of mechanism to find and walk through all of the objects that need to be destroyed.

## Go forth and copy

