---
layout: post
tags: 
  - java
title: The best singleton
---

This article is not about what the singleton is and even not about design patterns. Today, I talk on what 
is the best implementation of this admirable pattern.

To be honest, I used to think that __double-check__ is the best choose. However, after reading the article on
<a href="https://www.logicbig.com/tutorials/core-java-tutorial/java-multi-threading/happens-before.html" rel="nofollow">java memory model</a>
I realized that the world would be never the same. It turns up, __volatile__ is not cheap, and it is barely better than
__synchronized__ block. According to this fact, if you need lazy initialization and thread-safe implementation, 
you should use the approach below:

```java
public final class Singleton {
      private Singleton(){};
      
      public static Singleton getInstance(){
          return SingletonCreator.INSTANCE;
      }
      
      private static class SingletonCreator {
          private static final Singleton INSTANCE = new Singleton();
      }      
}
```

This code is __lazy__, because class `SingletoneCreator` is not initialized until it is not needed and not used in other places of program.
But there is only one place where this class might be used. This place is a method `getInstance`.

This code is __thread safe__ because the field `INSTANSE` is final and java memory model guarantees us that all final fields 
have been initialized only once and will be consistent for all threads.


