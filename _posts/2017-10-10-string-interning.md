---
layout: post
tags:
  - java
title: String interning
---

In java, as well known, strings are stored in a cache. This technique of optimization is called __string interning__.
To use memory effectively, all the same strings that encounters in a source code will refer to the same object during 
execution.

Give a look at the code below:

```java
String s1 = "hi";
String s2 = new String("h") + "i";
System.out.println(s1.equals(s2));
```

> false

The first string `s1` is known to the compiler, the second string `s2` is created in runtime hence 
the compiler does not apply the optimization and there are two different objects with type string.


By this way, all strings _"hi"_ that are known to the compiler (created with _new_ operator or set explicitly) will refer to the same object.
We can change the first object `s1` and will see that other strings with the same value will be affected.

```java
Field field = s1
                .getClass()
                .getDeclaredField("value");
field.setAccessible(true);

Field modifiers = Field
                    .class
                    .getDeclaredField("modifiers");
modifiers.setAccessible(true);
modifiers.setInt(field, field.getModifiers() & ~Modifier.FINAL);

field.set(s1, "bye".toCharArray());
```

After it, all string objects with value "hi" refers to the string object with value "bye" 😃. 

```java
System.out.println(s1.equals("bye"));
```

> true