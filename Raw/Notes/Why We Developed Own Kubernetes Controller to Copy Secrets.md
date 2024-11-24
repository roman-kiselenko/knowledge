---
title: Why We Developed Own Kubernetes Controller to Copy Secrets
source: https://medium.com/lonto-digital-services-integrator/why-we-developed-own-kubernetes-controller-to-copy-secrets-e46368ae6db9
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - operators
read: true
---

Kubernetes is a superior platform for deploying and managing applications, but sometimes simple tasks like having the same data — in our case Secrets — between namespaces can cause slight problems and you have to build your own solutions to those problems.

Hello! My name is Igor Latkin, I am a solutions architect at [Lonto](https://www.lonto.io/).

Today, I would like to share a project we developed internally — [***mirrors***](https://github.com/ktsstudio/mirrors) Kubernetes controller. We’ve created it inside of our DevOps department to solve the problem of copying Kubernetes secrets between cluster namespaces. As a result, *mirrors* turned into a general-purpose tool designed to synchronise data from different sources. In the article, I’ll tell a story how it all started and where we came in the end of the road. Hopefully, it will inspire you to create your own controller tailored to meet your needs.

**This is the article outline:**

1.  How it all started
2.  Dynamic environments with TLS
3.  The problem of too many certificates
4.  The problem of a single certificate
5.  Existed solutions and their drawbacks
6.  SecretMirror for the rescue
7.  Controller architecture
8.  Copying secrets between namespaces
9.  Copying secrets from HashiCorp Vault to the Kubernetes cluster
10.  Copying secrets from Kubernetes cluster to HashiCorp Vault
11.  Bonus
12.  Results and comparison diagrams

> 💡 **We faced a single big task** — to enable TLS in dynamic environments in the Lonto dev environment.

All development teams use our dev cluster, that’s why the process of ensuring secure connection must be fully autonomous. Later we understood that we could have used the developed solution for other tasks I’ll also tell about in this article later.

Let’s begin with dynamic environments.

We at Lonto faced the task of copying secrets between namespaces in the Kubernetes cluster for a while already. Our processes are built the way that makes every team — and even every developer in the company — quite independent and self-sufficient.

Every developer can create a repository in Gitlab, connect a shared CI/CD pipeline, define necessary domain names for the project in the local *gitlab-ci.yml* file, and get the project deployed in multiple dev- and production environments in a couple of minutes. And they actually don’t need to engage the DevOps team for that at all.

```yaml
include:  
  project: mnt/ci  
  file: front/ci.yaml  
  ref: ${CI_REF}

variables:  
  DEV_BASE_DOMAIN: projectA.dev.example.com  
  DEV_API_DOMAIN: projectA.dev.example.com  
  PROD_DOMAIN: projectA.prod.example.com
```

This gitlab-ci.yml in the project turns into a much bigger pipeline covering the tasks of building and deploying a project in various environments:

![[Raw/Media/Resources/169f42781d33a115ac6598b7c6bc885b_MD5.png]]

Some of the pipelines that occur out of that small .gitlab-ci.yml file

Almost all projects in the dev environment are deployed to a subdomain of a single domain, which we will refer to as [dev.example.com](http://dev.example.com/). That is, the projects can be deployed to the following subdomains:

-   projectA.dev.example.com
-   projectB.dev.example.com
-   projectC.dev.example.com

Another thing to note is that some applications consist of multiple micro-services united by different ingress rules. For example:

-   **A frontend application** maintains all paths of [projectA.dev.example.com/](http://projecta.dev.example.com/) URL
-   **API application** maintains all paths of [projectA.dev.example.com/api](http://projecta.dev.example.com/api) URL

As they are hosted on exactly the same domain name, it is desirable to correctly specify a single TLS certificate for these ingresses. They are deployed to different namespaces to enhance isolation and just because this is how the “certificate-based” integration of Gitlab with Kubernetes works.

> Generally speaking, the certificate-based integration of Gitlab with Kubernetes is already deprecated and it’s worth migrating to Gitlab Agent (or to another GitOps instrument such as ArgoCD or FluxCD). But it’s not what we are going to cover in this article.

You’d think one can just issue a certificate in every ingress of every project, and the problem is solved. This is exactly how we lived for a while. But in the end, we bumped in the [Let’s Encrypt limitations](https://letsencrypt.org/docs/rate-limits/) by the number of issued certificates. We experienced it especially vividly when we had bulk migration from one cluster to another, or when we had to re-issue all certificates for various reasons.

The second drawback of this solution is that you have to wait a certain time until the certificate is actually issued when a new project’s branch is created for example. Besides, this process can fail and you may not have a certificate at all. That’s why it seems natural to keep only one certificate and enable access to it for everybody.

But then another problem arises.

The certificate issued to \*.dev.example.com is valid for [projectA.dev.example.com](http://projecta.dev.example.com/), but invalid for [feature1.projectA.dev.example.com](http://feature1.projecta.dev.example.com/). Therefore, if we want to build a dynamic environments with subdomains, we’ll be a hostage to this solution.

So we articulated the task as the following:

> **💡 Task #1**
> 
> In order to support TLS in the project’s dynamic environments, it is necessary to issue the certificate for the **main** branch of every project and copy the Secret containing the certificate data to all other namespaces of this project.

![[Raw/Media/Resources/11d56c1d1091bd3b37e6476476a6f00a_MD5.png]]

For the project with name *<project\_name>*, id *<project\_id>* and the branch name equal to *<branch\_name>* the namespaces will have the names as the following:

*<project\_name>-<project\_id>-<branch\_name>-<some\_hash>*

Some examples:

-   project-a-1120-main-23hf
-   project-a-1120-dev-4hg5
-   project-a-1120-feature-1-aafd
-   project-b-1200-main-7ds9
-   project-b-1200-feature1–42qq

That is, physically the [Certificate](https://cert-manager.io/docs/usage/certificate/) object is created only in *project-a-1120-****main****\-23hf* and *project-b-1200-****main****\-7ds9* namespaces. The result of issuing the certificate — Secret that contains tls.crt and tls.key — must be copied to all other relevant namespaces:

```yaml
apiVersion: v1  
kind: Secret  
type: kubernetes.io/tls  
metadata:  
  name: lontodev-wild-cert  
data:  
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0F...  
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkF...
```
  

Now let’s consider the second task we’ve been able to cover. It is close enough to the first case, but has its own peculiarities.

Imagine that we are talking about some production environment, and this environment is in the Kubernetes cluster. Suppose it consists of multiple clusters distributed across geographic areas. And all these clusters provide services to a single domain name, for example, [myshop.example.com](http://myshop.example.com/). We really want our end user to see a single TLS certificate, irrespective of the cluster it gets onto. The certificate may initially come from a variety of sources:

1.  You can buy a certificate.
2.  A certificate can be issued via various tools such as cert-manager in one of the clusters. Then we have to decide how to deliver it to all of the production clusters.

Naturally, it makes sense to use a centralised storage to keep the certificate and use some tool to deliver it to the places of use you need.

Before we implemented our discussed solution, we had a point-blank but viable solution: the certificate was deployed using **helm** to all necessary clusters as a part of the infrastructure. To update the certificate, it was enough to update it in the helm chart and deploy it again. But we wanted to enhance automation and safety: for example, we didn’t like storing the certificate in the git repository.

Of course, the problems arise beyond the certificate realm as well. These can be any data you want to access in multiple places at once — credentials for image registry, database logins / passwords and much more.

So now we state both tasks we had:

> **💡 Task #1 (just to remember)**
> 
> In order to support TLS in the project’s dynamic environments, it is necessary to issue the certificate for the **main** branch of every project and copy the Secret containing the certificate data to all other namespaces of this project.
> 
> **💡 Task #2**
> 
> To be able to automatically synchronise Secret from the centralised storage to all Kubernetes clusters and namespaces chosen within them.

After we understood what exactly we wanted to do, we started searching for satisfactory solutions. Two projects we kept in mind to solve the first task:

-   [kubed](https://appscode.com/products/kubed/) by AppsCode
-   [kubernetes-reflector](https://github.com/EmberStack/kubernetes-reflector) by EmberStack

We set aside **kubed** right away, as it relies on labels you need to add both to Secrets or ConfigMaps, and the namespaces you copy them to. We didn’t have an opportunity to dynamically add labels when creating namespaces. Learn [more](https://appscode.com/products/kubed/v0.12.0/guides/config-syncer/intra-cluster/) about how kubed works .

**kubernetes-reflector** works similarly, but can follow the Сertificate objects and add extra annotations or labels to enable the secret synchronisation.

It is also allowed to specify a regular expression the namespace should meet for the secret to be copied to it. There is no need to label the Namespace objects. Let’s have a look at the example:

```yaml
apiVersion: v1  
kind: Secret  
metadata:  
 name: source-secret  
 annotations:  
   reflector.v1.k8s.emberstack.com/reflection-allowed: "true"  
   reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces: "project-a-1120-\*"  
data:  
 ...
```

Here, the most important part is the *reflector.v1.k8s.emberstack.com/reflection-allowed-namespaces* annotation. With it, we were supposed to easily solve the **first task**: deploy the Certificate only for the main branch, so it was copied automatically to all other branches. The *project-a-1120-\** mask perfectly describes, which branches the Secret needs to be copied to.

We set up all projects and deployed this solution to our dev cluster. Generally, kubernetes-reflector got things done, until developers started complaining that sometimes certificates could not be issued. As a matter of fact, the certificate was issued, but for some reason it would not copy to a new feature branch.

We’ve seen the following picture in the kubernetes-reflector resources usage graphs (different colors depict different pods):

![[Raw/Media/Resources/837fc3842c9ecf65b9cbf54dd8f3f430_MD5.jpg]]

CPU graph. Generally, the values are acceptable

![[Raw/Media/Resources/a5499b33c304b40226a6f3e52b6778d7_MD5.jpg]]

RAM graph. An extremely high memory consumption for a simple controller

![[Raw/Media/Resources/5699e577b48fecedda36a106eaf96456_MD5.jpg]]

Network utilisation graph. kubernetes-reflector obviously downloaded huge amounts of data

Because of the high memory consumption and the graph being multi-coloured, it was clear that the reflector had been killed by the OOM, which caused it to restart every time. Utilisation of network resources was also surprising and meant that the controller tried to download a lot of data from the kube-api.

Unfortunately, as the number of projects grew, drawbacks of the reflector’s internal architecture had been brought to light. At that time, the cluster had about 10k different Secrets. All things considered, the reflector watched each one of them and didn’t use the namespace details efficiently. That’s why we observed extensive use of the network traffic on the graphs.

> In later reflector versions, the performance issues have been [addressed](https://github.com/emberstack/kubernetes-reflector/commit/577562ecac48fc2939d1c80981eb82d7bfee83b0), but this happened two months after we migrated to our own solution.

Besides, we really wanted the Secret to appear in the necessary namespace only moments after it is created in the Kubernetes cluster, and not after some internal synchronisation period.

In the end, we decided to develop our internal tool as soon as possible. We wanted this tool to handle our tasks and address the performance issues the reflector had. We managed it in two weeks: deployed it to our dev cluster and migrated all teams to this tool.

![[Raw/Media/Resources/70afe2efc4d1004872edc521422bf80a_MD5.jpg]]

Athena presents Perseus a mirrored shield. “Allegory of Frederick the Great as Perseus” — Christian Bernhard Rode

So, here are the requirements for our new controller:

1.  **It must work with its own** [**CRD**](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)**.** It will be called SecretMirror. This requirement drastically lowers the API server workload and controller performance resources: obviously there is a much smaller amount of SecretMirror entities than Secrets themselves.
2.  **SecretMirror must accept the list of regular expressions of the namespaces where the Secret must be copied to.** This allows us to flexibly manage the target location of resources.
3.  **The controller must watch not only SecretMirror, but the Namespace objects as well.** This allows copying the Secret to a new namespace right after the namespace is created, and not after an arbitrary synchronisation period.
4.  **It has to maintain the up-to-date list of namespaces in the memory cache.** Therefore we save a lot of traffic by not retrieving the same list every time we need to synchronise the Secret to the namespaces. This cache is easy-to-support due to fulfilling point #3, and we can dynamically add and delete a namespace to or from it.
5.  **When deleting SecretMirror, all Secrets created by the controller must be automatically deleted.** However, we need an option to disable such behaviour.
6.  **In case the original source Secret changes, its synchronisation does not need to be immediate.** Otherwise, we’d have to watch all the Secret objects in the cluster again. It’s good enough if synchronisation takes place once in a certain period of time: for example, with a 3-minutes interval. This interval must be modifiable for every particular SecretMirror.
7.  **The controller must be expandable.** Therefore external systems, such as HashiCorp Vault, can be used as the Secret source or destination.

![[Raw/Media/Resources/ce21e380a741f34726424cb912fb7ca4_MD5.png]]

Controller architecture

It’s all quite simple. Inside the ***mirrors***, two controllers are launched that watch any changes of two [GVKs](https://book.kubebuilder.io/cronjob-tutorial/gvks.html) — *mirrors.kts.studio/v1alpha2.SecretMirror* and *v1.Namespace*. All namespace changes are stored in the controller memory to minimise the amount of the Kubernetes API access.

Now let’s get back to the task #2: to synchronise data between multiple clusters. Here at Lonto we actively use [HashiCorp Vault](https://www.vaultproject.io/) to store secret data, and decided to take the same path here.

Vault will be used as the state synchronisation system. For this, we needed to teach SecretMirror to read and write secrets from / to Vault. In the use cases below we’ll see how this can be used.

![[Raw/Media/Resources/fd63920f12e11ce4f7c9612bb3246c5f_MD5.png]]

The scenario flowchart

For starters, let’s consider the first scenario in detail, the one SecretMirror was initially created for. The manifest looks as follows:

apiVersion: mirrors.kts.studio/v1alpha2  
kind: SecretMirror  
metadata:  
  name: mysecret-mirror  
  namespace: default  
spec:  
  source:  
    name: mysecret  
  destination:  
    namespaces:  
      \- project-a-.+  
      \- project-b-.+

Its task is to copy a Secret named mysecret from the default namespace to all namespaces that start either with *project-a-*, or *project-b-* prefix. Let’s apply the manifest and output the list of all SecretMirrors:

$ kubectl apply \-f mysecret\-mirror.yaml  
$ kubectl get secretmirrors  
NAME              SOURCE TYPE   SOURCE NAME   DESTINATION TYPE   DELETE POLICY   POLL PERIOD   MIRROR STATUS   LAST SYNC TIME         AGE  
mysecret\-mirror   secret        mysecret      namespaces         delete          180           Pending         1970\-01\-01T00:00:00Z   15s

SecretMirror is in the Pending state — it is waiting for the source Secret to appear. Let’s deploy it:

apiVersion: v1  
kind: Secret  
metadata:  
  name: mysecret  
  namespace: default  
type: Opaque  
stringData:  
  username: hellothere  
  password: generalkenobi

The SecretMirror status is changed from Pending to Active:

$ kubectl get secretmirrors  
NAME              SOURCE TYPE   SOURCE NAME   DESTINATION TYPE   DELETE POLICY   POLL PERIOD   MIRROR STATUS   LAST SYNC TIME         AGE  
mysecret-mirror   secret        mysecret      namespaces         delete          180           Active          2022-08-05T21:28:55Z   5m2s

Now let’s create namespaces the Secret must be copied to:

$ kubectl create ns project-a-main  
$ kubectl create ns project-b-main

The Secrets are immediately copied to these new namespaces:

$ kubectl get secret -A | grep "mysecret"  
NAMESPACE          NAME         TYPE      DATA   AGE  
default            mysecret     Opaque    2      6m23s  
project-a-main     mysecret     Opaque    2      23s  
project-b-main     mysecret     Opaque    2      23s

In kubectl describe SecretMirror you can see more detailed information related to the object events:

Name:         mysecret-mirror  
Namespace:    default  
Labels:       <none>  
Annotations:  <none>  
API Version:  mirrors.kts.studio/v1alpha2  
Kind:         SecretMirror  
Metadata:  
  Creation Timestamp:  2022-08-05T21:23:55Z  
  Finalizers:  
    mirrors.kts.studio/finalizer  
  Generation:  2  
  Resource Version:  109673149  
  UID:               825ded22-0e90-4576-9608-1b63a1b02428  
Spec:  
  Delete Policy:  delete  
  Destination:  
    Namespaces:  
      project-a-.+  
      project-b-.+  
    Type:               namespaces  
  Poll Period Seconds:  180  
  Source:  
    Name:  mysecret  
    Type:  secret  
Status:  
  Last Sync Time:  2022-08-05T21:38:41Z  
  Mirror Status:   Active  
Events:  
  Type     Reason    Age                 From                Message  
  \----     \------    \----                \----                \-------  
  Warning  NoSecret  10m (x11 over 15m)  mirrors.kts.studio  secret default/mysecret not found, waiting to appear  
  Normal   Active    10m                 mirrors.kts.studio  SecretMirror is synced

![[Raw/Media/Resources/c272556db5fc5fae2ee90b6201fbe5d0_MD5.png]]

The scenario flowchart

Let’s consider another scenario. Imagine that our Vault cluster has a Secret with the following contents:

![[Raw/Media/Resources/026504875c41a64a557da16db23ded1c_MD5.png]]

And our goal is to regularly synchronise these data with the Secret in the Kubernetes cluster. This is how the manifest for SecretMirror that solves this task looks like:

```yaml
apiVersion: mirrors.kts.studio/v1alpha2  
kind: SecretMirror  
metadata:  
  name: myvaultsecret-mirror  
  namespace: default  
spec:  
  source:  
    name: myvaultsecret-sync  
    type: vault  
    vault:  
      addr: https://vault.example.com  
      path: /secret/data/myvaultsecret  
      auth:  
        approle:  
          secretRef:  
            name: vault-approle  
  destination:  
    type: namespaces  
    namespaces:  
      \- project-c-.+
```

Due to this configuration, ***mirrors*** controller will synchronise the Vault secret named *myvaultsecret* to Kubernetes Secret named *myvaultsecret-sync* to the namespaces, whose names start with *project-c-* prefix. Currently, our integration with Vault supports 2 types of authentication:

1.  Token-based
2.  AppRole-based

You can learn more about how to set up authentication in the project’s [README](https://github.com/ktsstudio/mirrors/#vault-examples).

So the task #2 can be easily solved with the described scenario using the centralised storage. In particular, we can place *tls.crt* and *tls.key* data in the Vault, set up SecretMirror and get an opportunity to maintain the up-to-date state of certificate in one or multiple clusters at any time by just updating the certificate in the Vault itself.

![[Raw/Media/Resources/86e3455e8974de8d6431e5a21d18fc52_MD5.png]]

The scenario flowchart

Getting back to one of our primary tasks one can remember that a TLS certificate can also be issued by the cert-manager. We’d like to synchronise it with other clusters of our production environment. Here, one can use the same integration with Vault. Only this time we will synchronise the secret not **from** the Vault, but **to** it from the Kubernetes Secret.  
Less words, more YAMLs:

```yaml
apiVersion: mirrors.kts.studio/v1alpha2  
kind: SecretMirror  
metadata:  
  name: myvaultsecret-mirror-reverse  
  namespace: default  
spec:  
  source:  
    name: mysecret  
  destination:  
    type: vault  
    vault:  
      addr: https://vault.example.com  
      path: /secret/data/myvaultsecret  
      auth:  
        approle:  
          secretRef:  
            name: vault-approle
```

In this case it should be clear that Secret *mysecret* will be used as the source, and the Secret *myvaultsecret* in Vault — as the destination. To synchronise the Secret to other clusters, it’s necessary to create a SecretMirror inside of them, as in the previous scenarios descrining synchronisation from Vault to Secret Kubernetes.

Let’s have a look at several bonus scenarios SecretMirror can help with due to its design.

## 1\. Distributing *dynamic* secrets from Vault to Kubernetes Secret

HashiCorp Vault is also known for being able to generate dynamic credentials to access [supported](https://www.vaultproject.io/docs/secrets/databases#database-capabilities) databases on-the-fly. For example, it can generate a temporary password to access PostgreSQL or MongoDB cluster for some back-up script or another cron routine. Static logins / passwords may leak in a variety of ways: in logs, in the messenger, or can be stored unencrypted in the developer’s PC. With dynamic secrets, you can avoid this issue, creating temporary access and destroying it after the timeout expiry.

Below is an example how SecretMirror can help synchronising the dynamic password for MongoDB:

apiVersion: mirrors.kts.studio/v1alpha2  
kind: SecretMirror  
metadata:  
  name: secretmirror-from-vault-mongo-to-ns  
  namespace: default  
spec:  
  source:  
    name: mongo-creds  
    type: vault  
    vault:  
      addr: https://vault.example.com  
      path: mongodb/creds/somedb  
      auth:  
        approle:  
          secretRef:  
            name: vault-approle  
  destination:  
    type: namespaces  
    namespaces:  
      \- default

> Please note that at every moment of synchronisation, mirrors will extend the so-called lease, instead of generating a new password each time. That’s why credentials will remain the same throughout the **max\_ttl** period set in the Vault.

## 2\. Copying secrets from Vault to Vault

You could have guessed that there is an opportunity to specify *source.type = vault* and *destination.type = vault* as well. This is really the case, and Kubernetes Secrets aren’t used at all. One of the possible applications is to copy a certain secret from one Vault cluster to the other one, or to copy a key from one place to another within a single Vault cluster.  
The SecretMirror example of copying between Vault clusters:

apiVersion: mirrors.kts.studio/v1alpha2  
kind: SecretMirror  
metadata:  
  name: secretmirror-from-vault-to-vault  
  namespace: default  
spec:  
  source:  
    name: mysecret  
    type: vault  
    vault:  
      addr: https://vault1.example.com  
      path: /secret/data/mysecret  
      auth:  
        approle:  
          secretRef:  
            name: vault1-approle

  destination:  
    type: vault  
    vault:  
      addr: https://vault2.example.com  
      path: /secret/data/mysecret  
      auth:  
        approle:  
          secretRef:  
            name: vault2-approle

Have we managed to solve the primary tasks? No doubt.

All the teams are happy now — certificates appear in the feature branches immediately, secrets between the clusters are synchronised for several DevOps clients without doing anything manually, and there is still room for improvements.

Our controller consumes very little CPU, memory and network resources, and has practically no additional workload on the cluster.

Diagrams to compare with kubernetes-reflector:

![[Raw/Media/Resources/b528de2e4e3de83d707cdafd247ce690_MD5.png]]

CPU Graph

![[Raw/Media/Resources/9237b391f6ad156cd5aadec2ef1dd3d7_MD5.png]]

RAM Usage Graph

![[Raw/Media/Resources/664cb9894bbec8308449791dfe1c4a97_MD5.png]]

Network Graph

This was the first custom Kubernetes controller that we developed within the team when no other solution was able to satisfy our needs.

It turned out that it is a rather uncomplicated procedure and it allows creating a lot of custom workflows within Kubernetes clusters, but moreover it simply broadens your knowledge of how Kubernetes works internally.

If you would like to try out ***mirrors*** for yourself, these are some useful links:

1.  [https://github.com/ktsstudio/mirrors](https://github.com/ktsstudio/mirrors) — main GitHub repository.
2.  [Helm-chart](https://github.com/ktsstudio/helm-charts/tree/main/charts/mirrors) with installation instructions.
3.  [Terraform-module](https://github.com/ktsstudio/terraform-modules/tree/main/mirrors) that installs the chart above.

Thank you for your time and see you soon :)