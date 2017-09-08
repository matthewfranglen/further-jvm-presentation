Overview
========

This will cover:
 * Memory Allocation, References and Garbage Collection
 * The analysis of running java programs for things like memory leaks or performance

[these are really related to concurrency, however they cover the java memory model well.
 I should use them for the concurrency talk]
https://shipilev.net/blog/2014/jmm-pragmatics/
https://shipilev.net/blog/2016/close-encounters-of-jmm-kind/

Memory Allocation
-----------------

Why do we need memory management?
What is memory management?
How does it work?

### Manual Memory Management

In C you have to manage your memory manually using malloc() and free().
This is extremely hard as:
 * Any memory allocated has to be freed (otherwise you have a memory leak)
 * Calling free on a pointer twice is an error
 * Using a freed pointer is an error
 * Using a pointer that has never been initialized is an error
These arn't small errors either.
Writing to an uninitialized pointer typically overwrites core system memory allocations.

Allocation of memory in this way can be very slow.
It's slow because it's a system call(?) to the OS memory manager.
So manual memory management can be overlaid on top of this.

This all sucks.
It's fast though.

### Pointer Bumping

Pointer bumping is one of the fastest ways to allocate memory. (I don't know of a faster one)
Allocation can take 10 machine instructions.

This involves having a block of memory and a pointer to the start of free memory within that block.
Allocation involves:

 * Move free memory pointer forward
 * Return original location as pointer to allocated memory

Java uses pointer bumping to allocate memory.

### Deallocating Memory

Pointer bumping requires that all free memory be contiguous.
If you free allocated memory then that leads to unusable gaps in the allocated memory space.

Need to collect the free memory by compacting the in use memory.

Compacting the memory requires changing the pointers that point to it.
This requires knowing all pointers to memory.

### Garbage Collection

What stops us from just allocating memory and never deallocating it?
We would need infinite memory.
Garbage collection is an attempt to simulate infinite memory.
