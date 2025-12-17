---
layout: post
tags: 
   - java
title: Hand-made collector
image: /assets/img/custom-collector.jpg
---

<img src="/assets/img/custom-collector.jpg" alt="custom collector"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@sknart" rel="nofollow">Snejina Nikolova</a> on Unsplash
</sup></sub>


After release java 1.8, we can work with collection with stream API. 
By this way, we are able to create a sequence of operations that are applied to elements from a some source such 
as collection, generating function to create either finite or infinite sequence, lines from a file etc. 
This topic is very broad and discussing it is out of the scope of the current article, and we are not going to talk about it. 


In this article we are going to look at the <a href="https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html" rel="nofollow">Collectors</a> closely. 
This kind of objects that let us create a result of performing the whole sequence of operations. 

Different types of collectors produce result in different way and different formats. There are already
several collectors in `java.util.stream.Collectors` such as __toList__ that creates a list of elements. 
Also, some collectors has more complex implementation, for instance, __groupBy__  that groups elements 
by some criteria. 

Our goal is to write a custom collector by our own. The new collector will group strings by the first letter.

First, we should get to know the interface `java.util.stream.Collector` that we will implement.

```java
interface Collector<T,A,R> {
        Supplier<A>          supplier();
        BiConsumer<A,T>      accumulator();
        BinaryOperator<A>    combiner();
        Function<A,R>        finisher();
        Set<Characteristics> characteristics();
}
```

There are only five signatures. Some of them are not supposed to be implemented. 
It is optional and needed only in some specific cases. These cases will be covered later.

1. __supplier()__ creates additional helper object that performs basic operations on elements. 
2. __acumulator()__ . It combines the result that we calculated earlier and the current element of the stream. 
It takes an object helper and the current element as parameters.
3. __combiner()__ . In case if stream is parallel, this function combines 
the intermediate results performed by different threads.
4. __finisher()__ returns the final result of execution.
5. __characteristics()__ According to the name, it returns collector's charactristics. It worths to list them:
   * _CONCURRENT_ the collector can work with parallel stream.
   * _UNORDERED_ the collector does not garantee the order of elements. It means if a stream has some order, 
   the final result does not nessasary have the same order. 
   * _IDENTITY_FINISH_ If a collector has this characteristic, execution __finisher()__ function is skipped.


By now, we know everything what to need to implement our own collector. Let us start with a class helper.

```java
public final class CustomCollectorBuilder {
    private final Map<Character, Set<String>> set;
 
    public CustomCollectorBuilder() {
        this.set = new HashMap<>();
    }
  
    public void add(String str) {
        Set<String> subSet = getSubset(str);
        
        if (subSet == null) 
            return;
        
        subSet.add(str);
    }
  
    public CustomCollectorBuilder addAll(Map<Character, Set<String>> collection) {
        for (Map.Entry<Character, Set<String>> entry : collection.entrySet()) {
            Set<String> subSet = getSubset(entry.getKey());
            subSet.addAll(entry.getValue());
        }
        return this;
    }
            
    public Map<Character, Set<String>> build() {
        return this.set;
    }
  
    private Set<String> getSubset(String str) {
        if (str == null || str.length() == 0) 
            return null;
        return getSubset(str.charAt(0));
    }
    private Set<String> getSubset(Character key) {
        Set<String> subSet = set.get(key);
        if (subSet == null) {
            subSet = new HashSet<>();
            set.put(key, subSet);
        }
        return subSet;
    }
}
```

I hope this code is clear for you and you do not have problems with understanding it.

The next step is to write the collector. 

```java
public final class CustomCollector implements Collector<String, 
                                                        CustomCollectorBuilder, 
                                                        Map<Character, Set<String>>> {

        @Override
        public Supplier<CustomCollectorBuilder> supplier() {
            return CustomCollectorBuilder::new;
        }

        @Override
        public BiConsumer<CustomCollectorBuilder, String> accumulator() {
            return CustomCollectorBuilder::add;
        }

        @Override
        public BinaryOperator<CustomCollectorBuilder> combiner() {
            return (first, second) -> first.addAll(second.build());
        }

        @Override
        public Function<CustomCollectorBuilder, Map<Character, Set<String>>> finisher() {
            return CustomCollectorBuilder::build;
        }

        @Override
        public Set<Characteristics> characteristics() {
            return EnumSet.of(Characteristics.CONCURRENT, Characteristics.UNORDERED);
        }
}
```

As you can notice, according to collector's characteristics, 
the collector works with parallel streams and does not guarantee any order of elements.  

That is all what I wanted to share with you. The example of usage of the new collector is demonstrated below:


```java
Collection<String> strings = Arrays.asList("apple", "orange", "banana", "pear", "peach");
Map<Character, Set<String>> result = strings
                                            .stream()
                                            .collect(new CustomCollector());

for (Character character : result.keySet())
    System.out.println(result.get(character));
```

The result:

> [pear, peach]    
> [apple]    
> [banana]   
> [orange]  