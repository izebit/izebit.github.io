---
layout: post
category: os
title: a life behind proxy
---

This topic is a cheat sheet for people who have to work behind proxy.

There are some placeholders that will be used later. It is obvious to guess what they mean, 
but I give some details to avoid misunderstanding.

**PROXY_URL** - proxy url, that all your traffic goes through.  
**PROXY_PORT** - proxy port.   

**PROXY_USER** - your login for a proxy.   
**PROXY_PASS** - your password.

### [apt](https://wiki.debian.org/Apt) (package manager of debian-based distribution)

```bash
cat /etc/apt/apt.conf.d/proxy.conf
```

```yaml

Acquire::https::Verify-Peer "false";
Acquire::https::Verify-Host "false";

Acquire {
  AllowUnauthenticated "true";
  HTTP::proxy "http://<proxy_user>:<proxy_pass>@<PROXY_URL>:<PROXY_PORT>/";
  HTTPS::proxy "http://<proxy_user>:<proxy_pass>@<PROXY_URL>:<PROXY_PORT>/";
}
```


### [gradle](https://gradle.org/) (dependency manager and build automation tool)

```bash
cat gradle.properties
```

```properties
systemProp.http.proxyHost=<PROXY_URL>
systemProp.http.proxyPort=<PROXY_PORT>
systemProp.http.proxyUser=<proxy_user>
systemProp.http.proxyPassword=<PROXY_USER>

systemProp.https.proxyHost=<PROXY_URL>
systemProp.https.proxyPort=<PROXY_PORT>
systemProp.https.proxyUser=<PROXY_USER>
systemProp.https.proxyPassword=<PROXY_PASS>
```


### [maven](https://maven.apache.org/) (dependency manager for java projects)

```bash
cat ~/.m2/settings.xml
```

Take your notice, file below involves environment variables. If you are not going to use them, 
just put ordinal values instead of `${env.*}`.  

```xml
<settings>
  <proxies>
   <proxy>
      <id>http-proxy</id>
      <active>true</active>
      <protocol>http</protocol>
      <host>${env.PROXY_URL}</host>
      <port>${env.PROXY_PORT}</port>
      <username>${env.PROXY_USER}</username>
      <password>${env.PROXY_PASS}</password>
      <nonProxyHosts>localhost|my-domain</nonProxyHosts>
    </proxy>
    <proxy>
      <id>https-proxy</id>
      <active>true</active>
      <protocol>https</protocol>
     <host>${env.PROXY_URL}</host>
      <port>${env.PROXY_PORT}</port>
      <username>${env.PROXY_USER}</username>
      <password>${env.PROXY_PASS}</password>
      <nonProxyHosts>localhost|my-domain</nonProxyHosts>
    </proxy>
  </proxies>

  ...
</settings>
```

### [sbt](https://www.scala-sbt.org/) (build tool for scala projects, preemptively)

```bash
export SBT_OPTS=" \
-Dhttp.proxySet=true \
-Dhttp.proxyHost=${PROXY_URL} \
-Dhttp.proxyPort=${PROXY_PORT} \
-Dhttp.proxyUser=${PROXY_USER} \
-Dhttp.proxyPassword=${PROXY_PASS} \
-Dhttp.nonProxyHosts=my-domain \
-Dhttps.proxySet=true \
-Dhttps.proxyHost=${PROXY_URL} \
-Dhttps.proxyPort=${PROXY_PORT} \
-Dhttps.proxyUser=${PROXY_USER} \
-Dhttps.proxyPassword=${PROXY_PASS} \
-Dhttps.nonProxyHosts=my-domain \
```


### git (you, definitely, know what it is) 🙂

```bash
cat ~/.gitconfig
```

```yaml
[http]
    proxy = http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
    sslVerify = false
[https]
    proxy = http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
    sslVerify = false
[http "local-git-repo"]
    proxy = ""
    sslVerify = false
[https "local-git-repo"]
    proxy = ""
    sslVerify = false
```


### [curl](https://curl.se/) (tool for downloading some stuff from the internet)

```bash
export http_proxy=http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
export https_proxy=http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
```

### [pip](https://pypi.org/project/pip/) (dependency manager for python projects)

Pip uses environment variables that curl uses, but there is another way to work with it.    
The program has a flag called `--proxy`:
```bash
pip install --proxy http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT> ...
```

You may have some troubles because of wrong SSL certificates. 
To say PIP to ignore this type of errors, you can also pass the param with all used repositories: 
`--trusted-host=pipy.org --trusted-host=files.pythonhosted.org ...`.   


### [npm](https://www.npmjs.com/) package manager for javascript projects. 

To work with npm behind proxy, you should type only 3 commands:

```bash
npm config set strict-ssl false
npm config set https-proxy http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
npm config set http-proxy http://<PROXY_USER>:<PROXY_PASS>@<PROXY_URL>:<PROXY_PORT>
```
