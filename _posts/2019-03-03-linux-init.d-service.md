---
layout: post
tags: 
  - linux
title: How to create init.d system service 
---


Sometimes it is needed not to only run program, but be sure that this process will work after reboot a host or in some unexpected fails.
There are several ways. One of them is to look after the process manually and another one is to let an operating system takes care about it on its own.
All you need for it is to make your program a system service. This article is about exactly this case. To be more precisely, 
it says how to create a system service with __update-rc.d__. 

Look at it closely and break the process down into several steps:
1. create a script to rule life cycle of the program.
2. grant all needed permissions to the script
3. register the created script as a linux system service
4. enable the system service

As you can see, it is not big deal. 

### 1. create a script

On the first stage, we should create a file __my-service__ located in a directory _/etc/init.d/_.

```shell
touch /etc/init.d/my-service
```

The template of the script is as follows:

```shell
#!/bin/bash
#
### BEGIN INIT INFO
# Provides:             my-service
# Required-Start:       $syslog $remote_fs
# Required-Stop:        $syslog $remote_fs
# Should-Start:         $local_fs
# Should-Stop:          $local_fs
# Default-Start:        2 3 4 5
# Default-Stop:         0 1 6
# Short-Description:    My Service
# Description:          My Service
### END INIT INFO
#
### BEGIN CHKCONFIG INFO
# chkconfig: 2345 55 25
# description: My Service
### END CHKCONFIG INFO
 
NAME="my-service"
APPLICATION_PATH="/opt/my-super-program"
PID_FILE="$APPLICATION_PATH/file.pid"
COMMAND="/usr/bin/python $APPLICATION_PATH/script.py 2> &1 > /dev/null & "


# run the program and write its pid to a file.
start() {
    echo "Starting $NAME"
    eval $COMMAND
    PID=$(pgrep -f $NAME) 
    echo $PID >> $PID_FILE
    RETVAL=0
}

# stop the process 
# get pid value from the file and kill the process with it.
stop() {
    if [ -f $PID_FILE ]; then
        echo "Shutting down $NAME"
        # kill the process
        kill -9 $(cat $PID_FILE)
        rm -f $PID_FILE
        RETVAL=0
    else
        echo "$NAME is not running."
        RETVAL=0
    fi
}
 
# restart
# nothing complex
restart() {
    stop
    start
}

# in case if list of running processes contains a process run from the a certain directory,
# it mean that our program has been already started otherwise it is stopped.
status() {
    echo `ps -ef` | grep -q "$APPLICATION_PATH"
    if [ "$?" -eq "0" ]; then
        echo "$NAME is running."
        RETVAL=0
    else
        echo "$NAME is not running."
        RETVAL=3
    fi
}

# a function will be called according to a given command 
case "$1" in
    start)
        start
        ;;
    stop)
        stop
        ;;
    status)
        status
        ;;
    restart)
        restart
        ;;
    *)
        echo "Usage: {start|stop|status|restart}"
        exit 1
        ;;
esac
exit $RETVAL
```

Given template is denoted to create a service that is started from the directory __APPLICATION_PATH__ by the command __COMMAND__.
To check the status of the process, __PID_FILE__ is needed. The file contains a pid value of running process. 
Important, the program has to be run as a daemon and does not print information to console. The prefix `_2> &1 > /dev/null &` does exactly 
what we need.

From the previous script, we can notice that the script involves 3 parts:
1) header with comments and system information. This part contains, for example, dependencies of the service and in which order 
they have to be started and stopped. Also, it contains the system title and system description. 
2) the main part that handles all received commands.
3) functions that implement stop, start, status check and so on operations.

### 2. grant permissions

To run the script, we have to grant some permissions to the file. The command below does it:

```shell
chmod +x /etc/init.d/my-service 
```

### 3. register a script as a system service

In order to make operating system know about existence of our service, we have to register it with next command:

```shell
update-rc.d my-service defaults
```

### 4. enable a system service

However, it is not all. After registering, our service will be known to the OS and we can execute some command on it.
For instance, to run or to stop:

```shell
service my-service start
```

But, the service will not be started automatically after reboot a host. It is default behavior for Linux. To change it, 
we have to execute a command with next parameters:

```shell
update-rc.d my-service enable
```

After it, your service is completely read to work. 😉
