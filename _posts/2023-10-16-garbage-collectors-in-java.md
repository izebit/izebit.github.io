---
layout: post
tags:
  - java
title: Garbage Collectors in JVM-based languages
---

<img src="/assets/img/garbage-collectors/thumbnail.png" alt="garbage collectors in java"/>

In this topic I would like to talk about **garbage collectors** (GCs). The aim that I target is to give a brief overview of 
main concepts of Garbage Collectors and list available GC implementations.

<h2>Contents</h2>
1. [What is Garbage Collector?](#definition)
2. [Serial Garbage Collector](#serial)
3. [Parallel Garbage Collector](#parallel)
4. [Concurrent Mark Sweep (CMS) Collector](#cms)
5. [G1 - Garbage First Collector](#g1)
6. [ZGC - The Z Garbage Collector](#zgc)
7. [ShenandoahGC](#ShenandoahGC)
8. [Azul C4 Garbage Collector](#c4)
9. [Epsilon Garbage Collector](#epsilon)

<h2 id="definition"> What is Garbage Collector?</h2>

> Garbage Collector is a form of automatic memory management  
> — <cite> Wikipedia

The component takes care about reclaiming memory that was allocated for objects needed earlier. There are programming languages that don't provide this functionality and a programmer has to manage memory manually.
**Java Virtual Machine** (JVM) includes garbage collector. It means that all JVM-based languages e.g. Java, Kotlin, Scala, Clojure supports automatic reclaiming unused objects.  
There are many algorithms that might be applied to check if an object is not needed and
memory for the object might be reused for other objects. The main idea of them is if there are no references 
to an object, the object is unreachable and can't be used. By this way, this type of objects might be deleted safely.

<img src="/assets/img/garbage-collectors/references-graph.png" alt="graph of references"/>

Garbage Collector traverses the reference graph from the several start points. They are called **GC Roots**. 
It might be static variables, classes loaded by system class loaders, active threads etc.

There are some common phases of the GC work that encounter in listed below garbage collectors:
1. Identify live objects that are reachable from the GC Roots
2. Reclaim memory held by dead objects
3. Relocate live objects or compact them

Let's talk about what type of garbage collectors you can use if you write code in JVM-based language.

<h2 id="serial">Serial Garbage Collector</h2>
It's one of the oldest garbage collector and exists from the first version of JVM. **Serial Garbage Collector** works simply. 
All running threads are suspended and all references are checked by one thread. It occurs *stop-the-world* pause.

To enable the GC, use the following command:
```bash
java \
-XX:+UseSerialGC # enable Serial Garbage Collector
```
<h2 id="parallel">Parallel Garbage Collector</h2>
Based on the name, you might guess how it works. The **Parallel Garbage Collector** works as 
**Serial Garbage Collector** works, but we can set number of threads. By this way, the total *stop-the-world pause* must be less if garbage collector works in multi-thread mode.

There are several JVM flags that works with Parallel Garbage Collector.
```bash
java \
-XX:+UseParallelGC        # enable Parallel Garbage Collector
-XX:MaxGCPauseMillis=<N>  # maximum time interval between GC
-XX:GCTimeRatio=<N>       #the ratio between GC time and time of execution application
-XX:ParallelGCThreads=<N> # number of threads 
```

<h2 id="cms">Concurrent Mark Sweep (CMS) Collector</h2>
The idea behind a work of the garbage collector is **The Weak Generational Hypothesis**. It says

> Objects will die young

i.e. the longer an object lives, the longer it will be reachable.

<img src="/assets/img/garbage-collectors/weak-hypothesis-of-generation.png" alt="The Weak Generational Hypothesis"/>

By this way, there is no need of scanning the whole set of objects. If memory is split into different categories and check them separately, it might be possible to reduce the *stop-the-world pause* by scanning  regions of memory at different rates.

CMS Collector works in pretty the same way.  All memory is split into 3 regions:
1. Permanent Generation
2. Young Generation
3. Old Generation  

**Permanent Generation** is a memory region where JVM locates static data e.g. loaded classes metadata, static methods, static variables.    
**Young Generation** is a segment where JVM allocates memory for a new objects. After several GC collections, if an object is still alive, it will be moved to **Old Generation**.  
**Old Generation** is memory region where "old" objects are located. At some point, object might be placed there directly if capacity of **Young Generation** doesn't let do it.

<img src="/assets/img/garbage-collectors/structure-of-heap.png" alt="Structure of the hotspot heap"/>

This garbage collector is a combination of two previous ones. **Serial Garbage Collector** and **Parallel Garbage Collector** get on the stage at different time.  
Initially, **Parallel Garbage Collector** pauses application threads twice. At *initial pause* 
it marks objects that are reachable from the *GC Roots* as live. Then **Parallel Garbage Collector** traces 
the available references concurrently and identifies reachable objects without suspending application threads. 
Since application is running during this step, checked references might be changed. That's why **Parallel Garbage Collector** needs the second *remark* pause. 
After it, all objects that aren't reachable might be removed and live objects are compacted. This operation happens during *Concurrent Sweep* phase.  
When **Old Generation** becomes full **Serial Collector** starts working. In this case, all application threads are stopped while collection is done.

There are several JVM flags that might come in handy to work with CMS Collector:
```bash
java \
-XX:+UseConcMarkSweepGC      # enable Concurrent Mark & Sweep Garbage Collector
-XX:ConcGCThreads=<N>        # total count of threads for GC
-XX:+CMSScavengeBeforeRemark # run remark stage after GC execution on new generation area
-XX:CMSWaitDuration          # run GC on tenured are after GC execution on new generation
-XX:ParallelGCThreads=<N>    # count of threads for parallel GC execution
-XX:+UseParNewGC             # use parallelism during GC on new generation area
```

#### Links:
- <a href="https://docs.oracle.com/en/java/javase/11/gctuning/concurrent-mark-sweep-cms-collector.html#GUID-FF8150AC-73D9-4780-91DD-148E63FA1BFF" rel="nofollow">HotSpot Virtual Machine Garbage Collection Tuning Guide</a>

<h2 id="g1">G1 - Garbage First Collector</h2>
Making latency predictable for application running on JVM is a challenging task. G1 Collector tries to keep *stop-the-world pause* in fixed bounders.   
In order to do it, all generations are split into equal-sized regions of memory. Based on calculated statistics, G1 can estimate how many regions it can check in set interval.

<img src="/assets/img/garbage-collectors/g1-heap-structure.png" alt="Structure of G1 heap"/>

At some cases, If an object requires larger and equal the size of a half region, this type of object gets allocated as sequence of several regions. This objects are called **Humongous Objects** .

G1 concentrates efforts on regions of **Young Generation** and checks regions of **Old Generation** occasionally. After **initial mark** the Garbage Collector pick the regions that will yield a large amount of free space. This approach lets the garbage collector be most effective. That's why it is called **Garbage First** - blocks with garbage will be handled first.  
After each **Young Generation** Collection, all remaining live objects are evacuated and compacted to an available region of **Young Generation** or if the threshold is met, objects are moved to a region of **Old Generation** accordingly.

The following flags might be used to tune G1 settings:

```bash
java \
-XX:+UseG1GC                       # enable Garbage First Collector
-XX:MaxGCPauseMillis=<N>           # Maximum interval between GC pauses
-XX:GCPauseIntervalMillis=<N>      # Maximum interval between G1 executions
-XX:G1HeapRegionSize=<size>[k|m|G] # heap memory size
-XX:ParallelGCThreads=<N>          # count of threads for parallel GC execution
-XX:ConcGCThreads=<N>              # total count of threads for GC
```

#### Links:
- <a href="https://docs.oracle.com/en/java/javase/11/gctuning/garbage-first-g1-garbage-collector1.html#GUID-6D6B18B1-063B-48FF-99E3-5AF059C43CE8" rel="nofollow">HotSpot Virtual Machine Garbage Collection Tuning Guide</a>
- <a href="https://www.oracle.com/technetwork/tutorials/tutorials-1876574.html" rel="nofollow">Getting Started with the G1 Garbage Collector</a>

<h2 id="zgc">ZGC - The Z Garbage Collector</h2>
This garbage collector targets to provide low latency, less 10ms and to work with multi-terabytes heap. It guarantees `O(1)` pause duration.
In case of big capacity of heap, a phase with moving objects and compaction takes too much time. Moreover it requires suspending application threads, in order to change references to new locations of relocated objects. **ZGC** solves this problem concurrently without *stop-the-world pause*. The trick that lets **Z Garbage Collector** do it is **load GC barriers** and **coloured references**.

There are 3 stages of GCs work:
1. Concurrent Mark - identify objects as reachable
2. Concurrent Relocate - to avoid memory fragmentation or promote object to Old Generation, move objects to another region
3. Concurrent Remap - change references to new locations of evacuated objects.

All these stages are performed without *stop-the-world pause*, thus application threads are not suspended. Let's look at how it might be done.
64-bit architecture provides 64 bits for memory addressing. However, not all 64 bits are used. Only 42 bits are used for addressing. 
By this way, other bits might store meta information. **Coloured References** are references where 4 bits are used by **ZGC**.

<img src="/assets/img/garbage-collectors/structure-of-coloured-reference.png" alt="Structure of Coloured References"/>

**Finalizable Bit** means that object is available through a finalizer
**Remapped Bit** denotes that reference is up to date and points to the current location
**Marked 0** and **Marked 1** bits are used to mark reachable objects.

**Load GC barrier** is a piece of code injected by JIT Compiler for all references. Each time when **ZGC** traverse the reference graph or application thread loads a variable through a reference, a load GC barrier checks a **coloured reference** and there can be 2 cases:
1. if **Remapped** bit is 0 then variable might be loaded without any additional actions safely.
2. otherwise there are following actions must be done
    1. reference colour must be fixed
    2. some actions must be done with the object **atomically** depends on the current stage of Garbage Collector, e.g. moving object to another location, change reference to a new location etc.

Like in case of **G1 Collector**, there are generational regions but have different size.
2 phases:
1. *Minor Collection* happens when ZGC works with **Young Generation**.
2. *Major Collection* - the Garbage Collector works with both generations.

As we can see we pays additional cost for loading references but this cost is not huge and it might be a trade-off between 
slight decreasing of application performance and extremely low GC pauses.

The following command is used to enable the **ZGC**:

```bash
java \
-XX:+UseZGC         # enable ZGC
-XX:+ZGenerational  # Generational ZGC is enabled
```

#### Links:
- <a href="https://docs.oracle.com/en/java/javase/11/gctuning/z-garbage-collector1.html#GUID-A5A42691-095E-47BA-B6DC-FB4E5FAA43D0" rel="nofollow">HotSpot Virtual Machine Garbage Collection Tuning Guide</a>
- <a href="https://www.youtube.com/watch?v=WU_mqNBEacw" rel="nofollow">[VDT19] Concurrent Garbage Collectors: ZGC & Shenandoah by Simone Bordet [IT]</a>
- <a href="https://wiki.openjdk.org/display/zgc" rel="nofollow">The Z Garbage Collector (ZGC)</a>

<h2 id="ShenandoahGC">ShenandoahGC</h2>
This garbage collector is available in RedHat OpenJDK & AdoptOpenJDK builds.
It uses the same approach like **Z Garbage Collector**. The notable difference between two garbage collectors is **ShenandoahGC** works also on 32-bit architecture.

Instead of using **Coloured references**, **ShenandoahGC** stores metadata in *HostSpot Object Header*. The header involves **Forward Pointer** that points to a current location or a new location of objects after performing *relocate* stage.
The garbage collector alike ZGC uses **Load GC Barriers**. The logic of work is the same - to check the current phase of the GC work and to perform some actions depends on the phase or just to follow to the point.

<img src="/assets/img/garbage-collectors/forwarding-pointer.png" alt="Forwarding references"/>


There are 3 stages of the GC work:
1. Concurrent Mark - mark all reachable objects as alive
2. Concurrent Evacuation - move live objects to new regions or compact them
3. Concurrent Update References - update references to new location
   All these stages might be merged into one. It is called **Traversal Mode**.

To use this **ShenandoahGC**, enable the JVM flag below:
```bash
java 
-XX:+UseShenandoahGC # enable Shenandoah Garbage Collector
```
#### Links:
- <a href="https://wiki.openjdk.org/display/shenandoah/Main" rel="nofollow">Shenandoah GC</a>

<h2 id="c4">Azul C4 Garbage Collector</h2>
The full name of GC is **Continuously Concurrent Compacting Collector**, or shortly **C4**.
This Garbage Collector might be used with Azul Zulu Builds of OpenJDK. It is generational pauseless GC.
It uses **read barrier** called **The Loaded Value Barrier (LVB)**. Like **ZGC** stores metadata in unused bits of references. It's field named **Not Marked Through** state. Each time when reference is loaded, the barrier checks the field and if additional actions are needed, they are applied.
There are 3 phases of C4 algorithm.
1. Marking  - mark all reachable objects
2. Relocation - move objects in order to free up larger consequent memory regions.
3. Remapping - change the references to a new location of relocated objects.

#### Links:
- <a href="https://www.azul.com/sites/default/files/images/c4_paper_acm.pdf" rel="nofollow">C4: The Continuously Concurrent Compacting Collector</a>
- <a href="https://www.azul.com/products/components/pgc/" rel="nofollow">Azul C4 Garbage Collector</a>
- <a href="https://www.youtube.com/watch?v=ocUx-QUJMfo" rel="nofollow">Really Understanding Garbage Collection Gil Tene Talk QCon SF 2019</a>
- <a href="https://www.infoworld.com/article/2078661/jvm-performance-optimization--part-4--c4-garbage-collection-for-low-latency-java-ap.html" rel="nofollow">JVM performance optimization, Part 4: C4 garbage collection for low-latency Java applications</a>

<h2 id="epsilon">Epsilon Garbage Collector</h2>
This garbage collector is pretty simple in comparison to the mentioned earlier ones. It doesn't collector garbage. With enabled  **Epsilon GC**  sooner or later `java.lang.OutOfMemoryError` occurs. For sure, if you don't have infinite capacity of RAM or your program doesn't allocate new objects at all. At some case, it might be a good option, if latency is important and overhead from a garbage collector is not acceptable e.g. microbenchmarking, garbage free or short-running programs.

With following flags you can enable this **Epsilon Garbage Collector**:

```bash
java \
XX:+UnlockExperimentalVMOptions   # unlock experimental features
-XX:+UseEpsilonGC                 # enable Epsilon Garbage Collector
```

#### Links:
- <a href="https://blogs.oracle.com/javamagazine/post/epsilon-the-jdks-do-nothing-garbage-collector" rel="nofollow">Epsilon: The JDK’s Do-Nothing Garbage Collector</a>