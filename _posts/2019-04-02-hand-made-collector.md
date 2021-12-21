---
layout: post
category: java
title: Hand-made collector
---

After release java 1.8, we can work with collection with stream API. 
By this way, we are able to create a sequence of operations that are applied to elements from a some source such 
as collection, generating function to create either finite or infinite sequence, lines from a file etc. 
This topic is very broad and discussing it is out of the box and the current article 
is not supposed to cover it. 


In this article we are going to look at the [Collectors](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Collector.html) closely. 
This kind of objects that let us create a result of performing the whole sequence of operations. 

Different types of collectors produce result in different way and different formats. There are already
several collectors in `java.util.stream.Collectors` such as __toList__ that creates a list of elements. 
Also, some collectors has more complex implementation, for instance, __groupBy__  that groups elements 
by some criteria. 

Our goal is to write a custom collector by our own. The new collector will group strings by the first letter.

First, we should get to know the interface `java.util.stream.Collector` that we will implement.

```java
interface Collector&lt;T,A,R&gt; {
        Supplier&lt;A&gt;          supplier()
        BiConsumer&lt;A,T&gt;      acumulator()
        BinaryOperator&lt;A&gt;    combiner()
        Function&lt;A,R&gt;        finisher()
        Set&lt;Characteristics&gt; characteristics()
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
    private final Map&lt;Character, Set&lt;String&gt;&gt; set;
 
    public CustomCollectorBuilder() {
        this.set = new HashMap&lt;&gt;();
    }
  
    public void add(String str) {
        Set&lt;String&gt; subSet = getSubset(str);
        
        if (subSet == null) 
            return;
        
        subSet.add(str);
    }
  
    public CustomCollectorBuilder addAll(Map&lt;Character, Set&lt;String&gt;&gt; collection) {
        for (Map.Entry&lt;Character, Set&lt;String&gt;&gt; entry : collection.entrySet()) {
            Set&lt;String&gt; subSet = getSubset(entry.getKey());
            subSet.addAll(entry.getValue());
        }
        return this;
    }
            
    public Map&lt;Character, Set&lt;String&gt;&gt; build() {
        return this.set;
    }
  
    private Set&lt;String&gt; getSubset(String str) {
        if (str == null || str.length() == 0) 
            return null;
        return getSubset(str.charAt(0));
    }
    private Set&lt;String&gt; getSubset(Character key) {
        Set&lt;String&gt; subSet = set.get(key);
        if (subSet == null) {
            subSet = new HashSet&lt;&gt;();
            set.put(key, subSet);
        }
        return subSet;
    }
}
```

I hope this code is clear for you and you do not have problems with understanding it.

The next step is to write the collector. 

```java
public final class CustomCollector implements Collector&lt;String, CustomCollector.CustomCollectorBuilder, Map&lt;Character, Set&lt;String&gt;&gt;&gt; {

        @Override
        public Supplier&lt;CustomCollectorBuilder&gt; supplier() {
            return CustomCollectorBuilder::new;
        }

        @Override
        public BiConsumer&lt;CustomCollectorBuilder, String&gt; accumulator() {
            return CustomCollectorBuilder::add;
        }

        @Override
        public BinaryOperator&lt;CustomCollectorBuilder&gt; combiner() {
            return (first, second) -&gt; first.addAll(second.build());
        }

        @Override
        public Function&lt;CustomCollectorBuilder, Map&lt;Character, Set&lt;String&gt;&gt;&gt; finisher() {
            return CustomCollectorBuilder::build;
        }

        @Override
        public Set&lt;Characteristics&gt; characteristics() {
            return EnumSet.of(Characteristics.CONCURRENT, Characteristics.UNORDERED);
        }
}
```

As you can notice, according to collector's characteristics, 
the collector works with parallel streams and does not guarantee any order of elements.  

That is all what I wanted to share with you. The example of usage of the new collector is demonstrated below:


```java
Collection&lt;String&gt; strings = Arrays.asList("apple", "orange", "banana", "pear", "peach");
Map&lt;Character, Set&lt;String&gt;&gt; result = strings
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