---
layout: post
tags: 
  - java
title: Converting dates from XMLGregorianCalendar to GregorianCalendar
---

During parsing xml file, I have faced an issue. It was about date converting. 
An object with XMLGregorianCalendar type and with value that was earlier than 1582 year had been converted into XMLGregorianCalendar object wrongly.

```java
XMLGregorianCalendar beforeDate = DatatypeFactory
                                                .newInstance()
                                                .newXMLGregorianCalendar();
beforeDate.setDay(11);
beforeDate.setMonth(11);
beforeDate.setYear(1581);
XMLGregorianCalendar alterDate = DatatypeFactory
                                                .newInstance()
                                                .newXMLGregorianCalendar();
alterDate.setYear(1582);
alterDate.setMonth(11);
alterDate.setDay(11);

SimpleDateFormat dateFormat = new SimpleDateFormat("dd:MMM:yyyy");
System.out.println(dateFormat.format(beforeDate.toGregorianCalendar().getTime()));

System.out.println(dateFormat.format(alterDate.toGregorianCalendar().getTime()));
```

output:
> 11:Jan:1582


This problem happened because of changing Julian calendar to Gregorian calendar. To disable it we need to set date value explicitly as follows:

```java
GregorianCalendar calendar = new GregorianCalendar();
calendar.clear();
calendar.set(xmlCalendar.getYear(), xmlCalendar.getMonth() - 1, xmlCalendar.getDay());

SimpleDateFormat dateFormat = new SimpleDateFormat("dd:MMM:yyyy");
System.out.println(dateFormat.format(calendar.getTime()));
```

output:
> 11:Jan:1580
