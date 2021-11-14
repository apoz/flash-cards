# Kubernetes
https://github.com/kodekloudhub/certified-kubernetes-administrator-course

## Core Concepts

### Cluster Architecture Overview

There are 2 types of nodes:
* _Worker Nodes_: they just run containers
  * _container-runtime-engine_: For example docker, containerd, rocket.
  * _kubelet_: Responsible of managing everything in the node, report status, create containers, etc.
  * _kube-proxy service_: responsible of the management of the communications for every container to be able to communicate with others containers.
* _Master Nodes_: control plane of Kubernetes cluster. Contains:
  * _ETCD_: KV store
  * _kube-scheduler_: Entity that decides which container has to run where.
  * _node-controller_: manages nodes, status of the nodes, adding new ones, etc.
  * _replication-controller_: responsible of the number of containers of each cluster, etc.
  * _kube-apiserver_: exposes the API of k8s that is used by the rest of the components.

### ETCD

Distributed reliable key-value store that is simple, secure and fast. Written in Go.

#### How to test it
[Instructions](https://github.com/etcd-io/etcd/releases)
````
ETCD_VER=v3.4.3

# choose either URL
GOOGLE_URL=https://storage.googleapis.com/etcd
GITHUB_URL=https://github.com/etcd-io/etcd/releases/download
DOWNLOAD_URL=${GOOGLE_URL}

rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
rm -rf /tmp/etcd-download-test && mkdir -p /tmp/etcd-download-test

curl -L ${DOWNLOAD_URL}/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz -o /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz
tar xzvf /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz -C /tmp/etcd-download-test --strip-components=1
rm -f /tmp/etcd-${ETCD_VER}-linux-amd64.tar.gz

/tmp/etcd-download-test/etcd --version
/tmp/etcd-download-test/etcdctl version
````

Setting a new k-v data
````
etcdctl set key1 value1
````

Getting a key data
````
etcdctl get key1
````
#### ETCD in Kubernetes

Stores information of the cluster:
* Nodes
* PODs
* Configs
* Secrets
* Accounts
* Roles
* Bindings
* Others

Everty info obtained from the cluster or every change made in the cluster (config, etc.) is set in ETCD.
Everything is stored under _/registry_.

Example of configuration:
````
ExecStart=/usr/local/bin/etcd \\
--name ${ETCD_NAME} \\ --cert-file=/etc/etcd/kubernetes.pem \\ --key-file=/etc/etcd/kubernetes-key.pem \\ --peer-cert-file=/etc/etcd/kubernetes.pem \\ --peer-key-file=/etc/etcd/kubernetes-key.pem \\ --trusted-ca-file=/etc/etcd/ca.pem \\ --peer-trusted-ca-file=/etc/etcd/ca.pem \\ --peer-client-cert-auth \\
--client-cert-auth \\
--initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
--listen-peer-urls https://${INTERNAL_IP}:2380 \\
--listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
--advertise-client-urls https://${INTERNAL_IP}:2379 \\
--initial-cluster-token etcd-cluster-0 \\
--initial-cluster controller-0=https://${CONTROLLER0_IP}:2380,controller-1=https://${CONTROLLER1_IP}:2380 \\ --initial-cluster-state new \\
--data-dir=/var/lib/etcd
````
Important options:
* _advertise-client-urls_: address in which ETCD listens. Default port is 2379. This is the address that has to be configured in the kube-apiserver to reach the ETCD.
* _initial-cluster_: Is where the diferent ETCD controllers are running in an HA environment.

### kube-apiserver

It's the main control component in Kubernetes. Any kubectl comand we execute, is authenticated and validated by the kube-apiserver, it retrieves the apropriate info from ETCD and then shows the info back to the user.

It's not necesary to use the `kubectl` command, we can use the API directly with curl or some other tool.
Example of sequence of events when creating a pod:
1. kube-apiserver writes the info in ETCD but it does not assign the pod to any node.
2. kube-scheduler is monitoring continously the apiserver, and realizes there's a pod without node. It selects one node, and communicates it back to the apiserver, who updates the info in the ETCD cluster.
3. The apiserver then passes the info to the kubelet in the worker node.
4. The kubelet deploys the container in the worker node.
5. The kubelet informs the apiserver that the pod is up and running.
6. The apiserver then updates the info in the ETCD node.

The kube-apiserver is in the center of any change that has to be done in the system.

Example of configuration:
````
[Service]
 ExecStart=/usr/local/bin/kube-apiserver \\
   --advertise-address=${INTERNAL_IP} \\
   --allow-privileged=true \\
   --apiserver-count=3 \\
   --audit-log-maxage=30 \\
   --audit-log-maxbackup=3 \\
   --audit-log-maxsize=100 \\
   --audit-log-path=/var/log/audit.log \\
   --authorization-mode=Node,RBAC \\
   --bind-address=0.0.0.0 \\
   --client-ca-file=/var/lib/kubernetes/ca.pem \\
   --enable-admission-plugins=Initializers,NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,Defa ultStorageClass,ResourceQuota \\
   --enable-swagger-ui=true \\ 
   --etcd-cafile=/var/lib/kubernetes/ca.pem \\
   --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
   --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\ 
   --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\ --event-ttl=1h \\ --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
   --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\ 
   --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
````

### kube-controller-manager

Its responsibilities:
1. Continously monitors various components within the system.
2. Executes the appropriate actions, through the apiserver, to the system to the desired status.

#### node controller

It monitors the status of the nodes every 5 seconds (node monitor period).
If some node does not send the keepalive, it's then set in a grace period of 40s (node monitor grace period).
It is then provided with 5 mins to come back (POD eviction timeout), if it does not comeback in those 5 mins, the pods are reassigned to other node.

#### replication controller / replica set

It is responsible of monitoring the replica sets. It's responsible of monitoring the amount of pods in a replica set and keep them in the desired status.
It is also reponsible for load balancing and automatic scallability of the pods when demand increases/decreases.

Replication controller is a technology that is being replaced by replica set.
To create a replication controller, we have to create a yaml file with the mandatory elements:
* apiVersion: v1
* kind: ReplicationController
* metadata:
** name: Name of the RC
** Tags:
*** app:
*** type:
* spec:
** template:  Template of the POD that will be managed by this replication controller. This section has to contain the metadata and spec sections of pod definition yaml.
** replicas: number of replicas that have to be running.

To create a replica set, we have to create a yaml file with the mandatory elements:
* apiVersion: apps/v1
* kind: ReplicaSet
* metadata:
** name: Name of the RC
** Tags:
*** app:
*** type:
* spec:
** template:  Template of the POD that will be managed by this replica set. This section has to contain the metadata and spec sections of pod definition yaml.
** replicas: number of replicas that have to be running.
** selector: Replica set can manage pods that were not created by the replica set definition. 
*** matchLabels: List of labels of the pods that will be managed by the replica set.





There are other controllers, all are grouped in a executable called kube-controller-manager.

### kube scheduler

It is responsible for deciding which pod goes in which node depending on certain criteria. It is NOT responsible of placing the pod in the node, that's kubelet work.

The scheduler tries to find the best node for it, in 2 phases:
1. Filter nodes that are NOT able to run the pod.
2. Then it ranks the nodes to see what's the best node to run the pod.

### kubelet

It's the one responsible to register the node in the k8s cluster. It also creates pod. It monitors its node and reports the status to the kubeapi.

### kube-proxy

In a kubernetes system, every pod is able to contact any other pod. That is achieved by means of a POD network, to which all the pods connect to.
There are many solutions to deploy such network.

It's a process that runs in each node and its responsibility is to monitor for new services and set the appropriate rules/configurations for the pods in that node to be able to communicate to those services (with iptables rules for example).

### PODs

We assume the app is already build and is available in a docker hub and the k8s cluster is up and running.

The containers we want to deploy are encapsulated in a kubernetes object called POD. A POD is a single instance of an application. It's the smallest object you can create in Kubernetes.

Usually there's a 1 to 1 correspondance between containers and PODs. To escalate horizontally you deploy more pods, NOT containers inside some POD.
A single POD can have multiple containers, but they will be of different type(helpers, etc.). They share the same network space, so they can communicate with each other as localhost. They can also easily share storage.

They can be created using a YAML configuration file.
Any k8s definition file (YAML) has the following top level (required) fields:
* *apiVersion*: Version of the k8s api you are using to create the object. Example: v1
* *kind*: Type of object we're trying to create. Example: Pod.
* *metadata*: Data about the object in the form of a dictionary.
  * It contains name, labels, etc
* *spec*: Specification section. Additional information for the object. It is also a dict.
  * For example: containers that is an array with pairs name, image

## Command reference

### Get system nodes
````
kubectl get nodes
NAME      STATUS     ROLES    AGE     VERSION
master    Ready      master   20m      v1.11.3
node01    Ready      <none>   20m      v1.11.3
````

### Run POD
````
kubectl run nginx --image nginx
````

### Get information about a pod
````
kubectl describe pod myapp-pod
````

### Get PODs
````
kubectl get pods

Params : 

-o json | yaml | wide
--selector
-- sort-by
````

### Get system PODs
````
kubectl get pods -n kube-system
NAMESPACE NAME READY STATUS RESTARTS AGE
kube-system coredns-78fcdf6894-prwvl 1/1 Running 0 1h
kube-system coredns-78fcdf6894-vqd9w 1/1 Running 0 1h
kube-system kube-apiserver-master 1/1 Running 0 1h
kube-system kube-controller-manager-master 1/1 Running 0 1h
kube-system kube-proxy-f6k26 1/1 Running 0 1h
kube-system kube-proxy-hnzsw 1/1 Running 0 1h
kube-system kube-scheduler-master 1/1 Running 0 1h
kube-system weave-net-924k8 1/1 Running 0 1h
kube-system weave-net-hzfcz 1/1 Running 0 1h
````
### Execute command on POD
````
kubectl exec etcd-master –n kube-system etcdctl get / --prefix –keys-only
/registry/apiregistration.k8s.io/apiservices/v1. 
/registry/apiregistration.k8s.io/apiservices/v1.apps 
/registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io 
/registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io 
/registry/apiregistration.k8s.io/apiservices/v1.autoscaling 
/registry/apiregistration.k8s.io/apiservices/v1.batch 
/registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io
/registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
 
````

### Create object in k8s
````
kubectl create -f pod-definition.yml
````

### Get replication controllers
````
kubectl get replicationcontroller
````

### Get replica set
````
kubectl get replicaset
````

### Update object
````
kubectl update -f replica-set-definition.yml
````

### Scale replica set
````
kubectl scale --replicas=6 -f replica-set-definition.yml

or

kubectl scale --replicas=6 -f object_type  object_name
````


### Drain a node

```
kubectl drain <node name> --ignore-daemonsets --force

#Uncordon the node to allow new pods to be scheduled there again
kubectl uncordon <node name>
```

### Api resources

```
kubectl api-resources
```

### Getting a sample of a yaml file
```
kubectl create deployment my-deployment --image=nginx --dry-run=client -o yaml
```


## Kubernetes concepts
### ConfigMap

A ConfigMap is an API object used to store non-confidential data in key-value pairs. Pods can consume ConfigMaps as environment variables, command-line arguments, or as configuration files in a volume.

A ConfigMap allows you to decouple environment-specific configuration from your container images, so that your applications are easily portable.

```
 apiVersion: v1
kind: ConfigMap
metadata:
  name: game-demo
data:
  # property-like keys; each key maps to a simple value
  player_initial_lives: "3"
  ui_properties_file_name: "user-interface.properties"

  # file-like keys
  game.properties: |
    enemy.types=aliens,monsters
    player.maximum-lives=5    
  user-interface.properties: |
    color.good=purple
    color.bad=yellow
    allow.textmode=true
immutable: true    # Not variables, better performance as kubelet does not have to check for updates periodically
```


There are four different ways that you can use a ConfigMap to configure a container inside a Pod:

- Inside a container command and args
- Environment variables for a container
- Add a file in read-only volume, for the application to read
- Write code to run inside the Pod that uses the Kubernetes API to read a ConfigMap

Example of pod making use of configMap:

```
apiVersion: v1
kind: Pod
metadata:
  name: configmap-demo-pod
spec:
  containers:
    - name: demo
      image: alpine
      command: ["sleep", "3600"]
      env:
        # Define the environment variable
        - name: PLAYER_INITIAL_LIVES # Notice that the case is different here
                                     # from the key name in the ConfigMap.
          valueFrom:
            configMapKeyRef:
              name: game-demo           # The ConfigMap this value comes from.
              key: player_initial_lives # The key to fetch.
        - name: UI_PROPERTIES_FILE_NAME
          valueFrom:
            configMapKeyRef:
              name: game-demo
              key: ui_properties_file_name
      volumeMounts:
      - name: config
        mountPath: "/config"
        readOnly: true
  volumes:
    # You set volumes at the Pod level, then mount them into containers inside that Pod
    - name: config
      configMap:
        # Provide the name of the ConfigMap you want to mount.
        name: game-demo
        # An array of keys from the ConfigMap to create as files
        items:
        - key: "game.properties"
          path: "game.properties"
        - key: "user-interface.properties"
          path: "user-interface.properties"

```

### Secret
A Secret is an object that contains a small amount of sensitive data such as a password, a token, or a key. Such information might otherwise be put in a Pod specification or in a container image. Using a Secret means that you don't need to include confidential data in your application code.

Secrets are similar to ConfigMaps but are specifically intended to hold confidential data.

Kubernetes Secrets are, by default, stored unencrypted in the API server's underlying data store (etcd). Anyone with API access can retrieve or modify a Secret, and so can anyone with access to etcd. Additionally, anyone who is authorized to create a Pod in a namespace can use that access to read any Secret in that namespace; this includes indirect access such as the ability to create a Deployment.


- Enable Encryption at Rest for Secrets.
- Enable or configure RBAC rules that restrict reading data in Secrets (including via indirect means).
- Where appropriate, also use mechanisms such as RBAC to limit which principals are allowed to create new Secrets or replace existing ones.

Pod in three ways:
- As files in a volume mounted on one or more of its containers.
- As container environment variable.
- By the kubelet when pulling images for the Pod.


```
apiVersion: v1
kind: Secret
metadata:
  name: secret-tls
type: kubernetes.io/tls
data:
  # the data is abbreviated in this example HAS to be base64 encoded. We can use stringData if we want normal encoding.
  tls.crt: |
        MIIC2DCCAcCgAwIBAgIBATANBgkqh ...
  tls.key: |
        MIIEpgIBAAKCAQEA7yn3bRHQ5FHMQ ...
```

Example of pod making use of a secret:

```
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret

# OR if we want to specify paths
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: mypod
    image: redis
    volumeMounts:
    - name: foo
      mountPath: "/etc/foo"
      readOnly: true
  volumes:
  - name: foo
    secret:
      secretName: mysecret
      items:
      - key: username
        path: my-group/my-username
        
# OR using them as environment variables
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-pod
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_USERNAME
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: username
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
  restartPolicy: Never
```

### Pod
Pods are the smallest deployable units of computing that you can create and manage in Kubernetes.

A Pod (as in a pod of whales or pea pod) is a group of one or more containers, with shared storage and network resources, and a specification for how to run the containers. A Pod's contents are always co-located and co-scheduled, and run in a shared context. 

The shared context of a Pod is a set of Linux namespaces, cgroups, and potentially other facets of isolation - the same things that isolate a Docker container. Within a Pod's context, the individual applications may have further sub-isolations applied.

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.14.2
    ports:
    - containerPort: 80
```

Pods in a Kubernetes cluster are used in two main ways:

- *Pods that run a single container*. The "one-container-per-Pod" model is the most common Kubernetes use case; in this case, you can think of a Pod as a wrapper around a single container; Kubernetes manages Pods rather than managing the containers directly.

- *Pods that run multiple containers that need to work together*. A Pod can encapsulate an application composed of multiple co-located containers that are tightly coupled and need to share resources. These co-located containers form a single cohesive unit of service—for example, one container serving data stored in a shared volume to the public, while a separate sidecar container refreshes or updates those files. The Pod wraps these containers, storage resources, and an ephemeral network identity together as a single unit.


Pods natively provide two kinds of shared resources for their constituent containers: networking and storage.

Pods and controllers

You can use workload resources to create and manage multiple Pods for you. A controller for the resource handles replication and rollout and automatic healing in case of Pod failure. For example, if a Node fails, a controller notices that Pods on that Node have stopped working and creates a replacement Pod. The scheduler places the replacement Pod onto a healthy Node.

Here are some examples of workload resources that manage one or more Pods:

- Deployment
- StatefulSet
- DaemonSet

#### Resource management in Pods

[here](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)

When you specify a Pod, you can optionally specify how much of each resource a Container needs. The most common resources to specify are CPU and memory (RAM); there are others.

When you specify the resource request for Containers in a Pod, the scheduler uses this information to decide which node to place the Pod on. When you specify a resource limit for a Container, the kubelet enforces those limits so that the running container is not allowed to use more of that resource than the limit you set. The kubelet also reserves at least the request amount of that system resource specifically for that container to use.

If the node where a Pod is running has enough of a resource available, it's possible (and allowed) for a container to use more resource than its request for that resource specifies. However, a container is not allowed to use more than its resource limit.

For example, if you set a memory request of 256 MiB for a container, and that container is in a Pod scheduled to a Node with 8GiB of memory and no other Pods, then the container can try to use more RAM.

If you set a memory limit of 4GiB for that Container, the kubelet (and container runtime) enforce the limit. The runtime prevents the container from using more than the configured resource limit. For example: when a process in the container tries to consume more than the allowed amount of memory, the system kernel terminates the process that attempted the allocation, with an out of memory (OOM) error.

Limits can be implemented either reactively (the system intervenes once it sees a violation) or by enforcement (the system prevents the container from ever exceeding the limit). Different runtimes can have different ways to implement the same restrictions.

Each Container of a Pod can specify one or more of the following:

- spec.containers[].resources.limits.cpu
- spec.containers[].resources.limits.memory
- spec.containers[].resources.limits.hugepages-<size>
- spec.containers[].resources.requests.cpu
- spec.containers[].resources.requests.memory
- spec.containers[].resources.requests.hugepages-<size>

#### Container Probes
 [here](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#container-probes)
 
 A Probe is a diagnostic performed periodically by the kubelet on a Container. To perform a diagnostic, the kubelet calls a Handler implemented by the container. There are three types of handlers:

- *ExecAction*: Executes a specified command inside the container. The diagnostic is considered successful if the command exits with a status code of 0.
- *TCPSocketAction*: Performs a TCP check against the Pod's IP address on a specified port. The diagnostic is considered successful if the port is open.
- *HTTPGetAction*: Performs an HTTP GET request against the Pod's IP address on a specified port and path. The diagnostic is considered successful if the response has a status code greater than or equal to 200 and less than 400.

Each probe has one of three results:
- **Success**: The container passed the diagnostic.
- **Failure**: The container failed the diagnostic.
- **Unknown**: The diagnostic failed, so no action should be taken.

 The kubelet can optionally perform and react to three kinds of probes on running containers:
- **livenessProbe**: Indicates whether the container is running. If the liveness probe fails, the kubelet kills the container, and the container is subjected to its restart policy. If a Container does not provide a liveness probe, the default state is Success.
- **readinessProbe**: Indicates whether the container is ready to respond to requests. If the readiness probe fails, the endpoints controller removes the Pod's IP address from the endpoints of all Services that match the Pod. The default state of readiness before the initial delay is Failure. If a Container does not provide a readiness probe, the default state is Success.
- **startupProbe**: Indicates whether the application within the container is started. All other probes are disabled if a startup probe is provided, until it succeeds. If the startup probe fails, the kubelet kills the container, and the container is subjected to its restart policy. If a Container does not provide a startup probe, the default state is Success.

 #### Restart Policies
 [here](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy)
 
 The spec of a Pod has a restartPolicy field with possible values Always, OnFailure, and Never. The default value is Always.

The restartPolicy applies to all containers in the Pod. restartPolicy only refers to restarts of the containers by the kubelet on the same node.
 
 #### Init containers
 [here] (https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)

 #### Scheduling pods
 
 [sched documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/kube-scheduler/)
 
 [nodeSelector documentation](https://kubernetes.io/docs/concepts/scheduling-eviction/assign-pod-node/)
 
 We can set flags we can specify for the kube-scheduler.
 
 Or we can use a nodeName to specify the node directly:
 ```
 apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
  nodeName: kube-01
 ```
 
 Command to manage label:
 ```
 kubectl label nodes nodename  special=true
 ```
### daemonSet
 
 [documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
