---
layout: post
tags: 
  - java
title: Types of references
image: /assets/img/type-of-references-and-gc.jpg
---


<img src="/assets/img/type-of-references-and-gc.jpg" alt="garbage collector and references"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@shanerounce" rel="nofollow">Shane Rounce</a> on Unsplash
</sup></sub>

Today, I am going to talk on type of references. There are four types of them: 
- __strong__
- __weak__
- __phantom__
- __soft__

To use them, except the first type, you should use a particular class from package `java.lang.refs`.   

So, what is the difference between them and what special traits do they have?
The difference is how __garbage collector__ deals with them.

## Strong reference
This type of reference is an ordinary reference that is used each time when you declare an object.

```java
Object obj = new Object();
```

The garbage collector does not collect the object until at least one __strong__ reference exists to the object.
I hope everything is clear about it.


### Weak reference
This type is completely ignored by the garbage collector. Even if there are some __weak__ references point to an object,
the object will be removed by it. Usually, this kind of references can be useful in cases when we want to avoid _memory leaks_.
One of the cases is caches. When there are no references except __weak__ ones, we do not want to store objects in our caches and 
want the garbage collector to remove them. 


### Soft reference
It is similar to the previous type of reference, except one thing. The garbage collector can remove an object that some soft references point to
in case OutOfMemoryError occurs. It means if JVM does not have enough memory, the garbage collector deletes these objects.  

The usage is them is easy:
```java
SoftReference<String> softReference = new SoftReference<>("it is my answer: ");
WeakReference<BigInteger> weakReference = new WeakReference<>(new BigInteger("42"));
System.out.println(softReference.get() + weakReference.get());
```


### Phantom references
Before staring talking about __phantom__ references, I should give some details about in which way garbage collector releases memory.
It goes over a graph of references started from the special vertices __GC roots__ and checks their availability.
These vertices denote local variables, method parameters active threads, JNI references, loaded classes loaded by the system loader.

By this way, if the graph does not involve an object, it is considered to be unused. From the garbage collector's view, it might be removed 
safely. If a method `finalize()` is implemented, an object will be removed in __two stages__:
1. At the first stage, a method `finalize()` is called, 
2. Then the garbage collector makes sure that the object is still unavailable and then deletes the object.

Thus, in a method `finalize()` we might by mistake or on purpose, make an object alive again by passing another object the reference to it.


Getting back to __phantom reference__, it seems a bit of strange type of reference for one thing,  a method `get()` returns always _null_.
This kind of references is used to implement some logic after calling method `finalize()`. Thus, we can implement clean up an object 
in two steps.

Look at creating a __phantom reference__ closely:

```java
PhantomReference<String> phantomReference 
        = new PhantomReference<>("I am a mystical reference type", new ReferenceQueue<>());
```

As you might have noticed, a constructor takes 2 arguments. First of them is an object that a reference points to 
and the second one is a queue.   
What is the point of it? It will be clear when we figure out how garbage collector deals with references.

when the garbage collector detects an object in one of the cases below:
- __weak reference__ exists or 
- __soft reference__ exists for the object and `OutOfMemoryError` occurs or
- at least, one __phantom reference__ exists.

It calls a method `finalize()` and then adds the object to the queue. The object is not removed until the queue contains it.
By this way, we can create a daemon thread, that retrieves elements from the queue and do some stuff. Moreover, it is already implemented in a class
`java.misc.Cleaner`. 

Look at the code below:

```java
Object obj = new Object();
Cleaner.create(obj, () -> System.out.println("the object is removed!"));
```

The code looks very robust and work with __phantom references__ is hided.

Pay your attention that usage of `finalize()` is not recommended in many cases and if you do not know exactly what you do, you
should avoid using finalization mechanism.