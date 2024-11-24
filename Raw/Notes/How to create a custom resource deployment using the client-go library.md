---
title: How to create a custom resource deployment using the client-go library
source: https://medium.com/@vsdvprmr/how-to-create-a-custom-resource-deployment-using-the-client-go-library-52f7ba0587eb
clipped: 2023-09-04
published: 
category: k8s
tags:
  - k8s
  - operators
read: true
---

In the [previous article](https://medium.com/@vsdvprmr/performing-crud-operations-on-nginx-deployment-with-go-gin-framework-a1dfbf7846ac), we discussed the process of creating an Nginx deployment using the client-go library. Now, let’s focus on exploring the steps involved in creating a custom resource deployment.

first, look at the *Nginx deployment* K8s manifest:-
```yaml
apiVersion: apps/v1     
kind: Deployment  
metadata:  
  name: nginx-deployment  
spec:  
  replicas: 3  
  selector:  
    matchLabels:  
      app: todo  
  template:  
    metadata:  
      name: nginx-pod  
      labels:  
        app: todo  
    spec:  
      containers:  
        - name: nginx-container  
          image: nginx:latest  
          ports:  
            - containerPort: 80
```

Kubernetes provides a set of built-in resource types like Pods, Deployments, Services, ConfigMaps, etc., like for nginx deployment those are specific in the above YAML file.

> <mark style="background: #FF5582A6;">To get the API-resources present in the K8s cluster in CMD run the command *kubectl api-resources*</mark>

In complex applications or specialized domains, using the existing resources requirement may not be fulfilled. That’s where Custom Resources came into the picture.

custom resources extend the Kubernetes API and provide a flexible and scalable way to model and manage complex applications and infrastructure within the Kubernetes framework.

To define a custom resource in Kubernetes, you need to perform the following steps:

Define the Custom Resource Definition (CRD):

-   Create a YAML or JSON file that describes the custom resource and its schema.
-   The CRD defines the API endpoints, structure, and validation rules for your custom resource.
-   The CRD should specify the group, version, and kind of custom resource.

```yaml
apiVersion: apiextensions.k8s.io/v1  
kind: CustomResourceDefinition  
metadata:  
  name: myresources.example.com  
spec:  
  group: example.com  
  version: v1  
  scope: Namespaced  
  names:  
    plural: myresources  
    singular: myresource  
    kind: MyResource
```

> Create CRD with command *kubectl apply -f crd-definition.yaml*

Create the Custom Resource (CR):
```yaml
apiVersion: example.com/v1  
kind: MyResource  
metadata:  
  name: myresource-sample  
spec:  
```

> Apply CR with command *kubectl apply -f custom-resource.yaml*

It was very brief about CRD and CR. let’s talk about the CR deployment with client-go library,

Nginx deployment was straight-forward where K8s client *“k8s.io/client-go/kubernetes”* was used

> client.AppsV1().Deployments(“namespace”).Create(context.TODO(), deployment, metav1.CreateOptions{})

Here’s a breakdown of the code:

-   `client`: It refers to an instance of the Kubernetes client that is already configured to communicate with a Kubernetes cluster. This client is typically created using the client-go library.
-   `AppsV1()`: It specifies the version of the Apps API group to which the Deployment resource belongs. In this case, it's using the v1 version.
-   `Deployments("namespace")`: It selects the "Deployments" resource within the specified namespace, which in this case is "`namespace`". Replace "default" with the actual namespace where you want to create the Deployment.
-   `Create(context.TODO(), deployment, metav1.CreateOptions{})`: it invokes the `Create` method to create the Deployment resource with the deployment object passed as a parameter.

The same client will not work for Custom Resource Deployment, and Go provide dynamic client *“k8s.io/client-go/dynamic”* for interacting with the Kubernetes API. It allows one to work with Kubernetes resources without requiring specific knowledge or code generation for each resource type. With the dynamic client, CRUD operations can be performed on resources using generic API calls. It’s useful when one needs to work with custom resources or when one wants to work with Kubernetes resources without having to define their specific types in code.

1.  Create Dynamic client

```go
var kubeconfig = "your kube config file path"  
config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)  
 if err != nil {  
  log.Fatalf("Error building kubeconfig: %v", err)  
 }

  
 client, err := dynamic.NewForConfig(config)  
 if err != nil {  
  log.Fatalf("Error creating dynamic client: %v", err)  
 }else{  
  log.Println("Dynamic client created")  
 }
```

2\. Create the unstructured object with the desired YAML content (Using `flinksessionjobs` which is Custom Resouce)

```go
obj := &unstructured.Unstructured{  
  Object: map\[string\]interface{}{  
   "apiVersion": "flink.apache.org/v1beta1",  
   "kind":       "FlinkSessionJob",  
   "metadata": map\[string\]interface{}{  
    "name": "name of your application",  
   },  
   "spec": map\[string\]interface{}{  
    "deploymentName": "Deployment name to which this FlinkSessionJob belongs",  
    "job": map\[string\]interface{}{  
     "jarURI":      "Your Jar file",  
     "parallelism": 4,  
     "upgradeMode": "upgrade mode",  
     "state": "running",  
    },  
   },  
  },  
 }
```

3\. Set the GroupVersionKind for the object
```go

obj.SetGroupVersionKind(schema.GroupVersionKind{  
  Group:   "flink.apache.org",  
  Version: "v1beta1",  
  Kind:    "FlinkSessionJob",  
 })
```

4\. Use the client to create custom resource

```go
result, err := client.Resource(schema.GroupVersionResource{  
  Group:    "flink.apache.org",  
  Version:  "v1beta1",  
  Resource: "flinksessionjobs",  
 }).Namespace("namespace").Create(context.TODO(), obj, metav1.CreateOptions{})

if err != nil {  
 fmt.Println("Error creating custom resource: %v", err)  
}else{  
fmt.Println("Custom resource created successfully ", result)  
}
```

This code creates Custom Resources of the type `flinksessionjobs` in the `flink.apache.org` API group with version `v1beta1` within the `"default"` namespace. It utilizes the `client.Resource` the method from the Kubernetes client to perform the creation operation. The `obj` object represents the configuration or instance of the `flinksessionjobs` resource.

5\. To list all the `flinksessionjobs` under any namespace
```go

list,err :=client.Resource(schema.GroupVersionResource{  
  Group:    "flink.apache.org",  
  Version:  "v1beta1",  
  Resource: "flinksessionjobs",  
 }).Namespace("namespace").List(context.TODO(), metav1.ListOptions{})
```

This is how one can use Dynamic client to create Custom Resource using the Client-go library

In summary, *“k8s.io/client-go/dynamic”* provides a generic and dynamic way to interact with Kubernetes resources, while *“k8s.io/client-go/kubernetes”* offers a typed and higher-level client for working with specific resource types. The choice between the two depends on your specific requirements and the level of flexibility and type safety you need in your code as specified above for Nginx deployment and FlinkSessionJobs(CR).