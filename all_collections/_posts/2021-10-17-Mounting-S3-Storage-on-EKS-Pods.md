---
layout: post
title: Mounting S3 Storage on EKS Pods 
lang: en
categories:
    - Kubernetes
    - EKS
    - AWS
    - S3
    - Storage
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---
Ever wondered whether its possible to mount S3 as storage for pods? Well it is, thanks to [datashim.io CSI driver](https://github.com/datashim-io/datashim) - A Kubernetes Framework to provide easy access to S3 and NFS Datasets within pods.
# Who needs this feature and who doesn't
Since S3 doesn't allow file editing and the communication is over HTTP - this storage is not a good replacement for disk storage if your system requires high I/O or does file edits - Databases, servers cache etc. This solution suits images / file storage, as of S3 itself.
# My Usecase
I am migrating legacy apps from bare metal servers onto kubernetes on AWS. Now the servers are storing images on a shared NFS. The legacy pods I create must share storage and S3 offers the cheapest storage solution for my images. So I mount S3 path over my EKS pods using the CSI driver and make them believe they still share that NFS, while the datashim operator converts the I/O communication to HTTP requests against S3.
# Installation 

## Create the S3 target bucket and User + Group
As for now, the driver doesn't support IAM role so a user must be created. I use terraform to create my AWS resources but feel free to do it ant other way:
```terraform
locals {
  account_num = "123456789"
  common_tags = {
    Env                = "dev"
    App                = "s3-storage-test"
    Author             = "devops"
  }
}
resource "aws_iam_user" "cluster-user" {
  name     = "cluster-user"
  path     = "/"
  tags = local.common_tags
}
module "s3-storage-users" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-group-with-policies"
  version = "~> 3.0"
  name = "s3-storage-users"

  group_users = ["cluster-user"]

  custom_group_policy_arns = [
    "arn:aws:iam::${local.account_num}:policy/storageTestS3FullAccess",
  ]
  tags = local.common_tags
}
resource "aws_s3_bucket" "storage-test" {
    bucket = "storage-test"
    acl    = "private"
    tags = local.common_tags
}
resource "aws_iam_policy" "storageTestS3FullAccess" {
    name        = "storageTestS3FullAccess"
    path        = "/"
    description = "Allow full access to storage-test s3"
    tags = local.common_tags
    policy      = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": [
        "s3:PutAccountPublicAccessBlock",
        "s3:GetAccountPublicAccessBlock",
        "s3:ListAllMyBuckets",
        "s3:ListJobs",
        "s3:CreateJob",
        "s3:HeadBucket",
        "s3:ListBucket"
      ],
      "Resource": "*"
    },
    {
      "Sid": "VisualEditor1",
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::storage-test",
        "arn:aws:s3:::storage-test/*"
      ]
    }
  ]
}
POLICY
}

```

## In-Cluster Operations:
- Deploy the CRDs to your cluster:
```console
kubectl apply -f https://raw.githubusercontent.com/datashim-io/datashim/master/release-tools/manifests/dlf.yaml
```
- Create secret for the user creds:
```yaml
apiVersion: v1
kind: Secret
metadata:
    name: cluster-user-creds
    namespace: dlf
data:
    accessKeyID: xxxxxx
    secretAccessKey: xxxxxx
```
- Create the dataset:
```yaml
apiVersion: com.ie.ibm.hpsys/v1alpha1
kind: Dataset
metadata:
  name: s3-dataset
  namespace: default
spec:
  local:
    type: "COS"
    secret-name: "cluster-user-creds" #see s3-secrets.yaml for an example
    secret-namespace: "dlf" #optional if the secret is in the same ns as dataset
    endpoint: "https://s3.eu-west-1.amazonaws.com/test-storage"
    bucket: "test-storage"
    readonly: "false" #OPTIONAL, default is false  
    region: "eu-west-1" #OPTIONAL
```
- After created you will see the pvc:
```bash
~ k get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS     AGE
s3-dataset       Bound    pvc-818acd89-bbd9-45ea-8483-f8ba9943f623   9314Gi     RWX            csi-s3           63m
```
- Mount the pvc to a pod:
  
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: "test-mount"
  namespace: monitoring
spec:
  containers:
  - name: test-mount
    image: nginx:alpine
    volumeMounts:
    - mountPath: /tmp
      name: s3-storage
      readOnly: false
      subPath: test_mount
  volumes:
  - name: s3-storage
    persistentVolumeClaim:
        claimName: s3-dataset 
```
- Ensure the pod is mounted:
```bash
~ k exec -ti test-mount ash
kubectl exec [POD] [COMMAND] is DEPRECATED and will be removed in a future version. Use kubectl kubectl exec [POD] -- [COMMAND] instead.
/ # df -h
Filesystem                Size      Used Available Use% Mounted on
overlay                 100.0G     11.4G     88.6G  11% /
tmpfs                    64.0M         0     64.0M   0% /dev
tmpfs                    14.9G         0     14.9G   0% /sys/fs/cgroup
s3-storage               1.0P         0      1.0P   0% /tmp
.....
```
Its that simple... Good Luck & Follow for more :heart:

## 3.2023 Update
Hi again :wave:
A few awesome readers reached out regarding this post lately, asking about features that might or might not be included in the datashim operator. Since writing this, I also got to work with another tool with a very active community called [MinIO](https://github.com/minio/minio). They also have an operator for mounting S3 storage over k8s, which you might want to [check out](https://min.io/docs/minio/kubernetes/upstream/index.html)