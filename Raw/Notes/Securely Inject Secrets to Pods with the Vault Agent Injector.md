---
title: Securely Inject Secrets to Pods with the Vault Agent Injector
source: https://medium.com/@seifeddinerajhi/securely-inject-secrets-to-pods-with-the-vault-agent-injector-3238eb774342
clipped: 2023-09-13
published: 
category: security
tags:
  - vault
  - secrets
  - k8s
read: false
---

> A Must-Have Tool for Securing Your Kubernetes Secrets

![[Raw/Media/Resources/3e4193a93e274eedd57e292a4b83db33_MD5.jpg]]

In today’s tech-driven world, keeping data secure is a big deal. Kubernetes helps manage software in containers, but it can have security limits, especially for sensitive info. To fix this, we can connect Kubernetes with Vault, a tool that stores secrets safely.

Vault Agent does the job here, so your apps don’t need to know about Vault. But to make this work, you need to set up Vault Agent with your apps.

The Vault Helm chart makes it easy. It lets you run Vault and Vault Agent Sidecar Injector, which helps manage secrets for your apps. This is great because:

✔️ Apps don’t have to worry about Vault; secrets are stored in their container.

✔️ You don’t have to change your existing setups; just add some annotations.

✔️ You can control who gets access to secrets using Kubernetes.

In this tutorial, we’ll set up Vault and the injector service with the Vault Helm chart. Then, we’ll deploy some apps to show how the injector service handles secrets.

![[Raw/Media/Resources/9fd025df9dc444345548fbb631609bd6_MD5.png]]

Before you begin, make sure you have the following tools installed:

-   *Docker*
-   *Kubernetes command-line interface (CLI)*
-   *Helm CLI*
-   *Minikube*

To get the web application and additional configuration, clone the `seifrajhi/vault-k8s-sidecar-injector` repository from GitHub using the following commands:

git clone https://github.com/seifrajhi/vault-k8s-sidecar-injector.git

After cloning, navigate to the cloned repository:

cd vault-k8s-sidecar-injector

This will take you to the repository directory.

