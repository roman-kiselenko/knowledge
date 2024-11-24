### Common kubectl command

```query
content:kubectl path:Stuff
```

#### Cluster Information
With the following command, we can get information about the cluster, like the version of Kubernetes that is running, the IP address of the master, and the names of the nodes in the cluster.

```bash
kubectl cluster-info
```

#### Resource Management
The following commands are used to manage resources in the cluster

```bash
kubectl get nodes # list all nodes in the cluster
kubectl get pods # list all pods in the namespace or with --all-namespaces flag
kubectl get services
kubectl get deployments
kubectl get secrets
kubectl edit <resource-type> <resource-name> # edit a resource in the default editor
watch -n 2 kubectl get pods  # Continuously monitor pod status every X seconds
```

Flags:
```bash
--all-namespaces # list the requested object(s) across all namespaces
--watch # after listing/getting the requested object, watch for changes
-o # output format example -o yaml or -o json
--show-labels # show labels in the last column
--selector # filter results by label selector example --selector env=prod
--sort-by # sort list types using a jsonpath expression example --sort-by='{.metadata.name}'
--field-selector # filter results by a field selector example --field-selector metadata.name=nginx
--no-headers # when using the default or custom-column output format, don't print headers (default print headers) 
--output-watch-events # output watch event objects when --watch or --watch-only is used example kube get pods --watch --output-watch-events
```

#### Inspecting Resources
The following commands are used to inspect resources in the cluster.
Also we have included the `kubectl explain` command which is used to get the documentation of a resource type.


```bash
kubectl describe nodes <node-name>
kubectl describe pods <pod-name>
kubectl describe services <service-name>
kubectl describe deployments <deployment-name>
kubectl explain <resource-type> # get the documentation of a resource type example kubectl explain pods
```

#### Creating and Updating Resources
With the following commands, we can create and update resources in the cluster.

```bash
kubectl create -f <file-name>
kubectl apply -f <file-name>
```

#### Deleting Resources
With the following commands, we can delete resources in the cluster. 

```bash
kubectl delete pods <pod-name>
kubectl delete services <service-name>
kubectl delete -f <file-name>
kubectl delete deployment <deployment-name> # delete a deployment
kubectl delete namespace <namespace-name> # delete a namespace
```

#### Scaling Deployments
With the following command, we can scale your deployments:

```bash
kubectl scale deployment <deployment-name> --replicas=<number-of-replicas>
```

#### Exposing Deployments

In this section, we'll learn how to expose Kubernetes deployments using kubectl with different service types: ClusterIP, NodePort, and LoadBalancer. 
Exposing a deployment allows us to make the application accessible to users or other services, either within the cluster or externally.

```bash
kubectl expose deployment <deployment-name> --type=ClusterIP --port=<port> # Exposes the service on a cluster-internal IP. Choosing this value makes the service only reachable from within the cluster.
kubectl expose deployment <deployment-name> --type=NodePort --port=<port>
kubectl expose deployment <deployment-name> --type=LoadBalancer --port=<port> # In cloud providers that support load balancers, an external IP address would be provisioned to access the service.
```

The loadbalancer service type is only supported in cloud providers that support load balancers.

#### Managing Rollouts
With the following commands, we can manage the rollouts of our deployments. 
Rollouts are an essential part of the deployment process, as they allow you to update your applications while minimizing downtime and ensuring  transitions between versions.

```bash
kubectl rollout status deployment <deployment-name> # Check the status of a rollout
kubectl rollout history deployment <deployment-name> # Check the history of a rollout
kubectl rollout undo deployment <deployment-name> # Rollback to the previous revision
kubectl rollout undo deployment <deployment-name> --to-revision=<revision-number> # Rollback to a specific revision
```

#### Working with Logs
With the following commands, we can work with the logs of our pods:

```bash
kubectl logs <pod-name> -n <namespace> # print the logs for a pod in a namespace
kubectl logs -f <pod-name> # follow the logs
kubectl logs -p <pod-name> # print the logs for the previous instance of the container in a pod if it exists
```

#### Executing Commands in Containers
Sometimes we need to execute commands in a container:

```bash
kubectl exec <pod-name> -- <command> # execute a command in a container
kubectl exec -it <pod-name> -- <command> # execute a command in a container interactively
# enter the container's shell
kubectl exec -it <pod-name> -- /bin/bash # bash shell, very common to get into a container
# copy files to and from a container
kubectl cp <pod-name>:<path-to-file> <path-to-local-file> -c <container-name>
kubectl cp <path-to-local-file> <pod-name>:<path-to-file> -c <container-name>
```

