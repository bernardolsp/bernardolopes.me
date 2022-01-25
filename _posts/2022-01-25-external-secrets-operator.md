---
layout: post
title: External Secrets Operator
categories: [Kubernetes]
tags: [guide]
date: 2022-01-25 08:00 -0300
---

### **External Secrets Operator**
External Secrets Operator is a tool to improve the security of your kubernetes secrets, by using a secure secret manager (i.e, AWS, Vault, etc) and automatically refreshing the data in your cluster.

#### **According to their own documentation**:   
> The goal of External Secrets Operator is to synchronize secrets from external APIs into Kubernetes. ESO is a collection of custom API resources - ExternalSecret, SecretStore and ClusterSecretStore that provide a user-friendly abstraction for the external API that stores and manages the lifecycle of the secrets for you.

#### **Basic components**:

- ExternalSecret:  
  This CRD main function is to **provide which secret we want to grab from the remote secret manager**. If it all behaves correctly, it'll create an [Opaque Secret](https://kubernetes.io/docs/concepts/configuration/secret/) in the cluster with the requested data.

- SecretStore:  
  This CRD main function is to **create a connection with the remote secret manager**. It also refreshes the state of such secret and update the generated Opaque Secret

- ClusterSecretStore:  
  This CRD main function is **the same as the SecretStore**. However, this should be used when **multi-namespaces are required**. That is, you have a cluster with several applications segregated by namespaces and want to have only one secret store.

#### **Hands on**:

We'll create a [**Secret Store**](https://external-secrets.io/api-secretstore/), which can only be interacted in the given namespace. If you happen to require a multi-namespace secret store (i.e, multiple namespaces using the same Secret Store), one can use the **Cluster Secret Store**

First of all, let's setup the CRDs using **Helm**. Here's my current kubernetes cluster:

```
➜  bernardolopes.me git:(main) ✗ k get all --all-namespaces
NAMESPACE     NAME                                           READY   STATUS    RESTARTS         AGE
kube-system   pod/calico-kube-controllers-7cd8958654-rg4mb   1/1     Running   12 (3d10h ago)   5d
kube-system   pod/calico-node-ddmz9                          1/1     Running   12 (3d10h ago)   5d
ingress       pod/nginx-ingress-microk8s-controller-kbrbj    1/1     Running   0                2d22h

NAMESPACE   NAME                 TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
default     service/kubernetes   ClusterIP   10.152.183.1   <none>        443/TCP   5d

NAMESPACE     NAME                                               DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR            AGE
kube-system   daemonset.apps/calico-node                         1         1         1       1            1           kubernetes.io/os=linux   5d
ingress       daemonset.apps/nginx-ingress-microk8s-controller   1         1         1       1            1           <none>                   2d22h

NAMESPACE     NAME                                      READY   UP-TO-DATE   AVAILABLE   AGE
kube-system   deployment.apps/calico-kube-controllers   1/1     1            1           5d
system        deployment.apps/controller-manager        0/1     0            0           2d21h

NAMESPACE     NAME                                                 DESIRED   CURRENT   READY   AGE
kube-system   replicaset.apps/calico-kube-controllers-6966456d6b   0         0         0       5d
kube-system   replicaset.apps/calico-kube-controllers-7cd8958654   1         1         1       5d
system        replicaset.apps/controller-manager-7bfc98cf47        1         0         0       2d21h
```

That's a bog standard **microk8s** cluster with Ingress enabled.

To install it, follow [their documentation](https://external-secrets.io/guides-getting-started/). The result should be:

```
NAME: external-secrets
LAST DEPLOYED: Tue Jan 25 09:45:59 2022
NAMESPACE: external-secrets
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
external-secrets has been deployed successfully!

In order to begin using ExternalSecrets, you will need to set up a SecretStore
or ClusterSecretStore resource (for example, by creating a 'vault' SecretStore).

More information on the different types of SecretStores and how to configure them
can be found in our Github: https://github.com/external-secrets/external-secrets
```

We're almost there!

I have set up a small [**vault**](https://www.vaultproject.io/) application for this use case. You can have a Vault project running in under 10 minutes.

If you choose to use this provider, consider creating a set of secrets such as the [**official documentation**](https://external-secrets.io/provider-hashicorp-vault/)

Let's create a **SecretStore** now. Here's my **secret-store.yaml**:

`➜ cat secret-store.yaml `
```yaml
apiVersion: external-secrets.io/v1alpha1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://10.123.228.81:8200"
      path: "secret"
      version: "v1"
      auth:
        tokenSecretRef:
          name: "vault-token"
          namespace: "default"
          key: "token"
---
apiVersion: v1
kind: Secret
metadata:
  name: vault-token
data:
  token: cy5zUUtPTHVoUG84YzhXdVZPNzRBaE1WWjIK # the token encoded in base64
```

Save this to a .yaml file, edit to match your server and token configuration. Then, run a:

`kubectl apply -f secret-store.yaml -n external-secrets`

Now, let's create a **ExternalSecret**:

```yaml
apiVersion: external-secrets.io/v1alpha1
kind: ExternalSecret
metadata:
  name: vault-example
spec:
  refreshInterval: "15s"
  secretStoreRef:
    name: vault-backend
    kind: SecretStore
  target:
    name: example-sync
  dataFrom:
    - key: foo
```

After applying all of them, we should have:

```
➜  external-secrets k get externalsecrets                  
NAME                                               STORE                REFRESH INTERVAL   STATUS
externalsecret.external-secrets.io/vault-example   vault-backend        15s                SecretSynced

NAME                                                 AGE
secretstore.external-secrets.io/vault-backend        25m
```

We can grab the secret data by **either attaching it to a pod/deploy/svc** or by running:

`➜  external-secrets k get secret example-sync -ojson`

and here we go:

```json
{
    "apiVersion": "v1",
    "data": {
        "my-value": "czNjcjN0"
    },
    [...]
```

Per default, this secret is encoded in base64, so we can run **echo czNjcjN0 | base64 -d**:  

`➜  external-secrets echo czNjcjN0 | base64 -d`
```
s3cr3t% 
```

Thanks for tuning in so far! :-) 