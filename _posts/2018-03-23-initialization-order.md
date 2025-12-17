---
layout: post
tags: 
  - java
title: The order of object initialization in java 
---

Have you ever thought about in which order objects in java initialize? 
If you haven't, this article can you help to find it out.  

Given: two classes: **Child** and **Parent**. According to their names, one inherits from another one. 
Each class involves static and ordinal init blocks, static and non-static fields. Let's look at this case more closely.

*Parent.java*
```java
public class Parent {
    private static final String staticField = staticMethod();
    static {
        System.out.println("parent static block");
    }

    {
        System.out.println("parent non-static block");
    }
    private final String field = initMethod();
    private String initMethod() {
        System.out.println("parent non static field");
        return "";
    }

    public static String staticMethod() {
        System.out.println("parent static field");
        return "";
    }

    public Parent() {
        System.out.println("parent constructor");
    }
}
```

*Child.java*  
```java
public final class Child extends Parent {
     private final static String staticField = staticMethod();
     private final String field = initMethod();

     static {
            System.out.println("child static block");
     }

     {
            System.out.println("child non-static block");
     }


     public static String staticMethod() {
            System.out.println("child static field");
            return null;
     }

     private String initMethod() {
            System.out.println("child non static field");
            return null;
     }

     public Child() {
            System.out.println("child constructor");
     }
}
```

If we run compile and run this code by this way:

```java
public static void main(String[] args) throws Exception {
        new Child();
}
```
we will see the next output:

>parent static field  
>parent static block  
>child static field  
>child static block  
>parent non-static block  
>parent non-static field  
>parent constructor  
>child non-static field  
>child non-static block  
>child constructor  


To sum up, we can see that objects inits in the particular order:  
1. all static blocks and fields of a parent class
2. the same happens in a child class 
3. all non-static blocks and fields of a parent class
4. invocation of a parent constructor
5. initialization of non-static fields and blocks in a child class
6. invocation of a child constructor

As we can see, the order is quite logical and easy to remember.
