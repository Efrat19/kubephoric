---
layout: post
title: Kubernetes Secrets Management 
lang: en
categories:
    - Kubernetes
    - CRD
    - AWS
    - EKS
# tags:
#     - hoge
#     - foo
permalink: /:slug
image:
 path: /assets/designer/tinified_meta.png
 width: 1200
 height: 630
---

In this post I will go through the traditional secrets management and than cover some sick new CRDs I work with, designed for GitOps cluster management.

## 1. kubernetes-native secrets management

Secrets are core part of kubernetes, Offering you the ultimate secured way to store your passwords. Once "kubectl applied", kubernetes will be in charge for encrypting / decrypting them.

### Pros: 
- core component - no installation is needed
- simplified process and less resources to manage
- easy decryption with kubectl
  
### Con: 

- once you try to manage your cluster with GitOps, meaning version-control everything, you are totally screwed, because for security reasons, your secrets should never be version-controlled, and so you will be doomed to waste yuor time chasing your secrets across the cluster, duplicating them between namespaces and writing ugly little bash scripts in case you have to edit a secret which already had been copied to 10 namespaces. 
This is really annoying, but over time, smart people came up with various solutions. 2 of them I had tried myself and am ready to share my expirience:

## 2. Bitnami-Labs **[Sealed secrets](https://github.com/bitnami-labs/sealed-secrets)**

with bitnami's SealedSecret controller, you will be able to encrypt your secrets and once encrypted, they can be safely stroed in your codebase, since only you and the sealed-secret controller on the cluster can decrypt them. The controller is responsible for transforming the encrypted SealedSecret resource into a regular kubernetes secret. 

### Pros:

- kubernetes-oriented solution - Designed for k8s, the controller uses configMaps to store the encryption keys and resources can be managed with kubectl
- quick installation with helm / kustomize
- offers key-rotation at a scheduled time to improve security
- offers an easy way to convert existing secret to SealedSecrets managed secrets

### Cons

- complicates the process of creating secrets
- currently doesn't support external keys management integration (e.g. KMS).
- Potential secret loss (like when a key configMap is accidentally deleted)

## 3. GoDaddy **[External Secrets](https://github.com/godaddy/kubernetes-external-secrets)**

As the name tells you, ExternalSecret uses an external secret manager to store the secrets.
supported backends are AWS Secrets Manager, AWS System Manager, Hashicorp Vault and Azure Key Vault.

For example - you create a secret in aws SecretsManager:

`aws secretsmanager create-secret --name context/of/mysecret --secret-string "thisIStheDATABASEpassword"`

once the secret is created on the secretsmanager, install the official [helm chart](https://github.com/godaddy/kubernetes-external-secrets/blob/master/charts/kubernetes-external-secrets/README.md) and then deploy to kubernetes a *reference* to the secret you created:

```yaml
apiVersion: kubernetes-client.io/v1
kind: ExternalSecret
metadata:
  annotations:
  name: mysecret
  namespace: legacy
spec:
  backendType: secretsManager
  data:
  - key: context/of/mysecret
    name: mysecret
```
once such an ExternalSecret is deployed, the ExternalSecrets controller will be in charge of creating a regular secret out of it.

### Pros:

- A variety of backends integrations 
- offers a sync cycle - the backend is being polled at a configurable time interval to search for new changes and apply them to the existing secrets on kubernetes
- single source if truth - you can have multiple ExternalSecrets deployed to different namespaces, and referring to the same single secret on the backend. No more cross-namespace secrets editing yay

### Cons

- efficient only if you already use AWS or Azure, otherwise the pain of user management doesn't pay off
- Pricing - external service as has its own costs you should consider.

Right now I use ExternalSecrets and am very happy with the choice. I believe you should try all of this solutions and see what best suits you. 


<!-- https://github.com/mozilla/sops -->

gluck!