[Minikube](https://minikube.sigs.k8s.io/) is a CLI tool that provisions and manages the lifecycle of single-node [Kubernetes clusters](https://kubernetes.io/docs/concepts/architecture/) locally inside Virtual Machines (VM) on your system.

🔭 Create a Kubernetes cluster using minkube CLI:

minikube start  
😄  minikube v1.30.1 on Darwin 13.3.1 (arm64)  
✨  Using the docker driver based on existing profile  
👍  Starting control plane node minikube in cluster minikube  
🚜  Pulling base image ...  
🔄  Restarting existing docker container for "minikube" ...  
🐳  Preparing Kubernetes v1.26.3 on Docker 23.0.2 ...  
🔗  Configuring bridge CNI (Container Networking Interface) ...  
🔎  Verifying Kubernetes components...  
   ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5  
🌟  Enabled addons: storage-provisioner, default\-storageclass  
🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default

Minikube provides a visual representation of the status in a web-based dashboard. This interface displays the cluster activity in a visual interface that can assist in delving into the issues affecting it.

In another terminal, launch the minikube dashboard.

$ minikube dashboard

The default browser opens and displays the dashboard.

![[Raw/Media/Resources/0239a5fe3424c4d1588c7d3e2081773c_MD5.png]]

Deploying the Vault Helm chart is the preferred method for running Vault on Kubernetes. Helm, acting as a package manager, simplifies the installation and configuration of all essential components required for Vault’s operation in various modes. Helm charts come equipped with templates that facilitate conditional and customizable execution. You can specify these parameters either through command-line inputs or by defining them in YAML files.

***Add the HashiCorp Helm repository:***

$ helm repo add hashicorp https:  
"hashicorp" has been added to your repositories

Update all the repositories to ensure `helm` is aware of the latest versions.

$ helm repo update  
Hang tight while we grab the latest from your chart repositories...  
...Successfully got an update from the "hashicorp" chart repository  
Update Complete. ⎈Happy Helming!⎈

Install the latest version of the Vault server running in development mode.

$ helm install vault hashicorp/vault --set "server.dev.enabled=true"

The Vault pod and Vault Agent Injector pod are deployed in the default namespace.

Display all the pods in the default namespace.

kubectl get pods  
NAME                                    READY   STATUS    RESTARTS   AGE  
vault\-0                                 1/1     Running   0          2m  
vault\-agent\-injector\-4991fb98b5\-tpgof   1/1     Running   0          2m

The `vault-0` pod runs a Vault server in development mode. The `vault-agent-injector` pod performs the injection based on the annotations present or patched on a deployment.

> Running a Vault server in development is automatically initialized and unsealed. This is ideal in a learning environment but NOT recommended for a production environment.

To demonstrate vault agent injector functionality, I will create the following.

1.  A set of secrets using vault kv engine
2.  Vault policy to read the secrets.
3.  Enable vault kubernetes authentication.
4.  Create a vault role to bind vault policy and kubernetes service account (vault-auth).
5.  Create a vault-auth kubernetes service account to be used for vault server authentication.

To establish a secret in Vault, it’s essential for the applications deployed in the next section to have Vault store a username and password at the designated location, namely `internal/database/config.` This process involves `[***enabling a key-value***](https://developer.hashicorp.com/vault/docs/secrets/kv/kv-v2)` secret engine and populating the specified path with the necessary username and password.

Exec into vault pod.

kubectl exec -it vault-0 -- /bin/sh

Your system prompt is replaced with a new prompt `/ $`. Commands issued at this prompt are executed on the `vault-0` container.

Enable the vault kv engine (key-value store) at the path `internal`.

$ vault secrets enable -path=internal kv-v2  
Success! Enabled the kv-v2 secrets engine at: internal/

Create a secret at the path `internal/database/config` with a `username` and `password`.

vault kv put internal/database/config username="db-readonly-username" password="db-secret-password"  
Key              Value  
\---              -----  
created\_time     2023-09-04T09:55:01.111711644Z  
deletion\_time    n/a  
destroyed        false  
version          1

Verify that the secret is defined at the path `internal/database/config`.

$vault kv get internal/database/config  
\====== Metadata ======  
Key              Value  
\---              -----  
created\_time     2023\-09\-04T09:55:01.111711644Z  
deletion\_time    n/a  
destroyed        false  
version          1

\====== Data ======  
Key         Value  
\---         -----  
password    db-secret-password  
username    db-readonly\-username

***Enable Kubernetes authentication:***

Vault provides a [Kubernetes authentication](https://developer.hashicorp.com/vault/docs/auth/kubernetes) method that enables clients to authenticate with a Kubernetes Service Account Token. This token is provided to each pod when it is created.

kubectl exec -it vault-0 -- /bin/sh  
vault auth enable kubernetes  
Success! Enabled kubernetes auth method at: kubernetes/

Vault accepts a service token from any client within the Kubernetes cluster. When authenticating, Vault validates the service account token’s legitimacy by querying a token review endpoint within Kubernetes.

Configure the Kubernetes authentication method to use the location of the Kubernetes API.

$ vault write auth/kubernetes/config \\  
    token\_reviewer\_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \\  
    kubernetes\_host="https://$KUBERNETES\_PORT\_443\_TCP\_ADDR:443" \\   
    kubernetes\_ca\_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt

Write out the policy named `internal-app` that enables the `read` capability for secrets at path `internal/data/database/config`.

$ vault policy write internal-app - <<EOF  
path "internal/data/database/config" {  
   capabilities = \["read"\]  
}  
EOF

Create a vault role named `internal-app` that binds `internal-app` and `internal-app` Kubernetes service account.

vault write auth/kubernetes/role/internal\-app \\  
      bound\_service\_account\_names=internal\-app \\  
      bound\_service\_account\_namespaces=default \\  
      policies=internal\-app \\  
      ttl=24h

***Define a Kubernetes service account:***

The Vault Kubernetes authentication role defined a Kubernetes service account named `internal-app`.

A service account provides an identity for processes that run in a Pod. With this identity we will be able to run the application within the cluster.

Create a Kubernetes service account named `internal-app` in the default namespace.

$ kubectl create sa internal\-app

Apply the deployment defined in `deployment-orgchart.yaml`.

$ kubectl apply -f deployment-orgchart.yaml  
deployment.apps/orgchart created

Get all the pods in the default namespace and note down the name of the pod with a name prefixed with `orgchart-.`

$ kubectl get pods  
NAME                                    READY   STATUS    RESTARTS   AGE  
orgchart\-69697d9598\-l878s               1/1     Running   0          30s  
vault\-0                                 1/1     Running   0          45m  
vault\-agent\-injector\-4991fb98b5\-tpgof   1/1     Running   0          45m

> The Vault-Agent injector looks for deployments that define specific annotations. None of these annotations exist in the current deployment. This means that no secrets are present on the `orgchart` container in the `orgchart` pod.

Verify that no secrets are written to the `orgchart` container in the `orgchart` pod.

$ kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items\[0\].metadata.name}") \\      
\--container orgchart -- ls /vault/secrets

The output displays that there is no such file or directory named `/vault/secrets`:

ls: /vault/secrets: No such file or directory command terminated with exit code 1

***Injecting Secrets into Pods:***

In this process, the deployment operates the pod using the `internal-app` Kubernetes service account within the default namespace. The Vault Agent Injector exclusively makes adjustments to a deployment when specific annotations are present. If an existing deployment lacks these annotations, its definition can be patched to include them.

spec:  
   template:  
      metadata:  
      annotations:  
         vault.hashicorp.com/agent-inject: 'true'  
         vault.hashicorp.com/role: 'internal-app'  
         vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'

1.  These [annotations](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations) define a partial structure of the deployment schema and are prefixed with `vault.hashicorp.com`.

-   `[agent-inject](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject)` enables the Vault Agent Injector service
-   `[role](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#role)` is the Vault Kubernetes authentication role
-   `[agent-inject-secret-FILEPATH](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject-secret-filepath)` prefixes the path of the file, `database-config.txt` written to the `/vault/secrets` directory. The value is the path to the secret defined in Vault.

Patch the `orgchart` deployment defined in `patch-inject-secrets.yaml`.

$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets.yaml)"  
deployment.apps/orgchart patched

