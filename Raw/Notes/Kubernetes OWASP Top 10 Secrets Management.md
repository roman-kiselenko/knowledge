---
title: Kubernetes OWASP Top 10 Secrets Management
source: https://itnext.io/kubernetes-owasp-top-10-secrets-management-c996faa87b47
clipped: 2023-09-04
published: 
category: security
tags:
  - k8s
  - secrets
read: false
---

In this latest entry to my rundown of the [Kubernetes OWASP Top 10](https://owasp.org/www-project-kubernetes-top-ten/), I will be focussing on Secrets Management. Arguably, secrets management, not only in the context of Kubernetes but across the industry as whole, is one of the most important aspects of security to get right. Whether it be API keys to prove identity or certificates and keys to ensure Encryption in Transit or at Rest, secrets if not managed properly, can lead to catastrophic consequences for the entire Confidential, Integrity and Availability of any given service.

Before I delve into the technical nuances of secrets in Kubernetes, there are some fundamentals in secrets management that should help keep secrets leakage risk as low as possible.

-   Secrets should allow for Just Enough Access to the system related to. This ensures there are no “Keys to the Kingdom” scenarios where a single secret expose the entire estate.
-   Secrets should be limited in scope.
-   Strong RBAC controls on Secrets Stores.
-   Segregation of duty, for example someone that manages a Key Store control plane should not have access to the data plane (i.e. the secrets themselves) with the same set of credentials
-   PKI (**Private** Key Infrastructure) should be exactly that, private. Limit any movement and sharing of private keys.

## Kubernetes Secrets

Kubernetes has a native Secret type, this allows for identification and separation of configurations that should be secret. It is simply a type, separated for the purpose of identification and requires that data stored in them are base64 encoded. Notice they are encoded **NOT** encrypted.

![[Raw/Notes/Raw/Media/Resources/0ab0dee0a91692d9506fc5a4af04c587_MD5.png]]

Vew the etcd Kubernetes data store, by default these are not encrypted.

Kubernetes needs to be configured to encrypt all secrets. This can be achieved by utilising an *EncryptionConfiguration* Kubernetes object that the API can utilise while communicating with the etcd datastore.

Kubernetes EncryptionConfiguration example

Kubernetes supports multiple encryption providers which varies in strengths and speeds depending on requirements.

Encryption Providers

-   Identity — Default provider, does not encrypt objects at rest, but restricted to allowed users (i.e. not anonymous, public access)
-   secretbox — A newer encryption standard utilising XSalsa20 and Poly1305
-   aesgcm — AES-GCM —Recommended key rotation every 200K writes
-   aescbc — AES-CBC — No longer recommended due to known weaknesses.
-   KMS — External Key Management Service, has the added benefit of the ability to utilise a HSM.  
    — [Azure KeyVault](https://learn.microsoft.com/en-us/azure/aks/use-kms-etcd-encryption)  
    — [AWS KMS](https://docs.aws.amazon.com/eks/latest/userguide/enable-kms.html)  
    — [GCP Cloud KMS](https://cloud.google.com/kubernetes-engine/docs/how-to/encrypting-secrets#existing-cluster)

Once an Encryption Configuration is created, the Kubernetes API server needs to be configured to use it.

-   Add the encryption-provider-config switch to the API server.
-   Create a Volume to containing the EncryptionConfiguration manifest
-   Mount the volume in the API pod ensuring it is mounted read-only in the path provided to the encryption-provider-config switch passed to the API.
-   Restart the API server.

**Important: Existing secrets are not encrypted, only new secrets after the config has been added. To be able to encrypt secrets prior to encryption they must be deleted and recreated.**

There are several methods to be able to get secrets into Kubernetes, methods with the least human interaction is preferred, reducing the risk of social engineering or leaks.

-   Inputting secrets via CLI.
-   Using CI/CD pipelines.
-   Kubernetes Secrets Store Container Storage Interface (CSI)

## Manual Secret Creation

Use of the Kubernetes REST API, ordinarily through the kubectl CLI interface is a common way of getting secrets into Kubernetes.

There are 3 types of secrets you can create using kubectl

-   Generic / Opaque — Standard key / values paired secrets
-   docker-registry — Credentials for pulling container images
-   TLS — Certificates and associated keys

In addition to different types of secrets using kubectl there are different methods getting secrets to the API servers.

**kubectl create secret generic**

-   — from-literal : Type in the key literally

kubectl create secret generic secretname --from-literal=password=password1 --from-literal=username=user1

-   — from-file (or — from-env-file) : use a stored file with credentials stored in plaintext

kubectl create secret generic secretname --from-file=password=./password.txt --from-file=username=./user.txt

**Using a manifest**

-   Create a Kubernetes Secret Object in a yaml or json manifest
-   Encode the secrets in base64

-   Apply the manifest to the API Server

kubectl apply -f ./secret.yaml

Notice how all the manually created secrets expose the secrets to the operator/developer creating or updating the secrets. This adds a risk that every developer has access to the secret in order to be able to use it. A better method would be have a securely stored secret and automation create the secret.

## Secrets in CI/CD pipelines

Automation is key to help ensure that secrets stay secret. Manual interaction introduces the risk of knowledge of secrets unnecessarily.

![[Raw/Notes/Raw/Media/Resources/4dd327735296a78e1b5a430edc93b158_MD5.png]]

Example flow of CI/CD pipeline

By using an automated process allows for additional methods of secret protection

-   Developers do not have to require access to secrets to production systems. Separating roles between secret creation/management and developers.
-   Access to secret stores can be restricted to pipeline accounts.
-   Reduction in incidence of secrets checked into source code repositories.

Considerations when using automated pipelines.

-   Ensure secrets are not recorded in logs, most pipelines software has the ability to mark secrets to exclude or mask in logs.
-   Ensuring caches are cleared on build agents and cannot be retrieved after runs.

## Kubernetes Secrets Store Container Storage Interface (CSI)

The Secrets Store CSI, is an extension of the Kubernetes API, which instruments the connection of a third party secret store/vault directly to pods requiring secrets and mounted as storage.

This approach allows for secrets to be accessed directly by workloads requiring access to them. The benefit of this is that Kubernetes secrets will only be generated while in use, so only exists for the duration of the workload requiring it in use.

![[Raw/Notes/Raw/Media/Resources/d38508e956b56563de8394a92e84621a_MD5.png]]

Secrets Store CSI connects to vault directly to provide secrets to pods

Supported Secrets Stores

-   AWS
-   Azure
-   GCP
-   Hashicorp Vault

As you can see, secrets management is an extensive topic, and deserves further articles to expand out into greater depth. Secrets Management should be thought about at design phase of the development lifecycle, ensuring that secrets are handed correctly and reducing risks to leaking out sensitive data.

## References

[Kubernetes — Encrypting Secrets at Rest](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

[Secrets Store CSI Driver](https://secrets-store-csi-driver.sigs.k8s.io/introduction.html)

[Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)