---
layout: post
title: "AWS Managed Airflow + EKS Scheduling, Part 2: Scheduling DAGs on EKS"
lang: en
categories:
    - Kubernetes
    - EKS
    - AWS
    - MWAA
    - Airflow
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---
If you completed [**part 1**](https://efrat19.github.io/kubephoric/aws-managed-airflow-eks-scheduling-part-1-creating-managed-airflow-instance) on this guide and created the Managed Airflow Environment on AWS, you are ready for the next level - 
# Part 2 - Enabling EKS scheduling

In Airflow we often use `KubernetesPodOperator` DAG class. Airflow contacts k8s API-server and asks it to spawn a pod to perform the task. In this part of the tutorial you will connect MWAA to an EKS cluster on the same VPC, and schedule an example pod on top of it.

## Pre-Requests:
You need an EKS cluster. if you are familiar with terraform, you can do it in one click using [my EKS module](https://github.com/Efrat19/terraform-eks-with-gitops)

## IAM Considerations:
We will add another permission to the MWAA execution role from part 1.


```terraform
locals {
  region = "xx-xxxx-x"
  account_id = "xxxxxxxxxx"
  eks_cluster_name = "xxxxxxxxxx"
}

resource "aws_iam_policy" "amazon_mwaa_eks_scheduling_policy" {
  name   = "amazon_mwaa_eks_scheduling_policy"
  path   = "/"
  policy = <<POLICY
{
    "Version": "2012-10-17",
    "Statement": [
               {
            "Effect": "Allow",
            "Action": [
                "eks:DescribeCluster"
            ],
            "Resource": "arn:aws:eks:${local.region}:${local.account_id}:cluster/${local.eks_cluster_name}"
        }     
    ]
}
POLICY
}
```
add the `aws_iam_policy.amazon_mwaa_eks_scheduling_policy.arn` to the `managed_policy_arns` list in the execution role:
```terraform
resource "aws_iam_role" "mwaa_role" {
...
  managed_policy_arns = [
    aws_iam_policy.amazon_mwaa_policy.arn,
    # Add the new policy:
    aws_iam_policy.amazon_mwaa_eks_scheduling_policy.arn,
  ]
}
```
Complete TF source can be found [here](https://github.com/Efrat19/mwaa_source_example/blob/main/mwaa.tf)

## EKS Considerations:
MWAA is outside the cluster scope. In order for managed Airflow to reach out for the API service, it will need an entry in the `aws-auth` configmap in the `kube-system` namespace. we are basically registering the MWAA execution role as a k8s user:
```yaml
    userarn  = "arn:aws:iam::<account_id>:role/airflow-mwaa-role"
    username = "mwaa-service"
    groups   = ["system:masters"]
```

also you will need to `kubectl apply` the RBAC Role and RoleBinding:
```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mwaa-role
  namespace: default
rules:
  - apiGroups:
      - ""
      - "apps"
      - "batch"
      - "extensions"
    resources:      
      - "jobs"
      - "pods"
      - "pods/attach"
      - "pods/exec"
      - "pods/log"
      - "pods/portforward"
      - "secrets"
      - "services"
    verbs:
      - "create"
      - "delete"
      - "describe"
      - "get"
      - "list"
      - "patch"
      - "update"
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: mwaa-role-binding
  namespace: default
subjects:
- kind: User
  name: mwaa-service
roleRef:
  kind: Role
  name: mwaa-role
  apiGroup: rbac.authorization.k8s.io
```

## Final Step - Kubeconfig:
Now the last task is to give MWAA a kubeconfig to use. generate one with:
```bash
aws eks update-kubeconfig \
--region your-region \
--kubeconfig ./kube_config.yaml \
--name mwaa-eks \
--alias aws
```
and upload the generated file to the source bucket, to the same path of the dags:
```bash
aws s3 cp kube_config.yaml s3://my-mwaa-source/mwaa_source_example/dags
```
You will be adding this config file path to DAGs using `KubernetesPodOperator`. [view example](https://github.com/Efrat19/mwaa_source_example/blob/4239387d5a7e1d4722004e2faf78e6d87d167227/dags/eks_scheduling.py#L43)

## All Done!
Head over to MWAA Airflow UI, enable and trigger the `kubernetes_pod_example` DAG. Once the DAG is started, inspect the created pod:

```bash
kubectl get po -n default -w
```
you will see the pod being spawned by airflow.

<img src="{{"/assets/img/scheduled-pod.png" | relative_url }}">

it didn't work? [troubleshoot](https://docs.aws.amazon.com/mwaa/latest/userguide/mwaa-eks-example.html)