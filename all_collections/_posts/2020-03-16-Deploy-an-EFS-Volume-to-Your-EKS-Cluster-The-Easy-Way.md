---
layout: post
title: Deploy an EFS Volume to Your EKS Cluster The Easy Way
# slug: cvbnjklbjm,.
lang: en
categories:
    - Kubernetes
    - helm
    - AWS
    - EKS
    - EFS
    - Persistent Volume
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

## Background <a href="#skipto">skip to practical</a>
Introduced on April 2015, EFS is an AWS abstraction over NFS - network file system, allowing you to share a server volume between multiple machines on the network. In this tutorial I will show you my painless way to deploy such EFS as a Kubernetes Persistent Volume.

### How the EFS provisioner works?

The EFS provisioner is a container, in charge of fulfilling the cluster PersistentVolumeClaims with real EFS PersistentVolumes, by making calls to AWS EFS API. It will be passed a ConfigMap, containing the EFS filesystem ID, the AWS region and a StorageClass name. The provisioner is also responsible of creating the StorageClass itself:

<img src="{{"/assets/designer/storage.png" | relative_url }}">


## <a id="skipto">Practical Steps:</a>

> :bulb: Over this tutorial I will name my storage class `example-efs`. You may customize it across the applied resources.

### 1. Creating the EFS

go to [aws efs dashboard](https://console.aws.amazon.com/efs/home), and click **Create file system**. Select your cluster VPC, choose subnets and **assign a security group that allows traffic from your EKS worker nodes**. leave anything else in its place. Once the EFS is created, copy its ID:

<img src="{{"/assets/img/efs-id-point.png" | relative_url }}">

### 2. Installing efs-provisioner helm chart

```console
helm install stable/efs-provisioner \
--set efsProvisioner.efsFileSystemId=YOUR_FS_ID \ 
--set efsProvisioner.awsRegion=YOUR_REGION \
--set efsProvisioner.storageClass.name="example-efs" \
--name example-efs-provisioner
```

this chart is stable and I like it, but it lacks a few core components, which we will apply now:

### 3. Creating the PVC

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: example-efs-pvc
  namespace: legacy
  annotations:
    volume.beta.kubernetes.io/storage-class: example-efs
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: "example-efs"
```
### 4. Creating the missing rbac role & role-binding:
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-efs-provisioner
rules:
  - apiGroups: [""]
    resources: ["endpoints"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: example-efs-provisioner
subjects:
  - kind: ServiceAccount
    name: example-efs-provisioner
    # replace with namespace where provisioner is deployed
    namespace: legacy
roleRef:
  kind: Role
  name: example-efs-provisioner
  apiGroup: rbac.authorization.k8s.io
```
once all this stuff is deployed, its time to mount and use the provisioned EFS:
### 5. Mounting over deployments:
`deployment.yaml`
```yaml
spec:
# ....
  template:
  # ...
    spec:
    # ...
      volumes:
      # declare the volume:
      - name: example-volume
        persistentVolumeClaim:
          claimName: example-efs-pvc

      # mount the volume over the container:
      containers:
        - name: example-container
        # ...
          volumeMounts:
          - name: example-volume
            mountPath: /tmp/volume-path
            subPath: volume-path
            readOnly: false
```

Once the deployment is created, make sure the `pvc` status is `Bound`:
```console
~ $  kubectl get pvc            
NAME                  STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
example-efs-pvc       Bound    pvc-138778e3-5898-11ea-99fe-06af61d5b152   50Gi       RWX            example-efs      24d
```

That means your PVC was successfully mounted over your application pods, and you are ready to rock.
