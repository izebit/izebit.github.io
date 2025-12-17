---
layout: post
tags:
  - java
  - architecture
title: Is 3-tier Architecture anti-pattern?
---

In this topic, I would like to discuss about 3-tier architecture, what kind of drawbacks it has and what might be used instead of it.

<img src="/assets/img/3-tier-architecture/thumbnail.png" alt="spring-modulith and hexagonal architecture"/>

## Three-tier Architecture
Let's talk about **three-tier architecture**. It's a well-known pattern that defines in which way we should organise modules of our programs. 
The 3-tier architecture is very popular across web applications, but it might be applied to not only implementing this type of application. 
Based on the name, you might guess that there are 3 tiers:

1. Presentation Layer
2. Logic Layer
3. Data Layer  

**Presentation Layer** contains logic that is in charge of providing data via REST or other protocols. The code of this layer handles requests, takes needed data, 
transforms the data to required formats and sends responses. Usually, a framework is responsible for most of these actions but a programmer still needs to declare 
API and to define from which place data will be taken and where data should be passed to.  
**Logic Layer** contains business logic e.g. creating new objects, changing values of existing domain objects and so on. 
It's the most interesting part of the application.  
The last one is the **Data Layer**. This layer provides access to a database in a convenient way. Due to the **Data Layer**, 
we don't have to think about how to persist and retrieve objects from a database. Moreover, in theory, this abstraction tier 
lets switch database type without changing business logic e.g. from MySQL to PostgreSQL, or even to a NoSQL solution.  
As a rule, each tier is represented by a package that contains all classes associated with a corresponding tier. Let's look at an example. 
It is a simple content management system. The system provides a Web Interface and REST API to operate posts, comments, and users. 
The picture below depicts the structure of the 3-tier architecture of the application. For convenience, I mark the related classes with the same emojis.  
<pre>
.
└──ru
   └── izebit
           ├── ApplicationLauncher.java 
           ├── controller
           │   ├── UserController.java 👽
           │   ├── PostController.java 🗒️
           │   └── CommentController.java ✏️
           ├── repository
           │   ├── UserRepository.java 👽
           │   ├── PostRepository.java 🗒️
           │   ├── CommentRepository.java ✏️
           │   └── model
           │       ├── User.java 👽
           │       ├── Post.java 🗒️
           │       └── Comment.java ✏️
           └── service
               ├── UserService.java 👽
               ├── AuthenticationService.java 👽
               ├── StopSpamService.java ✏️
               ├── CommentService.java ✏️
               ├── PostService.java 🗒️
               ├── SpellCheckerService.java 🗒️
               └── dto
                   ├── User.java 👽
                   ├── Post.java 🗒️
                   └── Comment.java ✏️
</pre>

As we can see, there are classes grouped in packaged by type, not by domain. All classes must be public because they should 
be accessible from other packages, otherwise, controllers can't work with services and services can't work with repositories.  
Moreover, not related classes are also accessible to each other inside a package, for example, `AuthenticationService` and `SpellCheckerService`.
In this way, based only on the structure, it's tough to understand the relationship between classes. In addition, the current package structure 
contradicts the <a href="https://en.wikipedia.org/wiki/Encapsulation_(computer_programming)" rel="nofollow">Encapsulation principle</a>.  

> A language mechanism for restricting direct access to some of the object's components.
>-  <cite>Wikipedia</cite>

To restrict access to components from other packages, we might place all related classes into the same "business logic" package 
and expose only classes that are used by other components. Other classes might have restricted modifiers e.g. package-private or private modifiers.
This approach can be implemented with **Hexagonal Architecture**.  

## Hexagonal Architecture

<img src="/assets/img/3-tier-architecture/hexagonal-arhitecture.png" alt="hexagonal architecture"/>

The backbone of **Hexagonal Architecture** is **Ports** and **Adapters**. Business logic has boundaries and is isolated from other components. 
There is no direct access to it from other packages.  
**Ports** provide a shareable API that might be used to work with our business logic and vice versa i.e. to define API for interacting 
with the external world as well. Thus, it stands for "<ins>WHAT</ins> our system can do and other systems can do with".  
**Adapters** define in which way ports might be invoked and which way our system might work with other components. One port might have many adapters. 
In this way, adapter stands for "<ins>HOW</ins> the system and other systems interact each other".  
This approach lets us write maintainable code with isolated domain-specific logic. All dependencies are represented as interfaces and might be mocked 
if necessary. It makes writing tests easier. Also, it affects flexibility because we can change adapters without touching core logic. 
In other words, this design encourages  <a href="https://en.wikipedia.org/wiki/Orthogonality_(programming)" rel="nofollow">Orthogonality</a>.  

