---
layout: post
tags: 
  - java
title: How to deal with jar hell
---

One day during deploy new a version of an app on test stage, I faced an exception **NoSuchMethodException**.   
A log file contained the next stacktrace:  
```log
java.lang.NoSuchMethodError: org.slf4j.spi.LocationAwareLogger.log(Lorg/slf4j/Marker;Ljava/lang/String;ILjava/lang/String;[Ljava/lang/Object;Ljava/lang/Throwable;)V
    at org.apache.commons.logging.impl.SLF4JLocationAwareLog.debug(SLF4JLocationAwareLog.java:133)
    at org.apache.http.impl.conn.tsccm.ThreadSafeClientConnManager$1.getConnection(ThreadSafeClientConnManager.java:221)
    at org.apache.http.impl.client.DefaultRequestDirector.execute(DefaultRequestDirector.java:401)
    at org.apache.http.impl.client.AbstractHttpClient.execute(AbstractHttpClient.java:820)</code></pre>
```
I was confused because it had worked perfectly on my local machine. 
The most likely cause of that error was a conflict of different versions of the same dependency. In java world
this situation is called _java hell_. So how to deal with it?

The first step is run your app with an option: `-verbose:class`. A jvm print a list of all loaded classes. 
You may see something like the output below.

```bash
java -jar -verbose:class my_super_app.jar

[Opened /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.lang.Object from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.io.Serializable from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.lang.Comparable from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.lang.CharSequence from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.lang.String from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
[Loaded java.lang.reflect.GenericDeclaration from /usr/lib/jvm/java-7-openjdk-amd64/jre/lib/rt.jar]
```

So, after it, we have information about a need class and a jar name that it comes from. 
The next step is find out how the dependency turns out in your project. In my case I use maven, but I am sure, 
other build tools have a command with pretty the same functionality.   
The command `mvn dependency:tree` shows a three of dependencies of your project. 
It is useful because we can see really all dependencies (include transitive ones!). 

```log
[INFO] +- org.glassfish.jersey.media:jersey-media-json-jackson:jar:2.12:compile
[INFO] |  +- org.glassfish.jersey.core:jersey-common:jar:2.12:compile
[INFO] |  |  +- javax.annotation:javax.annotation-api:jar:1.2:compile
[INFO] |  |  +- org.glassfish.jersey.bundles.repackaged:jersey-guava:jar:2.12:compile
[INFO] |  |  +- org.glassfish.hk2:hk2-api:jar:2.3.0-b10:compile
[INFO] |  |  |  \- org.glassfish.hk2.external:aopalliance-repackaged:jar:2.3.0-b10:compile
[INFO] |  |  +- org.glassfish.hk2:hk2-locator:jar:2.3.0-b10:compile
[INFO] |  |  \- org.glassfish.hk2:osgi-resource-locator:jar:1.0.1:compile
```

The final step is find a dependency with wrong version and name that we know from the previous step.
In most cases, the problem goes away after excluding the dependency in your _pom.xml_.     

```xml
<dependencies>
    ... 
       <dependency>
            <groupId>org.springframework</groupId>
            <artifactId>spring-context</artifactId>
            <version>${spring.version}</version>
            <exclusions>
                <exclusion>
                    <groupId>commons-logging</groupId>
                    <artifactId>commons-logging</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    ...
</dependencies>      
```

If it does not help, repeat the process again 🙂.  