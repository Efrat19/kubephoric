---
layout: post
title: kube-probe Basic Auth 
lang: en
categories:
    - Kubernetes
    - helm
    - httpd
    - kube-probe
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---
 

In this post I will demonstrate authorizing k8s probes, using helm templates. This authentication is useful for many auth-requiring services, but here I will focus on my specific use case on a legacy httpd server image.

Do any of you ever heard about `.htpasswd` file? I did not, until the day I had to migrate old httpd server to kubernetes. FYI, back in the old days people used to lock server routes with this file
on the `.htaccess` file you would declare:

```htaccess
AuthUserFile /path/to/htpasswdfile
AuthName "Admin"
AuthType Basic
<Limit GET POST>
 require valid-user
</Limit>
```
and than you would have to supply credentials to access the routes. 

OTOH k8s provides health checks - to keep an eye on the pod liveness and readiness, k8s probes it on a configurable time interval and according to the returned HTTP status, it will make a decision if to pass traffic to the pod or not, and weather it should be restarted.

> Any code greater than or equal to 200 and less than 400 indicates success. Any other code indicates failure.

BUT if all routes are locked, although the server is up and running, the probes will fail with 401:

```console
~ $ kubectl logs <httpd-pod>
10.130.92.253 - probe [19/Feb/2020:09:05:52 +0200] "GET / HTTP/1.1" 401 381 "-" "kube-probe/1.14+"
10.130.92.253 - probe [19/Feb/2020:09:05:57 +0200] "GET / HTTP/1.1" 401 381 "-" "kube-probe/1.14+"
10.130.92.253 - probe [19/Feb/2020:09:06:02 +0200] "GET / HTTP/1.1" 401 381 "-" "kube-probe/1.14+"
10.130.92.253 - probe [19/Feb/2020:09:06:07 +0200] "GET / HTTP/1.1" 401 381 "-" "kube-probe/1.14+"
10.130.92.253 - probe [19/Feb/2020:09:06:12 +0200] "GET / HTTP/1.1" 401 381 "-" "kube-probe/1.14+"
```
and eventually the pod will enter a `CrushLoopBackOff` mode, for the reason:

```bash 
~ $ kubectl describe po <httpd-pod>
........
Warning  Unhealthy  21m (x6 over 22m)     kubelet, ip-10-130-91-184.eu-west-1.compute.internal  Liveness probe failed: HTTP probe failed with statuscode: 401
Normal   Killing    21m (x2 over 22m)     kubelet, ip-10-130-91-184.eu-west-1.compute.internal  Container legacy failed liveness probe, will be restarted
Warning  Unhealthy  17m (x31 over 22m)    kubelet, ip-10-130-91-184.eu-west-1.compute.internal  Readiness probe failed: HTTP probe failed with statuscode: 401
Warning  BackOff    2m31s (x53 over 16m)  kubelet, ip-10-130-91-184.eu-west-1.compute.internal  Back-off restarting failed container
```

### solution: the kube-probe client must carry authentication headers. ###

First I add a new probe user to `.htpasswd`:
```
/var/www/html/site # htpasswd /path/to/htpasswdfile probe
New password: 123456
Re-type new password: 123456
Adding password for user probe
```
the probe user had been added to the file. commit and push the changes to the project repo.

## Adding the Authorization header

in my case, I was designing a helm chart for a few legacy PHP apps, and wanted to enable liveness and readiness probes. In the chart `values.yaml` file I included:

```yaml
...
enableProbes: true
probesTimeout: 3 
htpasswd: true
htpasswdCreds:
  user: probe
  password: "123456"      
...
```

and the `templates/deployment.yaml` had:

```yaml
containers:
  - name: {{ .Chart.Name }}
    {{- if .Values.enableProbes }}
    livenessProbe:
      httpGet:
        path: /
        port: http
        {{- if .Values.htpasswd }}
        httpHeaders:
        - name: Authorization
          value: Basic {{ printf "%s:%s" .Values.htpasswdCreds.user .Values.htpasswdCreds.password | b64enc }}
        {{- end }}
      timeoutSeconds: {{ .Values.probesTimeout }}
    {{- end }}
```

after rendering, it is going to look like:


```yaml
~ $ helm template charts/legacy -f probes-example.yaml
.........
.........
    livenessProbe:
    httpGet:
        path: /
        port: http
        httpHeaders:
        - name: Authorization
          value: Basic cHJvYmU6MTIzNDU2
    readinessProbe:
    httpGet:
        path: /
        port: http
        httpHeaders:
        - name: Authorization
          value: Basic cHJvYmU6MTIzNDU2
.......
```

After `helm install`ing this chart, the probe will succeed:

```console
~ $ kubectl logs <httpd-pod>
10.130.92.253 - probe [18/Feb/2020:23:38:29 +0200] "GET / HTTP/1.1" 200 16955 "-" "kube-probe/1.14+"
10.130.92.253 - probe [18/Feb/2020:23:38:33 +0200] "GET / HTTP/1.1" 200 16955 "-" "kube-probe/1.14+"
10.130.92.253 - probe [18/Feb/2020:23:38:39 +0200] "GET / HTTP/1.1" 200 16955 "-" "kube-probe/1.14+"
10.130.92.253 - probe [18/Feb/2020:23:38:43 +0200] "GET / HTTP/1.1" 200 16955 "-" "kube-probe/1.14+"
10.130.92.253 - probe [18/Feb/2020:23:38:49 +0200] "GET / HTTP/1.1" 200 16955 "-" "kube-probe/1.14+"
```