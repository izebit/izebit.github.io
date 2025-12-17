---
layout: post
tags: 
  - java
title: About design patter builder 
---

Today we are going to talk on a design pattern __Builder__.
We start from the very begging, so try to find out for what the design patter is required.   
The pattern belongs to a _creation patterns_ category. It means you can create objects with a builder.
It lets you split creation an object into several phases. Have a look at an example below. It demonstrates a typical use-case of
__Builder__.   

```java
public class Person {
    public final String name;
    public final int age;

    public Person(String name, int age) {
        this.age = age;
        this.name = name;
    }
}
```

In order to create the instance of class `Person`, we should know values of _name_ and _age_ parameters. 
A builder of this class is an intermediate class that contains all this values.   

```java
public class PersonBuilder {
    private int age;
    private String name;

    public PersonBuilder setAge(int age) {
        this.age = age;
        return this;
    }

    public PersonBuilder setName(String name) {
        this.name = name;
        return this;
    }

    public static PersonBuilder builder() {
        return new PersonBuilder();
    }

    public Person build() {
        return new Person(name, age);
    }
}
```

We can see, that this class looks pretty surplus. However, when we have to deal with many parameters and/or check them 
with some rules, we see all advantages of use of __Builder__ more clearly. We move all constructions related to 
creation new instances from the class `Person` to its builder class `PersonBuilder`. By this way, we do not overwhelm 
the first one with additional functionality.   

By now, it is possible to create an object by this way:   
```java
Person person = PersonBuilder
        .builder()
        .setAge(26)
        .setName("Artem")
        .build();
```  
Have a notice, that parameters can be set arbitrarily. Sometimes, it is not what we actually want. For instance, 
in case when some parameter check depends on several parameters. Thus, we use another approach:  
```java
public class PersonBuilder {
    private int age;
    private String name;

    public static PersonNameBuilder builder() {
        return new PersonNameBuilder(new PersonBuilder());
    }

    public Person build() {
        return new Person(name, age);
    }


    public static class PersonNameBuilder {
        private final PersonBuilder builder;

        private PersonNameBuilder(PersonBuilder builder) {
            this.builder = builder;
        }

        public PersonAgeBuilder setName(String name) {
            builder.name = name;
            return new PersonAgeBuilder(builder);
        }
    }

    public static class PersonAgeBuilder {
        private final PersonBuilder builder;

        private PersonAgeBuilder(PersonBuilder builder) {
            this.builder = builder;
        }

        public PersonBuilder setAge(int age) {
            builder.age = age;
            return builder;
        }
    }
}
```
What's going here?  Each time, when some parameter is set, a builder returns an instance of a new builder with only one method.
A method lets set a parameter and return a new builder and so on. By this way, we can guarantee the order of operations:

1. set a name
2. set an age
3. create a new person.

Moreover, the order is checking at compile time.

```java
Person person = PersonBuilder
        .builder()
        .setAge(26) // <-- compile error
        .setName("Artem")
        .build();</code></pre>
```

Without any doubts, the code above is not good to read. By renaming methods, we can create DSL looks like a natural language.

```java
Person man = Person
                   .withName("Artem")
                   .andSurname("Konovalov")
                   .Lives()
                         .in("Saratov")
                         .at("Moscovskay st.")
                   .Works()
                         .as("Programmer")
                         .foR("Some company")
                         .earning(1_000_000)
                         .inCurrency(DOLLARS)
                   .Hobbies()
                         .is("programming")
                         .and()
                         .is("sport");
```

But it requires more code and time to think about your API properly 🙂. I want not to overburden the article,
so I skip the implementation of the example above.


From time to time, a necessity of extending already existing builder classes occurs. A basic class can be extended 
and more and more child classes emerge. In this case, we expect to reuse an existing builder and add some additional methods.
A code of a new child class `Citizen` below:

```java
public class Citizen extends Person {
    public final String address;

    public Citizen(String name, String surname, String address) {
        super(name, surname);
        this.address = address;
    }
}
```

The basic builder:

```java
public abstract class BasePersonBuilder<B extends BasePersonBuilder<B, P>, P> {
    protected String name;
    protected String surname;

    @SuppressWarnings("unchecked")
    public B setSurname(String surname) {
        this.surname = surname;
        return (B) this;
    }

    @SuppressWarnings("unchecked")
    public B setName(String name) {
        this.name = name;
        return (B) this;
    }

    public abstract P build();
}
```

Thus, the builder of the class `Person` looks like this:
```java
class PersonBuilder extends BasePersonBuilder<PersonBuilder, Person> {
    @Override
    public Person build() {
        return new Person(name, surname);
    }
}
```

and a builder of class `Citizen`:
```java
class CitizenBuilder extends BasePersonBuilder<CitizenBuilder, Citizen> {
    private String address;

    public CitizenBuilder setAddress(String address) {
        this.address = address;
        return this;
    }

    @Override
    public Citizen build() {
        return new Citizen(name, surname, address);
    }
}
```

Creating new instances of these classes remains as easy and simple as it was.

```java
Person person = new PersonBuilder()
        .setName("Max")
        .setSurname("Pain")
        .build();

Citizen citizen = new CitizenBuilder()
        .setAddress("USA, NY")
        .setName("Join")
        .setSurname("Smith")
        .build();</code></pre>
```

That is all what I wanted to share with you today. Thank you for reading 😉