---
layout: post
tags: spring
title: how to get rid of hibernate.properties
---

In this article we find out how to move properties from _hibernate.properties_ to spring configuration.

First, we need to define properties for `sessionFactory` like that:

```xml
<bean id="sessionFactory" class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
    <property name="dataSource" ref="dataSource"/>
    <property name="annotatedClasses">
        <list>
            <value>ru.izebit.Model</value>
        </list>
    </property>
    <property name="hibernateProperties">
        <props>
            <prop key="enable_lazy_load_no_trans">false</prop>
            <prop key="hibernate.hbm2ddl.auto">validate</prop>
        </props>
    </property>
</bean> 
```

We can go further and move common properties to tomcat _context.xml_.
```xml
<?xml version='1.0' encoding='utf-8'?>

<context>
    <!--database source-->
    <resource name="jdbc/myDataSource" auth="Container"
              type="javax.sql.DataSource"
              driverClassName="oracle.jdbc.OracleDriver"
              url="jdbc:oracle:thin:@izebit.ru:1521:orcl"
              username="user" password="password"
              maxActive="100" maxIdle="10" maxWait="-1" initialSize="10"/>

    <!--hibernate dialect-->
    <environment name="hibernate.dialect" type="java.lang.String" value="org.hibernate.dialect.Oracle10gDialect"/>
    <!--hibernate schema-->
    <environment name="hibernate.default_schema" type="java.lang.String" value="user_schema"/>
</context>
```

then add props to spring context configuration: 

```xml
<jee:jndi-lookup id="dataSource" jndi-name="jdbc/myDataSource" expected-type="javax.sql.DataSource"/>
<jee:jndi-lookup id="dialect" jndi-name="java:comp/env/hibernate.dialect" expected-type="java.lang.String"/>
<jee:jndi-lookup id="schema" jndi-name="java:comp/env/hibernate.default_schema" expected-type="java.lang.String"/>
   
<bean id="sessionFactory" class="org.springframework.orm.hibernate4.LocalSessionFactoryBean">
        <property name="hibernateProperties">
            <props>
                <prop key="enable_lazy_load_no_trans">false</prop>
                <prop key="hibernate.hbm2ddl.auto">validate</prop>
                <prop key="hibernate.dialect">#{dialect}</prop>
                <prop key="hibernate.default_schema">#{schema}</prop>
            </props>
        </property>    
</bean>
```

Sometimes, we can not get rid of _hibernate.properties_ completely, but some parameters will be set dynamically from 
tomcat context. In order to do it, we will create a bean, that will set needed parameters during application bootstrap.

```xml
<!--load props from hibernate.properties-->
<util:properties id="hibernateProperties" location="classpath:spring/hibernate.properties"/>

<!-- getting parameters from tomcat context.xml -->
<jee:jndi-lookup id="dialect" jndi-name="java:comp/env/hibernate.dialect" expected-type="java.lang.String"/>

<bean lazy-init="false" id="dialectSetter" class="org.springframework.beans.factory.config.MethodInvokingFactoryBean">
        <property name="targetObject" ref="hibernateProperties"/>
        <property name="targetMethod" value="setProperty"/>
        <property name="arguments">
            <array>
                <value>hibernate.dialect</value>
                <ref bean="dialect"/>
            </array>
        </property>
</bean>

<bean id="sessionFactory" 
      class="org.springframework.orm.hibernate4.LocalSessionFactoryBean" 
      depends-on="dialectSetter">
        <property name="hibernateProperties" ref="hibernateProperties"/>
</bean>
```

There is one important point is order of bean initialization. The bean _dialectSetter_ must to earlier than 
_sessionFactory_. For that reason, we set its dependencies `depends-on = "dialectSetter"` explicitly.