A new `orgchart` pod starts alongside the existing pod. When it is ready the original terminates and removes itself from the list of active pods.

Wait until the re-deployed `orgchart` pod reports that it is `[Running](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)` and ready (`2/2`).

This new pod now launches two containers. The application container, named `orgchart`, and the Vault Agent container, named `vault-agent`.

Display the logs of the `vault-agent` container in the new `orgchart` pod.

$ kubectl logs $(kubectl get pod -l app=orgchart -o jsonpath="{.items\[0\].metadata.name}") \\         
\--container vault-agent

Vault Agent manages the token lifecycle and the secret retrieval. The secret is rendered in the `orgchart` container at the path `/vault/secrets/database-config.txt`.

Display the secret written to the `orgchart` container.

$ kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items\[0\].metadata.name}") \\         
\--container orgchart -- cat /vault/secrets/database-config.txt

You can use vault templates to render secrets in required formats. In this example, we will see how to use templates in deployment annotation.

To apply this template a new set of annotations need to be applied.

spec:  
   template:  
      metadata:  
      annotations:  
         vault.hashicorp.com/agent-inject: 'true'  
         vault.hashicorp.com/agent-inject-status: 'update'  
         vault.hashicorp.com/role: 'internal-app'  
         vault.hashicorp.com/agent-inject-secret-database-config.txt: 'internal/data/database/config'  
         vault.hashicorp.com/agent-inject-template-database-config.txt: |  
            {{- with secret "internal/data/database/config" -}}  
            postgresql://{{ .Data.data.username }}:{{ .Data.data.password }}@postgres:5432/wizard  
            {{\- end \-}}

This patch contains two new annotations:

-   `[agent-inject-status](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject-status)` set to `update` informs the injector reinject these values.
-   `[agent-inject-template-FILEPATH](https://developer.hashicorp.com/vault/tutorials/kubernetes/kubernetes-sidecar#agent-inject-template-filepath)` prefixes the file path. The value defines the [Vault Agent template](https://developer.hashicorp.com/vault/docs/agent-and-proxy/agent/template) to apply to the secret's data.

The template formats the username and password as a PostgreSQL connection string.

Apply the updated annotations.

$ kubectl patch deployment orgchart --patch "$(cat patch-inject-secrets-as-template.yaml)"   
deployment.apps/exampleapp patched

1.  Wait until the re-deployed `orgchart` pod reports that it is `[Running](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-phase)` and ready (`2/2`).
2.  Finally, display the secret written to the `orgchart` container in the `orgchart` pod.

$ kubectl exec $(kubectl get pod -l app=orgchart -o jsonpath="{.items\[0\].metadata.name}") \\         
\-c orgchart -- cat /vault/secrets/database-config.txt

The secrets are rendered in a PostgreSQL connection string is present on the container:

postgresql://db-readonly-user:db-secret-password@postgres:5432/wizard

Incorporating Kubernetes alongside Vault for secret management, the use of a Vault injector offers a seamless approach to integrating secrets into pods.

This approach eliminates the need for applications to directly interact with the secret management system; they simply retrieve secrets from a predefined file path.

Furthermore, it’s vital to ensure high availability of the production Vault server. An unavailable Vault server could potentially lead to application disruptions, particularly during activities such as pod restarts or scaling operations.