Let's look at the structure of the app with **Hexagonal Architecture**:
<pre>
.
└── ru
    └── izebit
        ├── ApplicationLauncher.java
        ├── comment
        │   ├── CommentServiceImpl.java
        │   ├── StopSpamService.java
        │   └── ports
        │       ├── in
        │       │   ├── CommentService.java
        │       │   └── adapters
        │       │       └── CommentController.java
        │       ├── models
        │       │   └── Comment.java
        │       └── out
        │           ├── CommentRepository.java
        │           ├── UserService.java
        │           └── adapters
        │               ├── CommentRepositoryImpl.java
        │               └── UserServiceImpl.java
        ├── post
        │   ├── PostServiceImpl.java
        │   ├── SpellCheckerService.java
        │   └── ports
        │       ├── in
        │       │   ├── PostService.java
        │       │   └── adapters
        │       │       └── PostController.java
        │       ├── models
        │       │   └── Post.java
        │       └── out
        │           ├── PostRepository.java
        │           ├── UserService.java
        │           └── adapters
        │               ├── PostRepositoryImpl.java
        │               └── UserServiceImpl.java
        └── user
            ├── AuthenticationService.java
            ├── UserServiceImpl.java
            └── ports
                ├── in
                │   ├── UserService.java
                │   └── adapters
                │       └── UserController.java
                ├── models
                │   └── User.java
                └── out
                    ├── UserRepository.java
                    └── adapters
                        └── UserRepositoryImpl.java
</pre>

In comparison to the previous one, this structure brings a few additional interfaces and packages, but at the same time, 
it gives more information on the relationships between components.  
The `protected` package visibility modifier hides all domain-specific classes. Only components of packages `*/ports/in/` and `*/ports/model/` 
must be `public` i.e. accessible from other packages. By now, there's still no clear relationship between packages. 
We need to provide public access to only particular packages.  
For instance, during working with comments and posts, we need to check users if they are authenticated, they present in blacklist etc. 
It means that classes from the `post` and `comment` packages should have access to public classes from the `user` package, but the reverse is false.  
Limiting access might be implemented with <a href="https://www.oracle.com/corporate/features/understanding-java-9-modules.html" rel="nofollow">Java Modules</a>, 
but we discover another approach.  

## Spring Modulith
This Spring component lets developers implement logical modules in Spring Boot applications. 
It supports structural validation, generation of Module Component diagrams etc. In this post, we are going 
to use a few modules of this Spring component.

First of all, we need to mark our public packages with the `NamedInterface` annotation.   
By default, all top-level packages are considered as modules and there is no need to define them explicitly. 
We do it only for sub-packages.
 ```java
@org.springframework.modulith.NamedInterface("in")  
package ru.izebit.user.ports.in;
```

Then we should define dependencies for each package. For this purpose, we use the following annotation:
 ```java
@org.springframework.modulith.ApplicationModule(  
        allowedDependencies = "user::in"  
)  
package ru.izebit.comment;
``` 
This information on packages are stored in `package-info.java` files placed in the relevant directories.

After these two steps, we can write a simple test that checks correct packages access.
```java
@Test  
void shouldBeCompliant() {  
    ApplicationModules.of(ApplicationLauncher.class)
				      .verify();  
}
```
If access is broken, we can see the following output:
```bash
org.springframework.modulith.core.Violations: - Module 'comment' depends on module 'user' via ru.izebit.comment.ports.in.adapters.CommentController -> ru.izebit.user.ports.out.UserRepository. Allowed targets: user::in.
```
The error message is descriptive and gives enough information to find out the root of the issue.  
In addition to checking package access violations, one of the features of **Spring Modulith** is generation of UML diagrams.
If you make the code below a part of build pipeline, you will always have up-date diagrams of your system.  
```java
void writeDocumentationSnippets() {  
    new Documenter(modules)  
            .writeModuleCanvases()  
            .writeModulesAsPlantUml()  
            .writeIndividualModulesAsPlantUml();  
}
```

<img src="/assets/img/3-tier-architecture/c4-plantUML-diagram.png" alt="c4 uml diagram"/>

The picture depicts <a href="https://github.com/plantuml-stdlib/C4-PlantUML" rel="nofollow">C4 PlantUML diagram</a> of the application.  

You can find the source code of the application <a href="https://github.com/izebit/spring-modulith-example" rel="nofollow">here</a>.
<a href="https://github.com/izebit/spring-modulith-example" rel="nofollow">
    <img src="/assets/img/github-icon.svg" width="100" height="100" alt="project on GitHub">
</a>

#### Links:
- <a href="https://medium.com/ssense-tech/hexagonal-architecture-there-are-always-two-sides-to-every-story-bc0780ed7d9c" rel="nofollow">Hexagonal Architecture, there are always two sides to every story</a>
- <a href="https://spring.io/blog/2022/10/21/introducing-spring-modulith/" rel="nofollow">Introducing Spring Modulith</a>
- <a href="https://docs.oracle.com/javase/tutorial/java/javaOO/accesscontrol.html" rel="nofollow">Controlling Access to Members of a Class</a>