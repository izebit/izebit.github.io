---
layout: post
tags: 
  - java
title: How to stop java application gracefully
image: /assets/img/shutdown-java-app.jpg
---

<img src="/assets/img/shutdown-java-app.jpg" alt="stop java app properly"/>

<sub><sup>
Photo by <a href="https://unsplash.com/@jack_anstey" rel="nofollow">Jack Anstey</a> on Unsplash
</sup></sub>

Today we are talking about how to stop java process gracefully.

How can we stop your application? The first thing comes to my mind is to kill 
the application by next shell command: `kill -9 <application-pid>`. 
It is possible to save PID app to somewhere, for instance into a file, and then read PID and execute the command. 
However, in reality sometimes it's an appropriate way. Imagine the situation - there is an application and this app has some work to do. 
We can't interrupt its work because, in this case, the state would be inconsistent. 
For example, the application is copying files or performing some transactions or something like that. 
We have to wait the moment when all work has been done, or the app will be ready to stop working on its own.

The code below illustrates the logic of app work:

```java
private boolean isRunning = true;
public void perform(){
    while(isRunning){
      val task = getNextTask();
      if (task == null)
        TimeUnit.SECONDS.sleep(1)
      else
        task.execute();
    }
}
```

### What steps do we have to do in order to stop app gracefully?

First of all, we need to set isRunning to true, then we wait until a task is done.
It's pretty clear, but what initiates this activity? We can send a system termination signal to an application.
As I mentioned earlier, kill <pid application> is an excellent candidate to do it. Please be attentive,
the command without flag `-9` . It's crucial because `kill -9` doesn't let us handle this signal at all.

In java, we can handle a termination signal and add some callback if an application received 
the signal from the operating system or somebody else. `java.misc.Signal` does exactly want we need. The code below creates callback.

```java
Signal.handle(new Signal("TERM"), new SignalHandler() {
             @Override
             public void handle(final Signal signal) {
                 stopServiceAndWaitUtilTaskIsDone();
             }
});
```
**stopServiceAndWaitUtilTaskIsDone** should set **isRunning** to **false** and wait until a task is done.
The second one can be implemented with flags as well. I leave it to do on your own.

By now, we have everything to stop an application properly. Thank you for reading, see you soon 🖖