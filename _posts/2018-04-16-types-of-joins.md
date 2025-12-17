---
layout: post
tags: 
  - sql
title: Types of joins
image: /assets/img/join-inner.png
---

To my shame I still do not know all types of joins. This article is my try to figure out in this topic.   

First, we should understand what join is.

> Join is an operation that union tables by some columns. 

There are several types of them:
1. left inner
2. right inner
3. left outer
4. right outer
5. some other types. 

The main difference between all of them is what rows the result includes.
Later, we will look at some examples in order to comprehend it better.

We will use 2 tables  _workers_ and _phones_ in our examples.

```sql
select * from workers;
```    
output:   
```text
---------------
name  | type
---------------
john  | developer
steve | manager
bruce | seller
neal  | architect

```

```sql
select * from phones;
```   
output:   
```text
------------------
name   | phone
------------------
john   | 100
jeramy | 103
steve  | 102
zach   | 101

```
To better understand, we will execute sql queries with different type of joins.

### left inner join

A query is given:
```sql
select * from workers left inner join phones on workers.name = phones.name;
```    

The result of execution of the query includes workers who has phone numbers.

```text
-------------------------------
name  |type        | phone
-------------------------------
john  | developer  | 100
steve | manager    | 102
```

<a href="https://en.wikipedia.org/wiki/Venn_diagram" rel="nofollow">Venn Diagram</a> illustrates result very well.  

<img src="/assets/img/join-inner.png" alt="inner join"/>

### left outer join

```sql
select * from workers left outer join phones on workers.name = phones.name
```
output:
```text
-------------------------------
name  |type         | phone
-------------------------------
john  | developer   | 100
steve | manager     | 102
bruce | seller      | null
neal  | architect   | null
```

The result involves, as you can see, list of __all__ workers.

<img src="/assets/img/join-left-outer.png" alt="left outer join"/>

### right outer join 

A query with this type of join works similarly to the previous query, but there 
is only difference is that the result contains all rows of the second table.  

```sql
select * from workers right outer join phones on workers.name = phones.name
```

```text
-------------------------------
name   |type         | phone
-------------------------------
john   | developer   | 100
steve  | manager     | 102
jeramy | null        | 103
zach   | null        | 101
```

<img src="/assets/img/join-right-outer.png" alt="right outer join"/>

### inner join and right inner join

The result of executions of these queries with __inner join__, __right inner join__ and __left inner join__ are identical.

```sql
select * from workers inner join phones on workers.name = phones.name
```

```text
-------------------------------
name  |type        | phone
-------------------------------
john  | developer  | 100
steve | manager    | 102
```

### full outer join
Beside basic types, there are several additional types. They are used rarely, but sometimes you can find them useful.
One of them is __full outer join__.


```sql
select * from workers full outer join phones on workers.name = phones.name
```

After execution the query abow, we will have rows from the results of __left outer join__ and __right outer join__.

```text
-------------------------------
name   |type         | phone
-------------------------------
john   | developer   | 100
steve  | manager     | 102
jeramy | null        | 103
zach   | null        | 101
bruce  | seller      | null
neal   | architect   | null
```

<img src="/assets/img/join-full-outer.png" alt="full outer join"/>


Moreover, we can use _where_ for the query and get rows that do not match.

```sql
select * from workers full outer join phones on workers.name = phones.name where phones.phone is null or workers.type is null;
```

The result:  
```text
-------------------------------
name   |type         | phone
-------------------------------
jeramy | null        | 103
zach   | null        | 101
bruce  | seller      | null
neal   | architect   | null
```

<img src="/assets/img/join-without-inner-part.png" alt="inner join without inner part"/>

### cross join

It is one of specific type of joins. The result of execution involves all possible combinations of rows from two tables.
It is called <a href="https://en.wikipedia.org/wiki/Cartesian_product" rel="nofollow">Cartesian product</a>.
The count of the result equals a product of count of rows of both tables.
```sql
select * from workers cross join phones
```

### others

This part is about some useful queries with different types of joins. 

If we can get a list of workers who do not have phone numbers, we can use the query below:

```sql
select * from workers left outer join phones where phones.phone is null
```

The result:
```text
-------------------------------
name   |type         | phone
-------------------------------
bruce  | seller      | null
neal   | architect   | null
```

<img src="/assets/img/join-right-anti.png" alt="left anti join"/>


You can get a set of workers who do not work and who have a phone number with the next query:

```sql
select * from workers right outer join phones where workers.type is null
```

The result:
```text
-------------------------------
name   |type         | phone
-------------------------------
jeramy | null        | 103
zach   | null        | 101
```

<img src="/assets/img/join-left-anti.png" alt="right anti join"/>

That is all what I want to share with you. I hope the article will come in handy.

[Visualization of join types](https://izebit.ru/sql-join-visialization.html)

