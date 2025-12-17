---
layout: post
tags: 
  - maven
title: Proof of concept of incremental compilation with maven 
---


In this article, I am going to talk about _proof of concept_ of incremental compilation with maven. Its aim to 
boost compilation maven multimodule project.

Let's look at the typical process of build project. In our favorite terminal we type `mvn clean install` than maven builds 
our whole project and copies new created artifact to local repository. It is not big deal and looks trivial. However, 
there are some problems. One of them is the process might take much time. Even if we skip running tests, maven still spends
time on creating _jar_ archive or other type of artifact and finally coping it to _$HOME/.m2/repository_


We can do it faster if we do not perform some step of build process for our multimodule project. As the rule, 
not all modules are affected. According to this fact, we can build modules that actually required to build. But doing it manually is 
too tedious because we have to detect what modules are changed and run build command with special maven flags _-pl_.

There is another solution with usage of git. I hope everyone uses _scv_ for his or her projects and this approach can be adapted for 
any type of it, not only git.   
The key idea is simple. After each time when maven installs artifact, we will add some information about last git commit. The information
lets us distinguish different artifact builds with the same project version. In case if revision is the same, 
to build a submodule again does not make sense, because there is already actual version of artifact in a local repository.


How can we implement it? To do it, we need several maven plugins and to code in _xml_  😃. 

To detect git revision, we add a plugin to deal with git:

```xml
<plugin>
    <groupId>pl.project13.maven</groupId>
    <artifactId>git-commit-id-plugin</artifactId>
    <version>2.2.4</version>

    <executions>
        <execution>
            <id>get-the-git-information</id>
            <goals>
                <goal>revision</goal>
            </goals>
            <phase>validate</phase>
        </execution>
    </executions>
    <configuration>
        <validationProperties>
            <validationProperty>
                <name>validating git dirty</name>
                <value>${git.dirty}</value>
                <shouldMatchTo>false</shouldMatchTo>
            </validationProperty>
        </validationProperties>
    </configuration>
</plugin>
```
 
_git.dirty_ parameter will be set to _true_ if there are some not committed changes. In that case, we will not compare revision versions 
and build will be required anyway. 

Comparing revision versions will be implemented with _ant_ tasks. It is the most "sophisticated" and cumbersome part of the article.

```xml
<!-- wp:code -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>1.8</version>
    <executions>
        <execution>
          <configuration>
            <exportAntProperties>true</exportAntProperties>
            <target>
              <!--add task defintions from ant-contrib-->
              <taskdef resource="net/sf/antcontrib/antcontrib.properties"/>
              <echo>Git revision: ${git.commit.id}</echo>
              <echo>Does repo contain any changes? ${git.dirty}</echo>
    
              <!--получаем путь до артифакта в локальном репозитории-->
              <propertyregex property="directory.name"
                                       input="${project.groupId}"
                                       regexp="[.]"
                                       replace="/"
                                       global="true"/>
              <property name="artifact.path"
                        value="${settings.localRepository}/${directory.name}/${project.artifactId}/${project.version}"/>
              <echo>Artifact path: ${artifact.path}</echo>
    
              <!-- detect if local changes do not exist  and two files: jar archive and file with revision version exist-->
              <condition property="skip.build" value="true" else="false">
                            <and>
                                <isfalse value="${git.dirty}"/>
                                <resourceexists>
                                    <file file="${artifact.path}/${project.build.finalName}.${project.packaging}"/>
                                </resourceexists>
                                <resourceexists>
                                    <file file="${artifact.path}/${git.commit.id}"/>
                                </resourceexists>
                            </and>
              </condition>
    
              <!--if build is required, remove all old files-->
              <if>
                <equals arg1="${skip.build}" arg2="true"/>
                <then>
                  <echo message="Skip build for ${project.artifactId}"/>
                </then>
                <else>
                  <echo message="Rebuild ${project.artifactId}"/>
                  <delete dir="${artifact.path}/" failonerror="false"/>
                  <mkdir dir="${artifact.path}"/>
                  <touch file="${artifact.path}/${git.commit.id}"/>
                </else>
              </if>
            </target>
          </configuration>
          <phase>validate</phase>
          <goals>
            <goal>run</goal>
          </goals>
        </execution>
      </executions>
    <dependencies>
            <dependency>
                <groupId>ant-contrib</groupId>
                <artifactId>ant-contrib</artifactId>
                <version>1.0b3</version>
                <exclusions>
                    <exclusion>
                        <groupId>ant</groupId>
                        <artifactId>ant</artifactId>
                    </exclusion>
                </exclusions>
            </dependency>
    </dependencies>
</plugin>
```

As you might notice, the implementation is simple. The local repository beside artifacts also contains one more file with revision versions.
The file name involves the version of the last commit. If all changes are committed and all required files 
(artifact file and file with revision version) exist,
the build process of submodule will be skipped otherwise clean up and rerun build.

Unfortunately, there are no ways to skip build for a particular submodule in _maven_. Thus, we have to pass skip parameter to all plugins 
that are used in a build process. It is not taught, but it makes our result pom ponderous.

```xml
<plugins>
    <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <configuration>
                <skipMain>${skip.build}</skipMain>
                <skip>${skip.build}</skip>
                <source>1.8</source>
                <target>1.8</target>
            </configuration>
     </plugin>
     <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-resources-plugin</artifactId>
            <version>3.0.0</version>
            <configuration>
                <skip>${skip.build}</skip>
            </configuration>
     </plugin>
     <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-jar-plugin</artifactId>
            <version>2.4</version>
            <configuration>
                <skipIfEmpty>true</skipIfEmpty>
            </configuration>
     </plugin>
     <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-install-plugin</artifactId>
            <configuration>
                <skip>${skip.build}</skip>
            </configuration>
     </plugin>
</plugins>
```

I hope you find the article interesting and useful 🖖.

<a href="https://gist.github.com/izebit/27070793ca954c47f81f3a16f7d392f1" rel="nofollow">
    <img src="/assets/img/github-icon.svg" width="100" height="100">
</a>