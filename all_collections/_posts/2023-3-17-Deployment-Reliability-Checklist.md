---
layout: post
title: Deployment Reliability Checklist
date: 2023-03-17
categories: ["Kubernetes","SRE","deployment"]
permalink: /:slug

---

Do you get health-alerts during version releases? clients keep reporting slippery 503 errors? During my time at multiple companies, working on different apps, I gathered a little list you can go through to make sure you are using the full power of kubernetes to increase app reliability:

## 1. Spread the Love!
K8s v1.18 comes with the expressive notion of pods `topologySpreadConstraints`, Which is a great way to improve your app reliability by nicely asking the scheduler to spread pods across nodes, node-groups or availability zones.

Spreading across **nodes** protects you in the event of multiple spots going down at once, taking out a massive percent of your pods, while not enough new ones have emerged into ready state.

Spreading across **node-groups** is useful during peak times - when you have to scale out but instances of certain types/ bid price/ other group config may not be available.

Spreading across **availability-zone** is a great high-availability safeguard, to tolerate the event of an entire availability zone going down. Of cause, to achieve a good level of HA, make sure the multi-AZ approach propagates upstream dependencies of your app like Databases, or alternatively - your app can linger in degraded mode until its dependencies are reachable again.

A demonstration of topology constrains covering the above, on AWS (other clouds requires different topology keys but the idea is the same):
```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: super-reliable-app
spec:
  selector:
    matchLabels:
      app: super-reliable-app
  replicas: 5
  template: 
    metadata:
      labels:
        app: super-reliable-app
    spec:
      topologySpreadConstraints:
        # The node label to base the topology spread upon:
      - topologyKey: kubernetes.io/hostname | eks.amazonaws.com/nodegroup | failure-domain.beta.kubernetes.io/zone
        # Spread the pods across at least 3 nodes/groups/AZs.
        # Beta since v1.25:
        minDomains: 3
        # Don't break scheduling if you can't spread:
        whenUnsatisfiable: ScheduleAnyway
        # we have 5 replicas to spread across 3 domains,
        # so we allow a diff of 1:
        maxSkew: 1
        # Like other affinity rules, topologySpreadConstraints
        # is defined at the pod level, hence we must let it know
        # how other pods of the deployment would look like:
        labelSelector: 
          matchExpressions:
          - key: app
            operator: In
            values:
            - super-reliable-app
      containers:
      ...
```

## 2. Fine-tune your updates
Ask quetions about the app update process. How long it takes for typical request to complete? (might want to increase `TerminationGracePeriod`). How long it takes for the app to be up? (tune `MinReadySeconds` to make sure during an update, old pods will not be terminated before the new pods are ready to serve) more details in [deployment-controller-revealed](/kubernetes/sre/deployment/2023/03/15/deployment-controller-revealed.html)

## 3. Handle spots termination
When running on cloud spot instances, you can save lots of money, but should also take precausions to avoid app failure during nodes termination. EKS **managed** node groups automatically take care of moving your pods during the ITN heads-up period (usually 2 minutes before the node goes down).

If you can't use managed node groups (I get that, they still lack some features :wink:) consider installing [aws-node-termination-handler](https://github.com/aws/aws-node-termination-handler) which takes care of that operation.

## 4. Don't trade reliability for autoscaling
Autoscalers have 2 jobs - 1. Add more nodes when your demand grows, 2. Compact pods onto less nodes when possible, AKA **consolidation**/ Cutting the fat. Autoscaling is a great way to save on your compute budget, instead of pre-provisioning up to your peak usage.
Obviously, autoscaling events are natural risks to your app reliability, but fear not - here are some tips to safeguard from tossing the baby with the bathwater:

**Configure Pod Disruption Budget:**
A native resource that defines the minimal amount of pods you need running at all time. Though cloud providers are not obliged to this notion when terminating your spots, autoscalers do respect PDB constrains, keeping the `minAvailable` num of pods untouched during the consolidation. 
```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: super-reliable-app-pdb
spec:
  minAvailable: 3 # or "60%"
  selector:
    matchLabels:
      app: super-reliable-app
```

**Autoscale with headroom:**
You might find yourself in situations with enterprise clients impatiently waiting for their workload to be up, where you just can't afford that 3 minutes wait for the autoscaler to initialize a new node. Thank god, you can leverage k8s `priorityClass` with a dummy deployment for *Overprovisioning*, and keep a hot pool of resources to address urgent demand, no matter which autoscaler you are using. [Show me how!!](https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/FAQ.md#how-can-i-configure-overprovisioning-with-cluster-autoscaler)

That would be all for today, let me know if you have questions, or unmentioned reliability pro-tips. I would love to know all about them. :blue_heart: