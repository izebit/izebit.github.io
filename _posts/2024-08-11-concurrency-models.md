---
layout: post
tags:
  - concurrency
title: 3 Concurrency Models
---

First, the goal of the post isn't covering all existing concurrency models. The post is inspired by the book 
"Seven Concurrency Models in Seven Weeks When Threads Unravel" by Paul Butcher.

Before diving into the topic, it seems important to remind the difference between concurrency and parallelism. 
These two terms are pretty similar but have different meanings. The good explanation I have found in the book "Solution Architect's Handbook" by Shrivastava:

> People often get confused between parallelism and concurrency by thinking they are both the same thing; however, 
> concurrency is different from parallelism. In parallelism, your application divides an enormous task into smaller 
> subtasks, which it can process in parallel with a dedicated resource for each subtask. In concurrency, however, 
> an application processes multiple tasks simultaneously by utilising shared resources among the threads.

In other words, concurrency is a part of implementation, but parallelism is a way of combining different tasks and isn't part of internal implementation.  
Now, we are ready to move forward. 

## Mutex and blockers

The model is straightforward and intuitive. When we work with this model, we have to deal with threads 
face-to-face. We have to think about how to split a task into subtasks, how to initiate threads, how to 
synchronize threads by mutex and locks. Convenience and in which level programming is error-prone highly depends on 
tools that programming language gives and guarantees that its memory model provides. 

To make it clear, let's write a simple example that calculates the count of words. We can do it concurrently. We use here 
the classic *Producer-Consumer Pattern*. There are two types of threads: *Readers* and *Counters*. 

The *Readers* fetches file content, splits text into words and put them in the common queue. The last readers 
operation is decreasing number of running readers.

```java
@AllArgsConstructor  
private static class Reader implements Callable<Boolean> {  
    private final Queue<String> queue;  
    private final String file;  
    private final AtomicInteger countOfRunningReaders;  
  
    @Override  
    @SneakyThrows   
    public Boolean call() {  
        Files.lines(Path.of(file))  
                .flatMap(line -> Arrays.stream(line.split(" ")))  
                .forEach(queue::add);  
  
        countOfRunningReaders.decrementAndGet();  
        return true;  
    }  
}
```

The *Counters* read words from the queue and calculate numbers of words. If the count of running readers equals 0, *Counters* stop working.

```java
@AllArgsConstructor  
private static class Counter implements Callable<Map<String, Integer>> {  
    private final BlockingQueue<String> queue;  
    private final AtomicInteger countOfRunningReaders;  
  
    @Override  
    @SneakyThrows    
    public Map<String, Integer> call() {  
        var counter = new HashMap<String, Integer>();  
  
        while (countOfRunningReaders.get() > 0) {  
            var word = queue.poll(200, TimeUnit.MILLISECONDS);  
            if (word == null)  
                continue;  
            counter.merge(word, 1, (k, v) -> v + 1);  
        }  
  
        return counter;  
    }  
}
```

The last step is to initiate the queue and thread pool. It will be used to submit our readers and counters. When *Counters* 
finish working, we get word numbers calculated by each *Counter* and combine their results into a sharable map. 

```java
var files = List.of("file1.txt", "file2.txt", "file3.txt");  
var countOfRunningReaders = new AtomicInteger(files.size()); 
var queue = new LinkedBlockingQueue<String>();  

var readers = files  
        .stream()  
        .map(file -> new Reader(queue, file, countOfRunningReaders))  
        .toList();  
var counters = IntStream.range(0, 3)  
        .mapToObj(e -> new Counter(queue, countOfRunningReaders))  
        .toList();  
  
var threadPool = Executors.newFixedThreadPool(readers.size() + counters.size());  
threadPool.invokeAll(readers);  
var totalMap = new HashMap<String, Integer>();  
for (Future<Map<String, Integer>> future : threadPool.invokeAll(counters)) {  
    var result = future.get();  
    result.forEach((k, v) -> totalMap.merge(k, v, (k1, v1) -> v1 + v));  
}  
  
System.out.println(totalMap);
```

## Functional programming

Last years, functional programming is getting popular. Its elements are appearing even in object-oriented programming languages. It has several key concepts:
1. Functions are first-citizen objects. They might be created, passed as arguments, returned as a result.
2. Functions tend to be pure functions, i.e., they don't have side effects and deal only with data taken as parameters.
3. It encourages immutable values. It means if we need to change state, we have to create a new one with different values.

The mentioned above information is enough for us to rewrite the program from the previous chapter in functional style.

Alike the previous example, we need a function that reads file content.

```java
@SneakyThrows  
private static Stream<String> readWords(String file) {  
    return Files
			.lines(Path.of(file))  
            .flatMap(e -> Arrays.stream(e.split(" ")));  
}
```

Then we create a *parallel stream* that performs operations in parallel. As we can see the code is more laconic and 
readable in comparison to the previous example. Thanks to pure functions and their composition.

```java
var result = List
		.of("file1.txt", "file2.txt", "file3.txt")  
        .parallelStream()  
        .flatMap(file -> readWords(file))  
        .map(word -> Pair.create(word, 1))  
        .collect(
			Collectors.toConcurrentMap(
						        Pair::getLeft, 
						        Pair::getRight, 
						        Integer::sum)
		);
```

## Actors

Actor model is another way of working concurrency. An actor stands for an operation. Each operation is performed by single thread associated to 
the operation. This fact lets us not think about synchronization between threads, because code looks like a one-threaded code. 
Actors communicate each other by sending events. An event contains input data for another actor. All sent events are added to an actor queue and handled one by one.
The diagram below depicts how our Word Counter program works:

<img src="/assets/img/3-concurrency-models.png" alt="actor model"/>

**File Reader** actor receives events that contain file names and then sends events to **Line Splitter**. Events contain only 
single line. According to the name, the actor splits lines and sends events. Each event involves a word. 
When **Words Counter** actor receives the event, it changes internal state to keep current number of words.

Also, worth to mention, with the actor model we can create distributed programs easily. Events might be sent over the network as well as within one host.