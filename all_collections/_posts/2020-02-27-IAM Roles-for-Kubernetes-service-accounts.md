---
layout: post
title: IAM Roles for Kubernetes service-accounts
lang: en
categories:
    - AWS
    - EKS
    - Kubernetes
    - IAM
    - CloudWatch
    - Fluentd
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---

On this post I will show you how to give IAM permissions to a k8s service account, right from the CLI with a few simple commands. As an example, we will create Fluentd and Cloud-watch DaemonSets which will collect logs across the cluster and stream them to AWS CloudWatch logs panel. The service-accounts will be granted k8s RBAC permissions, but we will also create for them an IAM role, and link it via k8s annotation.

## TL;DR:

`curl https://raw.githubusercontent.com/Efrat19/blog/master/snippets/eks-iam-1.sh | sh` :crossed_fingers:

## Exporting Required Vars:

kubectl must point to the target cluster, and aws cli should refer to the right profile. Now lets export the vars we will need on the process: 

<!-- <script src="https://gist.github.com/Efrat19/324b8920697d3b2614be73cdd9a91f11.js"></script> -->

```bash
export CLUSTER="$(kubectl config view -ojson | jq -r --arg CTX $(kubectl config current-context) '.contexts | .[] | select(.name == $CTX) | .context.cluster | split("/") | .[length-1]')"
export REGION=$(aws configure get region)
export OIDC=$(aws eks describe-cluster --name ${CLUSTER} --query cluster.identity.oidc | jq -r '.issuer | split("/") | .[length-1]')
export ACCOUNT=$(aws sts get-caller-identity --query Account --output text)

echo "cluster $CLUSTER in region $REGION and account number ${ACCOUNT} has OIDC token: ${OIDC}"
```


## Applying Kubernetes Resources:

took it from [this](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Container-Insights-setup-logs.html) tutorial, but it still needed some tweaks to function properly:
`curl https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml | sed "s/{{cluster_name}}/${CLUSTER}/;s/{{region_name}}/${REGION}/;s/name: REGION/name: AWS_REGION/;" | kubectl apply -f -`

## Creating the role and the trust relationship:

> **A note about the OIDC protocol:** OIDC or OpenID Connect, is a protocol that extends the existing OAuth 2.0 protocol. The OIDC flow starts with a user requesting a JSON Web Token from an identity provider that contains an appropriately scoped list of attributes about the user. The contents includes attributes such as an email address or name, a header containing extra information about the token itself, e.g. the signature algorithm, and finally the signature of the token that has been signed by the identity provider. This signature is used by the resource server to verify the the token contents using the Certificate Authority presented by the identity provider.
```console
~ $ aws eks describe-cluster --name $CLUSTER --query cluster.identity.oidc | jq
{
  "issuer": "https://oidc.eks.eu-west-1.amazonaws.com/id/THIS-IS-THE-OIDC-TOKEN"
}
```

creating the trust relationships file - this policy allows our EKS cluster to assume the role we will create:
```bash
echo "{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT}:oidc-provider/oidc.eks.${REGION}.amazonaws.com/id/${OIDC}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "oidc.eks.${REGION}.amazonaws.com/id/${OIDC}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}" >> trust_relationshsips.json
```
now the role itself:

`aws iam create-role --role-name k8s-cloudwatch-agent --assume-role-policy-document file://trust_relationshsips.json`

We will attach a cloud-watch agent policy:

`aws iam attach-role-policy --role-name k8s-cloudwatch-agent --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy`

## Annotating the serviceaccounts 

This is a custom eks annotation to mark this service accounts as allowed to assume the rolw we created:

`kubectl annotate -n amazon-cloudwatch sa cloudwatch-agent fluentd "eks.amazonaws.com/role-arn=arn:aws:iam::${ACCOUNT}:role/k8s-cloudwatch-agent"`

and now lets restart the pods so the change will take over:
`kubectl get po -n amazon-cloudwatch -oname | xargs kubectl delete`

## Done! head over to cloudwatch for your cluster insights.

look for log groups starting with `/aws/containerinsights/your_cluster/`.


