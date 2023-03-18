---
layout: post
title: Tweaking cluster DNS
lang: en
categories:
    - Kubernetes
    - DNS
    - ExternalName
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---

I use Kubernetes external names and endpoints for custom DNS mapping over the cluster. the most important lesson I learned from the work that was done here is to look for the simple stuff. k8s got build in solutions for so many things, all you have to do is learn how it works.

## use case 1:

You have to migrate to k8s a legacy huge monolith, called `face-app`, which appears to contain a hard coded reference to a database named `face-mysql`. you would like to point it to your new RDS instance url, which is `face-db.bis650gy3b0g.us-east-2.rds.amazonaws.com`, but its impossible to make the change in the codebase because it might break down the production.

### solution: 

k8s [external-name service](https://kubernetes.io/docs/concepts/services-networking/service/#externalname) :tada:

>Services of type ExternalName map a Service to a DNS name, not to a typical selector. You specify these Services with the spec.externalName parameter.

ExternalName this is a special type of service where you point a hostname to another one.
Behind the scene kubernetes creates for you a CNAME DNS record, which will finally affect the DNS requests:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: face-mysql
  namespace: face-namespace
spec:
  type: ExternalName
  externalName: face-db.bis650gy3b0g.us-east-2.rds.amazonaws.com
```	
  
once this file had been kubectl-applied, every DNS request for `face-mysql` in that namespace, will return a CNAME record answer, with the value `face-db.bis650gy3b0g.us-east-2.rds.amazonaws.com`:


```console
~ $ nslookup face-mysql
Server:         172.20.0.10 
Address:        172.20.0.10#53

face-mysql.face-mysql.svc.cluster.local      canonical name = face-db.bis650gy3b0g.us-east-2.rds.amazonaws.com.
Name:   face-db.bis650gy3b0g.us-east-2.rds.amazonaws.com
Address: 10.120.11.27
```

## use case 2: 
I am about to deploy the notorious `face-app` monolith on a little staging cluster I have created in my servers farm. alongside the cluster there is a staging mysql server. its IP on the network is `10.0.234.56`.
as you remember from use-case 1, the app refers to `face-mysql` as its DB. if I would like to point that name to my server IP, I have a few options:

### Solutions

#### 1. The classic solution
Use [host-alias](https://kubernetes.io/docs/concepts/services-networking/add-entries-to-pod-etc-hosts-with-host-aliases/#adding-additional-entries-with-hostaliases):

>Adding entries to a Pod’s /etc/hosts file provides Pod-level override of hostname resolution when DNS and other options are not applicable. In 1.7, users can add these custom entries with the HostAliases field in PodSpec.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: face-app
spec:
  hostAliases:
  - ip: "10.0.234.56"
    hostnames:
    - "face-mysql"
```

that means you have to set the `host-alias` entry for every deployment that needs your DB. I tried to use it but quickly it became such a headache to maintain like 12 different legacy apps * 2 environments, so I came up with a little cleaner solution, keep reading:

#### 2. *My* solution

This one requires a bit knowledge about how k8s service actually work. Apparently I have that knowledge and I will share it with you:
While you create a service + deployment, under the hood k8s is creating another resource type, called `EndPoints`. lets go ahead and hit `kubectl get endpoints` to see what I am talking about. the output will be something like this:

```yaml
NAME							ENDPOINTS                                AGE
face-app-svc					10.120.110.220:80,10.120.130.28:80       2d8h
another-app-svc					10.120.120.60:80,10.120.130.121:80       2d8h
```

the endpoints objects points an IP to a terget pod, And being updated every time a pod is created or died. It is like a map for the service to refer every time a request has to be delivered. The service is doing a round robin between the Endpoints IPs list and choosing a pod to handle the request.

taking it a bit further - why not create my own EndPoints object?

there you go now - here is my EndPoints object + the headless service referring to it:

```yaml
kind: Endpoints
apiVersion: v1
metadata:
 name: face-mysql
 namespace: face-namespace
subsets:
 - addresses:
     - ip: 10.0.234.56
   ports:
     - name: "3306"
       port: 3306
---
kind: Service
apiVersion: v1
metadata:
 name: face-mysql
 namespace: face-namespace
spec:
 ports:
 - name: "3306"
   port: 3306
   targetPort: 3306
```
once this is applied - no need to set any `hostAliases`- every request in that namespace to `face-mysql` service will be redirected to its single EndPoint - `10.0.234.56`

## Creating a dedicated helm chart
I managed to find a solution for every problem I had, but I still had to repeat it on every namespace, for the `face-mysql` hostname, and many others.
I wanted to control it more easily so I created a dedicated helm chart and called it `external-hosts`, and released it everywhere it was needed. 
[full source](https://gist.github.com/Efrat19/9a428d3730f859e2bf43c7b98587737a) available on GitHub. 
Its cool, I know.



