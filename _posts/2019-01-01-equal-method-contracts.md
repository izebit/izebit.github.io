---
layout: post
tags: 
  - java
title: contacts of Equals and Hashcode methods
---

It is well known that `equals()` method is declared in `java.lang.Object` class. 
It has a naive implementation that compares references of two objects and in case 
if they are the same it returns _true_ otherwise _false_.

There are many situations when we need to override this method. If you want your program works in way you expected, 
you should follow some contracts.

One of them is related to the companion of `equals()` method. It is `hashCode()` method. 
Both methods are extremely important for correct work with data structures based on hash algorithm such as `java.util.HashMap` or `java.util.HashSet`.
The rule for overriding is simple. If equals method returns _true_ for 2 objects, it means that hashCode should return the same value, 
but equality of hash codes does not mean that objects are the same. 
The last part of the statement is explained by <a href="https://en.wikipedia.org/wiki/Hash_collision" rel="nofollow">hash collision</a>.

It is not only thing that you should do. There is one more contact. 
According to it, your implementation of _equals_ method should be:  
1. __reflexive__     
   Given:   
   _a_ is an object,   
   so if we call `a.equals(a)`, the method should always return _true_  
2. __symmetric__   
   Given:    
      _a_ and _b_ are different objects.   
   If `a.equals(b)` returns _true_, `b.equals(a)` must return _true_ as well and vice versa.
   If `b.equals(a)` returns _false_, `a.equals(b)` must return _false_.
3. __transitive__    
    Given:  
      _a_, _b_, _c_ are different objects.    
    If `a.equals(b)` returns _true_ and `b.equals(c)` returns _true_, `a.equals(c)` must be _true_.
4. __consistent__    
    The result of invocation does not depend on how many times we call a method until we do not change the state of objects. 
    In other words, if `a.equals(b)` returns _true_, it will be true until _a_ and _b_ has not been changed.
5. __null__    
    Given:   
      _a_ is an object.   
    The result of `a.equals(null)` is always _false_. 
