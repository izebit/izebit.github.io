---
layout: post
tags: 
  - hibernate
title: Caching of hibernate
---

Recently, I faced a task related to caching queries in <a href="https://hibernate.org/" rel="nofollow">Hibernate framework</a>. 
I would like to tell about various types of caches in hibernate and how to use and tune them. 

First, there are tree cache levels:
- session level cache;
- second level cache;
- query level cache.   

### Session level cache
According to its name, it is used during one session to database. Each session has own cache and when a session is closed, 
a cache is cleaned up. To be more clear, I list an example below:  

```java
Session session = HibernateUtil.getSessionFactory().openSession();
try {
    Criteria criteria = session.createCriteria(Account.class);
    criteria.add(Restrictions.eq("id", id));
    Account firstAccount = (Account) criteria.uniqueResult();
    Account secondAccount = (Account) criteria.uniqueResult();
} finally {
    session.close();
}
```
In this case, we see only query to a database. The second call `criteria.uniqueResult()` returns a value from a cache.
The session cache level is enabled by default, and you have to do nothing to use it in contrast to other ones.  
Hibernate framework does not provide implementations for second cache level and query cache level. In order to use them,
you have to add some third-party libraries to your project.

### The second level cache

This type of cache, or it is also so called _sessionFactory_ affects all sessions.
By default, it is disabled. As I said before, you should add some implementation of the cache. 
I use <a href="http://www.ehcache.org/" rel="nofollow">ehcache</a> but you can use whatever you want. It is worth to mention, 
the configuration in the article is tested with hibernate 4.0.   

I add some dependencies to my _pom.xml_:

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>${hibernate.version}</version>
</dependency>
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-ehcache</artifactId>
    <version>${hibernate.version}</version>
</dependency>
<dependency>
    <groupId>net.sf.ehcache</groupId>
    <artifactId>ehcache-core</artifactId>
    <version>2.6.11</version>
</dependency>
``` 

and also add some parameters to hibernate configuration:

```xml
<property name="hibernate.cache.use_second_level_cache">
  true
</property>
<property name="hibernate.cache.region.factory_class">
  org.hibernate.cache.ehcache.SingletonEhCacheRegionFactory
</property>
<property name="net.sf.ehcache.configurationResourceName">
  ehcache.xml
</property>
```


To tune _Ehcache_, you should create a file _ehcache.xml_ with parameters:

```xml
<ehcache>
  <diskStore path="java.io.tmpdir"/>
  <defaultCache
            maxElementsInMemory="100000"
            eternal="false"
            statistics="true"
            timeToIdleSeconds="120"
            timeToLiveSeconds="120"
            overflowToDisk="true"
            diskPersistent="false"
            diskSpoolBufferSizeMB="500"
            diskExpiryThreadIntervalSeconds="120"
            memoryStoreEvictionPolicy="LRU"/>
</ehcache>
```
To avoid misunderstanding, I should to give some remarks.

- if _eternal_ option is true, objects will never be removed from a cache.  
- _timeToLiveSeconds_ - how many seconds, an object lives in a cache before removing from it.
- _diskStore_ option contains _path_ parameter that should be set to a path, where you want objects will be saved. In my case, it is a temporary directory. 

It is almost all what we have to do. The last thing is to say hibernate that which entities we want to cache.  
If we set mapping via xml, add an option `<cache usage="read-write"/>`, but if you use annotations, you should add 
`@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)` to an entity class.

A _usage_ parameter can have parameters:  
- when _read-write_ is set, objects are updates concurrently according to a transaction level.
- _nonstrict-read-write_ is supposed to update object in cache without blocking in multithreading mode. 
  However, using this mode, objects retrieved from cache might be outdated. This mode is a trade-off between performance and consistency.
- _read-only_ can be useful for object that never changed.


### Query level cache

To use the query level cache, you should add an implementation to your project as well. _Ehcache_ is an appropriate one and 
the configuration from previous section is legal for a query level cache. 
Also, you should add `<property name="hibernate.cache.use_query_cache">true</property>` to your hibernate configuration and that is all.
Each time when you want queries to be cached, you should just call `setCacheable(true)` for them.

