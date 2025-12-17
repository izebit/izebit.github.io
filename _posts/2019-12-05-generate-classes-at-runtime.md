---
layout: post 
tags: 
  - java 
title: Code generation at runtime
image: /assets/img/code-generation-at-runtime.jpg
---

<img src="/assets/img/code-generation-at-runtime.jpg" alt="code generation at runtime"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@sigmund" rel="nofollow">Sigmund</a> on Unsplash
</sup></sub>


At some points, we need to generate classes with a particular implementation. 
Unfortunately, Java  is not as so dynamic as Groovy, JavaScript or Python. Anyway, there are several ways for doing it.
One of them is generating bytecode outright but this way is sophisticated. 

In this article we consider another way. It involves four steps:  
1. to write java code
2. to compile
3. to load 
4. to execute.

The standard java library contains `javax.tools.JavaCompiler` class. We need it to compile code. The method below does it 
and stores a class in a temp directory. It returns the path to the compiled class.

```java
private static String compile(String className, String code) throws Exception {
    JavaCompiler compiler = ToolProvider.getSystemJavaCompiler();
    StandardJavaFileManager fileManager = compiler.getStandardFileManager(null, null, null);
    Path output = Files.createTempDirectory("_" + System.currentTimeMillis());

    JavaCompiler.CompilationTask task = compiler
                .getTask(null,
                        fileManager,
                        null,
                        Arrays.asList("-d", output.toAbsolutePath().toString()),
                        null,
                        singletonList(new JavaSourceFromString(className, code))
                );

    boolean result = task.call();
    if(!result) throw new IllegalStateException("something wrong happened");

    return output.toAbsolutePath().toString();
}
```
As you can see, method `getTask` takes a collection of objects as the last parameter. These objects are resources contain 
information where source code can be gathered. For instance, it can be a file or something else.   

Unfortunately, there is no implementation for simple string resources, so we can do it on our own. 

```java
private static class JavaSourceFromString extends SimpleJavaFileObject {
     private final String code;

     private JavaSourceFromString(String name, String code) {
          super(URI.create("string:///" + name.replace('.', '/') + Kind.SOURCE.extension), Kind.SOURCE);
          this.code = code;
     }

     public CharSequence getCharContent(boolean ignoreEncodingErrors) {
           return code;
     }
} 
```

When we have a compiled class, we should load it with a class loader:
```java
private static ClassLoader getClassLoader(String classPath) throws Exception {
  return new URLClassLoader(
                new URL[]{Paths
                        .get(classPath)
                        .toUri()
                        .toURL()
                },
                null);
} 
```
After it, having ClassLoader, we can create a new object with type `Class<?>`. The class is not defined in compile time,
so we can't invoke any methods except listed in `java.lang.Object` directly. Thus, we should use <a href="https://www.oracle.com/technical-resources/articles/java/javareflection.html" rel="nofollow">Reflection</a>.

```java
private static void method(Class<?> type, String methodName) throws Exception {
    Method method = type.getDeclaredMethod(methodName);
    method.setAccessible(true);
    System.out.println(method.invoke(type.newInstance()));
}
```

The code below creates two new classes with the same name `ru.izebit.Person`. 
These classes have a method `say` with different implementation. After loading the classes, it invokes these methods.

```java
ClassLoader classLoader = CustomCompiler.getClassLoader("ru.izebit.Person",
                        "package ru.izebit;\n" +
                        "public class Person {\n" +
                        "    public String say() {\n" +
                        "        return \"i am happy 🤠\";\n" +
                        "    }\n" +
                        "}");

method(classLoader.loadClass("ru.izebit.Person"), "say");

classLoader = CustomCompiler.getClassLoader("ru.izebit.Person",
                        "package ru.izebit;\n" +
                        "public class Person {\n" +
                        "    public String say() {\n" +
                        "        return \"i am tired 😫 \";\n" +
                        "    }\n" +
                        "}");
method(classLoader.loadClass("ru.izebit.Person"), "say");
```

I would like to notice, that despite having the same names, the program works without any errors. There is no collision
because of using different class loaders.   
The result of work is below:

```text
i am happy 🤠
i am tired 😫 
```

<a href="https://github.com/izebit/hazelcast-demo/blob/master/client/src/test/java/ru/izebit/client/serialization/DynamicUtils.java" rel="nofollow">
    <img src="/assets/img/github-icon.svg" width="100" height="100">
</a>
