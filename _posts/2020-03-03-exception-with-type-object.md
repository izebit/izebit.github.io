---
layout: post
tags: 
  - java
title: How to throw an exception with type `java.lang.Object`
image: /assets/img/exception-with-type-object.jpg
---

<img src="/assets/img/exception-with-type-object.jpg" alt="exception with type object"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@helloimnik" rel="nofollow">Hello I'm Nik</a> on Unsplash
</sup></sub>


As you might know, we can throw exceptions that inherit from `java.lang.Throwable`. 
This class is a parent for all exception in java. `ATHROW` is a byte instruction for throwing exceptions.
I am curious, are there some type checks for this instruction after compilation. 
In the article we will generate a small method with <a href="https://commons.apache.org/proper/commons-bcel/" rel="nofollow">bcel</a> lib.


First of all, create a new class `ru.izebit.Test`. With _bcel_, it looks pretty easy: 

```java
String className = "ru.izebit.Test";
ClassGen classGenerator = new ClassGen(className, "java.lang.Object", "Test", Const.ACC_PUBLIC, new String[0]);
InstructionFactory instructionFactory = new InstructionFactory(classGenerator);
```

The class involves `public static void main(Object...args)` method. Inside the method `main` there is only one command. 
It is an invocation `test` method.

```java
InstructionList instructionListOfMainMethod = new InstructionList();
InstructionHandle invokeHandler = instructionListOfMainMethod.append(
instructionFactory.createInvoke(className, "test", Type.VOID, Type.NO_ARGS, Const.INVOKESTATIC));
instructionListOfMainMethod.append(InstructionFactory.createReturn(Type.VOID));
```
The next code block defines the method `main` with input parameters.

```java
MethodGen mainMethodGenerator = new MethodGen(
                                              Const.ACC_PUBLIC | Const.ACC_STATIC,
                                              Type.VOID, 
                                              new Type[]{new ArrayType(Type.STRING, 1)}, new String[]{"args"},
                                              "main", 
                                              className,
                                              instructionListOfMainMethod, 
                                              classGenerator.getConstantPool());
```

Also, it might come in handy to catch all exceptions occur in the method `test` and print information about them.  

```java
InstructionHandle catchBlock = instructionListOfMainMethod.append(
                                               instructionFactory.createFieldAccess(
                                                                            "java.lang.System", 
                                                                            "out",
                                                                            new ObjectType("java.io.PrintStream"), 
                                                                            Const.GETSTATIC));
 instructionListOfMainMethod.append(InstructionConst.DUP);
 instructionListOfMainMethod.append(
                                              new PUSH(classGenerator.getConstantPool(), 
                                              "An exception has been caught"));
 instructionListOfMainMethod.append(
                                               instructionFactory.createInvoke(
                                                                            "java.io.PrintStream", 
                                                                            "println", 
                                                                            Type.VOID, 
                                                                            new Type[]{Type.STRING}, 
                                                                            Const.INVOKEVIRTUAL));
 instructionListOfMainMethod.append(InstructionFactory.createReturn(Type.VOID));
```

For the catch block, we should declare a `try` block. This block catches all exceptions with type `java.lang.Object`.
It sounds ridicules, but nothing stops us from doing it, so why not?

```java
mainMethodGenerator.addExceptionHandler(
                                          invokeHandler,
                                          invokeHandler.getNext(), 
                                          catchBlock, 
                                          ObjectType.getInstance("java/lang/Object"));
classGenerator.addMethod(mainMethodGenerator.getMethod());
```

The next step is to implement method `test`. All this method does is create a new `java.lang.Object` and throws it.
As you can see, it is not big deal. The code below generates this logic.

```java
InstructionList instructionListOfTestMethod = new InstructionList();
instructionListOfTestMethod.append(instructionFactory.createNew("java/lang/Object"));
instructionListOfTestMethod.append(InstructionConst.DUP);
instructionListOfTestMethod.append(
                instructionFactory.createInvoke(
                                       "java/lang/Object",
                                       "<init>",
                                       Type.VOID, 
                                       Type.NO_ARGS, 
                                       Const.INVOKESPECIAL));
instructionListOfTestMethod.append(InstructionConst.ATHROW);
 instructionListOfTestMethod.append(InstructionFactory.createReturn(Type.VOID));
```

Also, we should define a declaration of the method `public static test() throw Object`:

```java
MethodGen testMethodGenerator = new MethodGen(
                                    Const.ACC_PUBLIC | Const.ACC_STATIC,
                                    Type.VOID, 
                                    Type.NO_ARGS, 
                                    new String[0],
                                    "test", 
                                    className,
                                    instructionListOfTestMethod, 
                                    classGenerator.getConstantPool());
 testMethodGenerator.addException("java/lang/Object");
 testMethodGenerator.stripAttributes(true);
 testMethodGenerator.setMaxStack();
 testMethodGenerator.setMaxLocals();
 classGenerator.addMethod(testMethodGenerator.getMethod());
```
That is all about generation code. We can save our byte code to file. 

```java
String outputPath = this.getClass().getProtectionDomain().getCodeSource().getLocation().getPath() + "ru/izebit/Test.class";
try (FileOutputStream fos = new FileOutputStream(outputPath)) {
      classGenerator.getJavaClass().dump(fos);
}
```

Just in case, if you got lost, I show the generated class completely:

```java
package ru.izebit;

public class Test {
    public static void main(String[] args) {
        try {
            test();
        } catch (Object var1) {
            System.out.println("An exception has been caught");
        }
    }

    public static void test() throws Object {
        throw new Object();
    }
}
```

Trying to execute our program:

```bash
java ru.izebit.Test

Error: Unable to initialize main class ru.izebit.Test
Caused by: java.lang.VerifyError: (class: ru/izebit/Test, method: test signature: ()V) Can only throw Throwable objects
```

The java byte code validation does not let us execute the program. Fortunately, there is an option to disable this check:

```bash
java -Xverify:none ru.izebit.Test
```

Hooray, after it, you can see the result:

```bash
An exception has been caught
```

<a href="https://gist.github.com/izebit/cb3036818fbeb55ef6d0596ba489c7e0" rel="nofollow">
    <img src="/assets/img/github-icon.svg" width="100" height="100">
</a>