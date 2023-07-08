+++
title = "Baby's second garbage collector"
description = ""
+++

# Baby's 2nd garbage collector. 

This blog post is meant to an iteration on the tutorial and introduction to garbage collection in [this blog post](https://journal.stuffwithstuff.com/2013/12/08/babys-first-garbage-collector/). It takes most of the same structure and code, but replaces the Mark and Sweep collector with what is variably called a _Semispace_ or _Copying_ collector, along with the necessary allocator and infrastructure.

I thought this would be cool because the original code already has a functional if tiny mutator in parts of a Lisp implementation, so you can see what differences slotting in a new collector have (and what other features we can take advantage of).

## The semispace algorithm

The semispace garbage collection algorithim requires there two be two spaces (or heaps) that memory is allocated. 

1. When GC begins, find all of the root.

2. From the roots, find all reachable objects from those roots via checking if an object is _marked_ (usually by changing a boolean in some common object header field), and if so marking it and then recursively marking each of that objects fields.

3. Once

## The worlds tiniest allocator

## Copying the objects

## Patching the roots

## Issues, designs, and other features