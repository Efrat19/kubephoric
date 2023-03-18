---
layout: post
title: Migrate fluxcd repo from helm2 to helm3 with (almost) zero downtime
# slug: cvbnjklbjm,.
lang: en
categories:
    - Kubernetes
    - helm
    - fluxcd
    - GitOps
# tags:
#     - hoge
#     - foo
permalink: /:slug 
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---
{% include smooth-scroll.html %}

In my company we use fluxcd gitops operator to manage our kubernetes clusters. We also use flux official helm-operator, to deploy our helm charts. the helm-operator was originally designed for helm2 charts, but once helm V3 was out, fluxcd team did a great work on the helm-operator, and now it is is compatible with both helm2 and helm3 charts. In this post I will cover the steps I took to complete my charts migration.

## What is Flux and how to get started

#### (<a href="#move">skip</a> if you are already a flux ninja <img src="https://emojipedia-us.s3.dualstack.us-west-1.amazonaws.com/thumbs/120/emojipedia/240/ninja_1f977.png" srcset="https://emojipedia-us.s3.dualstack.us-west-1.amazonaws.com/thumbs/240/emojipedia/240/ninja_1f977.png 2x" alt="Ninja on Emojipedia 13.0" width="30" height="30">)

> Flux is a tool that automatically ensures that the state of your Kubernetes cluster matches the configuration you’ve supplied in Git. It uses an operator in the cluster to trigger deployments inside Kubernetes, which means that you don’t need a separate continuous delivery tool.

1. Fork [flux repo](https://github.com/fluxcd/flux-get-started) and rename it. 
For example, my fork is [https://github.com/Efrat19/local-cluster](https://github.com/Efrat19/local-cluster).

2. Follow the official [installation steps](https://github.com/fluxcd/helm-operator-get-started). Those will help you get started with flux + the flux helm-operator, allowing you to gitopsly manage your workloads & helm charts.

1. test:
```console
~ $ kubectl get po -n flux
NAME                              READY   STATUS    RESTARTS   AGE
flux-6b578c8cd-p696h              1/1     Running   0          4m32s
flux-memcached-8647794c5f-gdw2p   1/1     Running   0          4m32s
helm-operator-66d5477cb7-xwgzh    1/1     Running   0          4m1s
```

## Why I couldn't use helm 2to3 plugin

the helm team created a plugin which is supposed to help you migrate your released from helm2 to helm3. **BUT** if you are using fluxcd to manage your cluster, this tool is helpless because your changes will conflict with the chart source, and overwritten by the flux operator on the next sync cycle. :cry:

## <a id="move">Upgrading Steps:</a>

### Step 1: switch to multiversioned helm-operator

Figure out in which namespace your flux helm-operator is running, and its version:
```console
~ $ helm ls flux-helm-op --output yaml
Next: ""
Releases:
- AppVersion: 1.0.0-rc7
  Chart: helm-operator-0.5.0
  Name: flux-helm-op
  Namespace: flux
  Revision: 1
  Status: DEPLOYED
  Updated: Tue Jan 21 12:44:31 2020             Tue Jan 21 12:44:31 2020        DEPLOYED        helm-operator-0.5.0     1.0.0-rc7       flux            
```

as you can see, my helm operator is deployed into `flux` namespace. I should take care of 2 things:
1. The image version must be `1.0.0-rc5` or higher, otherwise I should bump it up.
2. I will add "HELM_VERSION=v2,v3" environment variable. 

```console
~ $ echo "
image:
  tag: "1.0.0-rc9"
extraEnvs: 
- name: HELM_VERSION
  value: "v2,v3"
" >> flux-helm3-values.yaml
```
```console
 ~ $ helm upgrade helm-operator fluxcd/helm-operator --wait \
--namespace flux -f flux-helm3-values.yaml
```

As you can see in helm-operator [source code](https://github.com/fluxcd/helm-operator/blob/9951e409d5f8e14eee0139194b85290f42939247/cmd/helm-operator/main.go#L213-L234), once the change is done, both v2 and v3 helm versions will be respected.


### Step 2: Update the repo: 

**Bump the Chart API Version:**

When using fluxcd, your cluster content is version-controller in a dedicated git repository, and you keep your charts under the  `/charts` path. cd into `charts/example`. In the `Chart.yaml` file, change `apiVersion` from `v1` to `v2`.

> (Theres are a lot more differences between the versions, which you can view if you run `helm create helm2-chart` and `helm3 create helm3-chart` and compare the generated directories. In this post I only cover the minimal changes needed to get this done, for my super busy readers :fire:) 

**Bump the HelmRelease Helm version:**

go to your helm release (`/releases/example.yaml` in the repo). Set the `spec.helmVersion` to `v3`. (the default value is `v2`).

### Step 3: Make the swap

commit and push the changes. Meanwhile watch the helm-operator pod logs. you will see something like:

```console
ts=2020-03-12T08:55:24.281650359Z caller=release.go:216
component=release release=example
targetNamespace=example resource=example:helmrelease/example
helmVersion=v3 error="Helm release failed"
revision=ccdf7cc0e4b57c624019d43fb2f066b8afd24ce9
err="failed to upgrade chart for release [example]:
rendered manifests contain a resource
that already exists. Unable to continue with install:
existing resource conflict:
kind: PersistentVolumeClaim, namespace: example, name: example-efs-pvc"
```
#### Don't Panic Yet!
this error message tell us about **resources conflicts**. This is happening because **By default, Flux operator does not delete resources**. It is creating & updating resources but deletions must be done manually. So flux is now trying to deploy your helm3 new release resources but bumps into the helm2 old release pieces. So once you see those errors you know you can safely delete the helm2 release:

```console
helm delete --purge example
```

#### Congrats!
You have cleared the way for the new helm3 release, and now the logs will show:
```conosle
ts=2020-03-12T09:02:22.244932236Z caller=helm.go:69
component=helm version=v3 info="creating 7 resource(s)"
targetNamespace=example release=example
ts=2020-03-12T09:02:22.374214468Z caller=release.go:266
component=release release=example targetNamespace=example
resource=example:helmrelease/example helmVersion=v3
info="Helm release sync succeeded" revision=d058e851afb648eddd4d5ec29be0c163d16d0c85
```

From now on, your release can be viewed using `helm3`. ([In this post I cover helm3 client installation steps](/blog/kubernetes/helm/2020/03/11/the-server-was-unable-to-return-a-response-in-the-time-allotted,-but-may-still-be-processing-the-request-(get-configmaps)-OR-Why-Its-Time-to-Move-to-Helm3.html)):

```console
~ $ helm3 ls
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART           APP VERSION
example         example         1               2020-03-12 09:02:21.66010842 +0000 UTC  deployed        example-0.1.0   1.0        
```

Gently roll this update across your cluster. If you are careful and patient like me <span style='font-size:40px;'>&#129496;&#127995;&#8205;&#9792;&#65039;</span>, You will not experience any downtime (The helm2 release pods will terminate and the helm3 release pods will start on the same time).

## Happy Helm3ing!
