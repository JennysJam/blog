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

  /* Pointer to a Heap object*/
  Heap* heap;
} VM;
```

We also update how we allocate objects to remove the code for the intrusive linked list:

```c
Object* newObject(VM* vm, ObjectType type) {
  if (vm->numObjects == vm->maxObjects) gc(vm);

  Object* object = heapAlloc(vm->heap, sizeof(Object));
  object->type = type;
  vm->numObjects++;

  return object;
}
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

Finally, we want to update our VM object to contain a Heap pointer, and to free it when cleaning up our VM.


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

Forwarding is pretty simple

```c
/// @brief Performs the copying/forwarding and if not forwarded, adds the item to worklist.
Object* forward(VM* vm, Object* object) {
  assert(inFromHeap(vm->heap, object), "Object must be in from-heap.");
  
  if (object->type == OBJ_FRWD) return object->forward;

  Object* copied = heapAlloc(vm->heap, sizeof(Object));
  vm->numObjects++;
  memcpy(copied, object, sizeof(Object));
  // thread copied object into worklist

  //set forwarding pointer
  object->type = OBJ_FRWD;
  object->forward = copied;
  vm->lastObject = copied + 1;
  return copied;
}
```