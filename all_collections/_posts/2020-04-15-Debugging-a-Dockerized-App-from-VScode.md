---
layout: post
title: Debugging a Dockerized App from VScode
lang: en
categories:
    - docker
    - PHP
    - Xdebug
    - VScode
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---

So you are running your PHP app locally on a docker container, and you also got your code mounted to your working directory, so you can edit your code live. But how can you debug it? In this post I will cover the steps for installing & configuring xdebug inside your php container, so you can run debugging sessions from your VScode IDE.

**Disclaimer** This post looks very similar to another one I posted a couple of months ago, about [debugging an app that runs on a kubernetes pod](/debugging-inside-kubernetes-pod). Except in this one I show you how to debug an app when the container is running on your **local machine**, via docker or docker compose. the technique is quite similar but the configurations are not, So I made this one too cause I remember how I struggled upon this and I think I people might find it useful.

### Understanding XDEBUG mechanism 

Lets see how PHP debugging actually work. well, once you enable debugging on your text editor, its becoming a server - starts  listening on the debugging port, usually 9000. On the other hand, your app, which runs with xdebug extension enabled, is the client - it is logging debugging data right to that port 9000. So all I got to do is to tell the container which server to look for, and its going to be my very own local machine address on the docker network, which is `172.17.0.1` on linux, or `docker.for.mac.localhost` on mac.

### Installation steps:

#### Create an `xdebug.ini` Configuration File:
```ini
[Xdebug]
xdebug.idekey=VSCODE
zend_extension=/usr/lib/php7/modules/xdebug.so
xdebug.remote_enable=1
xdebug.remote_port=9000
xdebug.remote_autostart=1

; for linux users: 
xdebug.remote_host=172.17.0.1 

; for mac users:
xdebug.remote_host=docker.for.mac.localhost
```

notice the `xdebug.remote_host` setting varies between the operating systems. 
> :bulb: If you got to make the debugger **both** linux and mac compatible, keep in mind you can use an environment variable to override the settings, for instance you can set `xdebug.remote_host=172.17.0.1` in the `xdebug.ini` file, so it fits linux users, and mac users will override it with an injected environment variable `XDEBUG_CONFIG=remote_host=docker.for.mac.localhost` yeah I just saved you hours of despair you welcome

#### Install Xdebug and Copy the Conf to the Dev Image 
Add the following to the Dockerfile
  ```Dockerfile
# apk if you run alpine linux. 
# Otherwise use your own package manager:
RUN apk add xdebug 

# and copy the conf:
COPY xdebug.ini /etc/php7/conf.d/
```

#### VSCODE Configurations:

Now lets head to VScode and open up the project. We add the following debugging configurations to the `.vscode/launch.json` file:
```json
"version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for XDebug",
            "type": "php",
            "request": "launch",
            "port": 9000,
            "stopOnEntry": true,
            "ignore": [
                    "**/vendor/**/*"
                ]
            "pathMappings": {
                "/projectPathInsideYourContainer":"${workspaceRoot}",
            }
        },
```

#### Done! Start the docker container and [start VSCODE debugging](https://code.visualstudio.com/docs/editor/debugging). Once a request will hit your app, an xdebug session will start and you will be free to debug it as much as you want.
