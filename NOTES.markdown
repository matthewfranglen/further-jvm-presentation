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

### References

These allow you to implement caches and take sophisticated actions for cleaning up reclaimable objects.
The finalize method isn't reliable enough, as it can only be called once.

https://docs.oracle.com/javase/8/docs/api/java/lang/ref/package-summary.html

 * An object is strongly reachable if it can be reached by some thread without traversing any reference objects. A newly-created object is strongly reachable by the thread that created it.
 * An object is softly reachable if it is not strongly reachable but can be reached by traversing a soft reference.
 * An object is weakly reachable if it is neither strongly nor softly reachable but can be reached by traversing a weak reference. When the weak references to a weakly-reachable object are cleared, the object becomes eligible for finalization.
 * An object is phantom reachable if it is neither strongly, softly, nor weakly reachable, it has been finalized, and some phantom reference refers to it.
 * Finally, an object is unreachable, and therefore eligible for reclamation, when it is not reachable in any of the above ways. 

Strong Reference - what you are used to using.
Things pointed to by strong references are strongly reachable.

Soft Reference - A reference to an object which does not prevent the object from becoming softly reachable.
When cleared the object can become weakly reachable or worse.

Weak Reference - A reference to an object which does not prevent the object from becoming weakly reachable.
When cleared the object can be finalized.

Phantom Reference - A reference to a potentially *reclaimed* object.

### Garbage Collection

What stops us from just allocating memory and never deallocating it?
We would need infinite memory.
Garbage collection is an attempt to simulate infinite memory.

The garbage collector needs to reclaim memory on demand.
The garbage collector can collect things that are not strongly reachable.

http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/gc01/index.html

Really nice introduction by oracle/sun.

#### What is Automatic Garbage Collection?

Automatic garbage collection is the process of looking at heap memory, identifying which objects are in use and which are not, and deleting the unused objects. An in use object, or a referenced object, means that some part of your program still maintains a pointer to that object. An unused object, or unreferenced object, is no longer referenced by any part of your program. So the memory used by an unreferenced object can be reclaimed.

In a programming language like C, allocating and deallocating memory is a manual process. In Java, process of deallocating memory is handled automatically by the garbage collector. The basic process can be described as follows.

##### Step 1: Marking

The first step in the process is called marking. This is where the garbage collector identifies which pieces of memory are in use and which are not.

Referenced objects are shown in blue. Unreferenced objects are shown in gold. All objects are scanned in the marking phase to make this determination. This can be a very time consuming process if all objects in a system must be scanned.

##### Step 2: Normal Deletion

Normal deletion removes unreferenced objects leaving referenced objects and pointers to free space.

The memory allocator holds references to blocks of free space where new object can be allocated.

##### Step 2a: Deletion with Compacting

To further improve performance, in addition to deleting unreferenced objects, you can also compact the remaining referenced objects. By moving referenced object together, this makes new memory allocation much easier and faster.

##### Why Generational Garbage Collection?

As stated earlier, having to mark and compact all the objects in a JVM is inefficient. As more and more objects are allocated, the list of objects grows and grows leading to longer and longer garbage collection time. However, empirical analysis of applications has shown that most objects are short lived.

Here is an example of such data. The Y axis shows the number of bytes allocated and the X access shows the number of bytes allocated over time.

As you can see, fewer and fewer objects remain allocated over time. In fact most objects have a very short life as shown by the higher values on the left side of the graph.

##### JVM Generations

The information learned from the object allocation behavior can be used to enhance the performance of the JVM. Therefore, the heap is broken up into smaller parts or generations. The heap parts are: Young Generation, Old or Tenured Generation, and Permanent Generation

The Young Generation is where all new objects are allocated and aged. When the young generation fills up, this causes a minor garbage collection. Minor collections can be optimized assuming a high object mortality rate. A young generation full of dead objects is collected very quickly. Some surviving objects are aged and eventually move to the old generation.

Stop the World Event - All minor garbage collections are "Stop the World" events. This means that all application threads are stopped until the operation completes. Minor garbage collections are always Stop the World events.

The Old Generation is used to store long surviving objects. Typically, a threshold is set for young generation object and when that age is met, the object gets moved to the old generation. Eventually the old generation needs to be collected. This event is called a major garbage collection.

Major garbage collection are also Stop the World events. Often a major collection is much slower because it involves all live objects. So for Responsive applications, major garbage collections should be minimized. Also note, that the length of the Stop the World event for a major garbage collection is affected by the kind of garbage collector that is used for the old generation space.

The Permanent generation contains metadata required by the JVM to describe the classes and methods used in the application. The permanent generation is populated by the JVM at runtime based on classes in use by the application. In addition, Java SE library classes and methods may be stored here.

Classes may get collected (unloaded) if the JVM finds they are no longer needed and space may be needed for other classes. The permanent generation is included in a full garbage collection.
