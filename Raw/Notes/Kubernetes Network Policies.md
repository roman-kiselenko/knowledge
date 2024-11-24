---
title: "Kubernetes: Network Policies"
source: https://yuminlee2.medium.com/kubernetes-network-policies-a93c2f588e31
clipped: 2023-09-04
published: 
category: network
tags:
  - k8s
  - network
read: false
---

![[Raw/Media/Resources/cdf97a66acbd7d3603f0524738f81760_MD5.png]]

ingress and egress traffic

**Ingress** **traffic** refers to the **incoming network traffic** that is **directed to a pod or a group of pods** in the Kubernetes cluster. For example, if a user outside the cluster sends a request to a pod within the cluster, the traffic would be considered ingress traffic to that pod.

**Egress traffic**, on the other hand, refers to the **outgoing network traffic** **from a pod or a group of pods** in the Kubernetes cluster. For example, if a pod in the cluster sends a request to a service or an external endpoint outside the cluster, the traffic would be considered egress traffic from that pod.

**By default**, Kubernetes clusters allow **unrestricted communication between pods and external access**, which can pose security risks, especially in multi-tenant environments where multiple applications and teams coexist.

**NetworkPolicy** is a **Kubernetes object** that allows you to create policies that **define how pods can communicate with each other and with external entities within a specific namespace**. NetworkPolicy rules can be based on various factors such as IP addresses, ports, protocols, and labels, enabling you to restrict traffic to specific pods or groups of pods based on your security requirements.

$ kubectl api-resources  
NAME             SHORTNAMES   APIVERSION            NAMESPACED  KIND  
networkpolicies  **netpol**       networking.k8s.io/v1  true        NetworkPolicy

## podSelector

The `podSelector` field **selects pods based on their labels** and determines which pods the policy applies to.

![[Raw/Media/Resources/d8cc47394b54f346ee4ee0b7fa780e8d_MD5.png]]

podSelector

apiVersion: networking.k8s.io/v1  
kind: NetworkPolicy  
metadata:  
  name: backend-network-policy  
spec:  
  podSelector:  
    matchLabels:  
      name: backend  
  policyTypes:      
    \- Ingress      
    \- Egress  
  ingress:  
    \- from:  
        \- podSelector:  
            matchLabels:  
            name: frontend  
      ports:  
        \- port: 8080  
          protocol: TCP  
  egress:  
    \- to:  
        \- podSelector:  
            matchLabels:  
            name: database  
      ports:  
        \- port: 5432  
          protocol: TCP

In this case, this NetworkPolicy targets pods labeled with `name: backend`. The `ingress` section defines **incoming traffic rules** from `name: frontend` pods on `port` 8080. The `egress` section defines **outgoing traffic rules** to `name: database` pods on `port` 5432.

## namespaceSelector

`namespaceSelector` is a field that allows you to **select particular namespaces** and **apply network policy rules to all the pods within those namespaces**.

![[Raw/Media/Resources/f7e89f4ddb1501b83dc00594a19ce5e1_MD5.png]]

namespaceSelector

...  
ingress:  
  - from:  
      - **namespaceSelector:  
          matchLabels:  
          name: namespace1**  
    ports:  
      - port: 8080  
        protocol: TCP  
...

In this case, it allows traffic from the pods in `namespace1`.

## **ipBlock**

`ipBlock` is a field used to **specify IP address blocks** that are **allowed to access or denied access to the pod**. It can be used to define a CIDR block or a single IP address.

![[Raw/Media/Resources/1d9c79cf9507e6049b2995ff2cc04445_MD5.png]]

ipBlock

...  
ingress:  
  - from:  
      **- ipBlock:  
          cidr: 192.168.0.0/16**  
    ports:  
      - port: 8080  
        protocol: TCP  
...

In this example, the `ipBlock` field is used to specify the CIDR block `192.168.0.0/16`. The `ingress` section allows incoming traffic to the pod on `port` 8080 using the TCP protocol only if the source IP address is within this CIDR block.