#### Port Forwarding
With the following command, we can forward a local port to a port on a pod or service.
This is very handy if the application is not exposed to the outside world, but you still want to access it.


```bash
kubectl port-forward <pod-name> <local-port>:<container-port>
kubectl port-forward <service-name> <local-port>:<container-port>
```

#### Labeling and Annotating Resources
With the following commands, we can add labels and annotations to resources in the cluster.
This is useful for organizing and grouping resources, as well as for attaching metadata to resources.

```bash
kubectl label pods <pod-name> <label-key>=<label-value>
kubectl annotate pods <pod-name> <annotation-key>=<annotation-value>
```

#### Taints and Tolerations
Taints and tolerations are a way to control how pods are scheduled on nodes. 
For example, we can use taints to prevent pods from being scheduled on nodes that have sensitive data.

```bash
kubectl taint nodes <node-name> <key>=<value>:<effect> # taint a node
kubectl taint nodes <node-name> <key>:<effect> # remove a taint from a node
kubectl taint nodes <node-name> <key>:NoSchedule # prevent pods from being scheduled on the node
kubectl taint nodes <node-name> <key>:NoExecute # prevent pods from being scheduled on the node and evict existing pods
kubectl taint nodes <node-name> <key>:PreferNoSchedule # prefer not to schedule pods on the node
```

Tolerations are applied to pods, and allow (but do not require) the pods to schedule onto nodes with matching taints.

Example pod spec with tolerations:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
    containers:
    - name: nginx
        image: nginx
    tolerations:
    - key: "key"
        operator: "Equal"
        value: "value"
        effect: "NoSchedule"
```

#### Configuring Kubectl

```bash
kubectl config view # display merged kubeconfig settings or a specified kubeconfig file
kubectl config view --minify --output 'jsonpath={..namespace}' # display the namespace within the context
kubectl config current-context # display the current-context
kubectl config use-context <context-name> # set the current-context in a kubeconfig file
kubectl config set-context <context-name> # set a context entry in kubeconfig
kubectl config set-cluster <cluster-name> # set a cluster entry in kubeconfig
kubectl config set-credentials <user-name> # set a user entry in kubeconfig
kubectl config unset users.<name> # unset a user entry in kubeconfig
```

#### set a cluster entry in kubeconfig
```bash
kubectl config set-cluster <cluster-name> \
  --certificate-authority=<path-to-ca-file> \
  --embed-certs=<true|false> \
  --server=<address-and-port-of-api-server> \
  --kubeconfig=<path-to-kubeconfig-file>
```

#### Dry Run
Dry run allows you to preview the changes that will be made to the cluster without actually making them. This is useful for debugging and testing.


```bash
kubectl apply --dry-run=client -f <file-name>
```

#### Looping Through Resources
Sometimes you need to loop through resources. For example, you want to delete all pods in a namespace. we can use the following commands to do that.

```bash
kubectl get pods -o name | cut -d/ -f2 | xargs -I {} kubectl delete pod {}
for pod in $(kubectl get pods -o name | cut -d/ -f2); do kubectl delete pod $pod; done
```

Loop through secret data items and print them out

```bash
 kubectl get secrets -o json | jq -r '.items[] | .metadata.name as $name | .data | to_entries[] | "\($name) \(.key): \((.value|@base64d))"'
```

#### Create Resources with `cat` command

With `cat` we can create resources from stdin 

```bash
cat <<EOF | kubectl create -f -
# create a n nginx pod
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
    containers:
    - name: nginx
        image: nginx
EOF
```

#### Getting API Resources

```bash
kubectl api-resources
kubectl api-resources --namespaced=true # namespaced resources
# how to get curl call from kubectl with log level 9
kubectl get pods -v=9  2>&1 | grep curl # get curl call for a specific resource
kubectl get pods -v=9  2>&1 | grep curl | sed 's/.*curl -k -v -XGET/https/' | sed 's/ -H .*//' # get curl call for a specific resource

```

#### Flattening Kubeconfig Files

Assume that you have multiple kubeconfig files and you want to merge them into one file. You can use the following command to do that.

```bash
KUBECONFIG=~/.kube/config:~/.kube/config2:~/.kube/config3 kubectl config view --flatten > ~/.kube/merged-config
```
then you can run the following command to use the merged kubeconfig file:

```bash
export KUBECONFIG=~/.kube/merged-config
```

#### Mount file to pod

создать config map `gitconfig.yaml`

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
	name: gitconfig-renovate-config
	data:   
		.gitconfig: |
		     .... config here ...
```

