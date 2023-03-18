---
layout: post
title: Debugging inside Kubernetes pod
lang: en
categories:
    - Kubernetes
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

Containers where designed to isolate an application, so it can get a consistent environment and you can run it anywhere and expect same results.
In practice, things are a bit different.... and it is common to deal with an app that works perfectly on dev and staging environments, but looses it on production. On that moments, the developers are so sorry they didn't put enough logs on their little app, but its too late. That brings us back to the old days, and have know choice but to debug on production.

I experienced this a few times, while migrating the company's old PHP apps from the servers to our kubernetes cluster, and here I will show you how I do debug PHP7 app on an alpine k8s container, with xdebug and VScode editor, in 6 simple steps.

- "6? comeon, thats to many steps. nobody's gonna read this through"
- "chill, they're baby-small steps. After a few times practice I could finish the whole process in less than a minute. Besides, you have no choice, cause you got no logs and your production is down."

### How its going to work (skip this part if prod is really down);

Lets see how PHP debugging actually work. well, once you enable debugging on your text editor, its becoming a server - starts  listening on the debugging port, usually 9000. On the other hand, your app, which runs with xdebug extension enabled, is the client - it is logging debugging data right to that port 9000.
What I do is to use ngrok to publicly expose my local port 9000, and once it is exposed - install xdebug on the app container and send debugging logs straight out to my exposed port.

### Your side configurations:

#### 1. install ngrok:
   
- [Download the binary](https://ngrok.com/download), and unzip it.
- [signup to ngrok (its free)](https://dashboard.ngrok.com/signup) and **copy your token**.
- Run `ngrok authtoken <YOUR_AUTH_TOKEN>` to complete the installation.

#### 2. start ngrok:

```
ngrok tcp 9000
``` 
the output will be:

<img src="{{"/assets/img/ngrok-connected.png" | relative_url }}">

Keep the terminal open until you are done debugging.

The important thing here is this line: `Forwarding         tcp://0.tcp.ngrok.io:18033 -> localhost:9000`

It means that every request to `tcp://0.tcp.ngrok.io:18033` will be forwarded to `localhost:9000`.

#### 3. copy the pod content:

```
kubectl cp <namespace>/<pod>:/ ~/.kube/kube-debug1
```

#### 4. set debugger configurations:

run: 
```
mkdir ~/.kube/kube-debug1/.vscode
echo $'
"version": "0.2.0",
    "configurations": [
        {
            "name": "Listen for XDebug",
            "type": "php",
            "request": "launch",
            "port": 9000,
            "stopOnEntry": true,
            "pathMappings": {
                "/":"${workspaceRoot}/",
            }
        },
' >> ~/.kube/kube-debug1/.vscode/launch.json
```

### Pod side configurations:

#### 5. get inside the pod: 

`kubectl exec -tin <namespace> <pod> sh`

#### 6. install & run xdebug in the container:

```sh
# apk if you run alpine linux. 
# Otherwise use your own package manager:
apk add xdebug 

echo $'
zend_extension=/usr/lib/php7/modules/xdebug.so
xdebug.remote_enable=1
xdebug.remote_port=10035
xdebug.remote_log = /var/log/xdebug/xdebug.log
xdebug.remote_autostart=1

; replace with your ngrok external address:
xdebug.remote_host=0.tcp.ngrok.io:18033
' >> /etc/php7/conf.d/xdebug.ini
```
and now you restart the server. I use fpm so I run:

`killall php-fpm7`


### Done! `
hit `code ~/.kube/kube-debug1/` to open the code in VScode, and enable debgging.
I hope your internet connection is reliable, otherwise you are going to suffer.

Enjoy!
