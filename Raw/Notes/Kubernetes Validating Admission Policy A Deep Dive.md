---
title: "Kubernetes Validating Admission Policy: A Deep Dive"
source: https://medium.com/@simardeep.oberoi/kubernetes-validating-admission-policy-a-deep-dive-3089a0ce37cd
clipped: 2023-11-15
published: 
category: k8s
tags:
  - development
read: false
---

Kubernetes is always evolving, and with the introduction of the validating admission policy in Kubernetes 1.26, there’s a new layer of intricacy added. Let’s unpack this feature, understand its significance, and see how to use it effectively.

Before delving into the validating admission policy, it’s crucial to grasp how the Kubernetes API functions:

1.  **Sending a Request**: When you, for example, send a request to apply a deployment resource, it’s essentially an HTTP request with a YAML that defines a resource.
2.  **Authorization & Authentication**: This request is vetted by the authorization and authentication layers to confirm the legitimacy of the request.
3.  **Admission Controllers**: The validated request is then sent to the admission controllers. While there are numerous built-in admission controllers, our focus here is on mutating admission. It forwards webhook requests to controllers that might modify the resource definition. Remember, these mutating controllers are not built-in. You can write your own, or they might come from third-party tools, like a service mesh or tools like Kyverno.
4.  **Object Schema Validation**: Once mutations are complete, the final resource version undergoes object schema validation. This step ensures the resource complies with the cluster’s definitions.
5.  **Validating Admission Controller**: Post validation, the request moves to the validating admission controller. It operates similarly to mutating admissions by sending webhook requests. The primary goal here is to validate the resource against specific rules, ensuring good practices and system stability. This is a pivotal feature for any organization aiming to maintain a large-scale, secure Kubernetes setup.
6.  **Persisting in ETCD**: After validation, the resource gets persisted in etcd, a distributed database storing the cluster state. Post this, one or more controllers handle the resource creation, like creating a replica set or pods.

Tools like Kyverno, Datree, and OPA Gatekeeper are built atop these validating admission controllers. They help define and enforce policies. However, with Kubernetes 1.26’s validating admission policy, there’s a potential shake-up in how these tools might operate.

**Creating the Validation Admission Policy:**

To illustrate, consider this example. Suppose you want to apply a validating admission policy to ensure no deployment resource has more than five replicas. You would:

-   Update the kube-apiserver.yaml file to enable the AdmissionPolicy and Beta API

```yaml
vi /etc/kubernetes/manifests/kube-apiserver.yaml

spec:  
  containers:  
  \- command:  
    \- \--feature-gates=ValidatingAdmissionPolicy=true  
    \- \--runtime-config=admissionregistration.k8s.io/v1beta1
```

-   Create a namespace with a label, e.g. `**environment: development**`.

```yaml
apiVersion: v1  
kind: Namespace  
metadata:  
  labels:  
    environment: development  
  creationTimestamp: null  
  name: admission-policy  
spec: {}  
status: {}
```

-   Define a validating admission policy named ‘Max replicas’, which would set a failure policy to reject any request trying to deploy more than five replicas.

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1  
kind: ValidatingAdmissionPolicy  
metadata:  
  name: "maximum-replicas"  
spec:  
  failurePolicy: Fail  
  matchConstraints:  
    resourceRules:  
    \- apiGroups:   \["apps"\]  
      apiVersions: \["v1"\]  
      operations:  \["CREATE", "UPDATE"\]  
      resources:   \["deployments","statefulsets"\]  
  validations:  
    \- expression: "object.spec.replicas <= 5"  
      message: "replicas should be less than or equal to 5"
```

-   Bind this policy to the namespace with the `**environment: development**` label.

```yaml
apiVersion: admissionregistration.k8s.io/v1beta1  
kind: ValidatingAdmissionPolicyBinding  
metadata:  
  name: "maximum-replicas"  
spec:  
  policyName: "maximum-replicas"  
  validationActions: \[Deny\]  
  matchResources:  
    namespaceSelector:  
      matchLabels:  
        environment: development
```

The policy uses CEL (Common Expression Language) to define these conditions. CEL is separate from other languages you might encounter, but its simplicity and ease make it worth exploring. For those interested in a deeper dive, I’ll link to more extensive resources at the end of this article.

-   Your deploy.yaml file should look like below. Once confirmed, apply your changes to kubectl

```yaml
\---  
apiVersion: v1  
kind: Namespace  
metadata:  
  labels:  
    environment: development  
  creationTimestamp: null  
  name: admission-policy  
spec: {}  
status: {}  
\---  
apiVersion: admissionregistration.k8s.io/v1beta1  
kind: ValidatingAdmissionPolicy  
metadata:  
  name: "maximum-replicas"  
spec:  
  failurePolicy: Fail  
  matchConstraints:  
    resourceRules:  
    \- apiGroups:   \["apps"\]  
      apiVersions: \["v1"\]  
      operations:  \["CREATE", "UPDATE"\]  
      resources:   \["deployments","statefulsets"\]  
  validations:  
    \- expression: "object.spec.replicas <= 5"  
      message: "replicas should be less than or equal to 5"  
\---  
apiVersion: admissionregistration.k8s.io/v1beta1  
kind: ValidatingAdmissionPolicyBinding  
metadata:  
  name: "maximum-replicas"  
spec:  
  policyName: "maximum-replicas"  
  validationActions: \[Deny\]  
  matchResources:  
    namespaceSelector:  
      matchLabels:  
        environment: development
```

kubectl create \-f deploy.yaml

**Deploying an Application:** Now deploy a sample nginx deployment. This is accomplished via:

```
k create deploy nginx \--image=nginx \--replicas=1 \--namespace=admission-policy  
deployment.apps/nginx created

k get pods \--namespace=admission-policy  
NAME                     READY   STATUS    RESTARTS   AGE  
nginx-7854ff8877-zwrdh   1/1     Running   0          3s
```

The outcome was successful. The likely reason? The application fits within the established policy constraints. This is evident when we list all the pods in the ‘admission-policy’ namespace and observe that there’s a single pod.

**Testing Policy Limits:** To demonstrate the policy’s constraints, let’s intentionally breach them. I’ll change the number of replicas count in the deployment from to seven.

```
k apply \-f  nginx.yaml \--namespace=admission-policy  
The deployments "nginx" is invalid: : ValidatingAdmissionPolicy 'maximum-replicas' with binding 'maximum-replicas' denied request: replicas should be less than or equal to 5

The outcome is as expected: our deployment effort encounters a validation policy. This policy, named ‘Max replica’ and bound to ‘Max replica binding’, intervenes to indicate the violation: replicas must be five or fewer.
```

**Pros**:

-   **Standardization**: The validating admission policy is set to become the standard for validating Kubernetes resources.
-   **Expressive Language**: The adoption of CEL offers flexibility and power, surpassing pure YAML.

**Cons**:

-   **Lack of Context**: The main drawback of admission controllers, including the validating admission policy, is their inability to understand the broader context. For instance, while looking at replicas in a deployment, one should also consider related resources like HPA (Horizontal Pod Autoscaler).

The introduction of the validating admission policy in Kubernetes 1.26 signals an important evolution in Kubernetes. While third-party tools are valuable, once the Kubernetes community defines a standard API, it’s generally prudent to adopt it. The use of CEL is a commendable choice, making the task of defining conditions more straightforward and expressive. However, there’s room for improvement, especially in understanding and handling resource context more effectively.