Create configMap

```shell
kubectl create configmap gitconfig-renovate-config --from-file=gitconfig.yaml
```

потом замаунтить его

```yaml
spec:
      containers:
        .....
        volumeMounts:
        - name: gitconfig-renovate-config
          mountPath: /home/ubuntu/.gitconfig
          subPath: .gitconfig
      volumes:
        - name: gitconfig-renovate-config
          configMap:
            name: gitconfig-renovate-config
            items:
              - key: .gitconfig
                path: .gitconfig
```

#### ConfigMap to Pod

Create a file **example.env**

```shell
cat << EOF > example.env
FOO=foo
BAR=true
EOF
```

Now create a `Kubernetes` config map object:

```shell
kubectl create configmap plainecho-config --from-env-file=./example.env
```

Add to workload

```yaml
spec:
      containers:
      - name: plainecho-app
        image: stefanprodan/podinfo:latest
        resources:
          requests:
            cpu: "100m"
        imagePullPolicy: IfNotPresent
        env:
        - name: FOO
          valueFrom:
            configMapKeyRef:
              name: plainecho-config
              key: FOO
        - name: BAR
          valueFrom:
            configMapKeyRef:
              name: plainecho-config
              key: BAR
```
#### Change component's log level
```yaml
# create cluster role and rolebinding
cat << EOT | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: edit-debug-flags-v
rules:
- apiGroups:
  - ""
  resources:
  - nodes/proxy
  verbs:
  - update
- nonResourceURLs:
  - /debug/flags/v
  verbs:
  - put
EOT
```
```yaml
cat << EOT | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: edit-debug-flags-v
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: edit-debug-flags-v
subjects:
- kind: ServiceAccount
  name: default
  namespace: default
EOT
```

```shell
# pord forward
kubectl -n kube-system port-forward kube-scheduler-kind-control-plane 10259:10259

# gen token
TOKEN=$(kubectl create token default)
curl -s -X PUT -d '5' https://localhost:10259/debug/flags/v --header "Authorization: Bearer $TOKEN" -k
```
#### DNS debug
```sh
kubectl exec -it dnsutils -- nslookup nginx.default 10.10.235.148
```
#### Stress test pod
Create pod
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-demo
  namespace: stress-test
spec:
  containers:
  - name: pod-demo
    image: polinux/stress
    resources:
      limits:
        memory: "600Mi"
        cpu: 0.2
      requests:
        memory: "600Mi"
        cpu: 0.1
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "256M", "-c", "2", "--vm-hang", "1"]
```

Check cgroup configuration

```sh
➜  ~ kubectl get pods -n stress-test  -oyaml | grep uid
    uid: e5090cc3-8d82-4bd2-9213-7e8df96cfe9b
➜  ~
➜  ~ cat /sys/fs/cgroup/kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b/cpu.max
20000 100000
➜  ~ systemd-cgtop
CGroup                                                                 Tasks   %CPU   Memory  Input/s Output/s
kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b             5       79.5   257.2M        -        -
```

Resize to increase the CPU limit:

```sh
➜  ~ kubectl -n stress-test patch pod pod-demo --patch '{"spec":{"containers":[{"name":"pod-demo", "resources":{"requests":{"cpu":"100m","memory":"600Mi"} ,"limits":{"cpu":"800m", "memory":"600Mi"}}}]}}'
pod/pod-demo patched
➜  ~ cat /sys/fs/cgroup/kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b/cpu.max
80000 100000
➜  ~ systemd-cgtop
CGroup                                                                 Tasks   %CPU   Memory  Input/s Output/s
kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b             5       79.5   257.2M        -        -
```

Resize to decrease the CPU limit:

```sh
➜  ~ kubectl -n stress-test patch pod pod-demo --patch '{"spec":{"containers":[{"name":"pod-demo", "resources":{"requests":{"cpu":"100m","memory":"600Mi"} ,"limits":{"cpu":"200m", "memory":"600Mi"}}}]}}'
pod/pod-demo patched
➜  ~ cat /sys/fs/cgroup/kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b/cpu.max
20000 100000
➜  ~ systemd-cgtop
CGroup                                                               Tasks   %CPU   Memory  Input/s Output/s
kubepods/burstable/pode5090cc3-8d82-4bd2-9213-7e8df96cfe9b           5       19.4   257.4M        -        -
```