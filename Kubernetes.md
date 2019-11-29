# Kubernetes

## Core Concepts

### Cluster Architecture Overview

There are 2 types of nodes:
* _Worker Nodes_: they just run containers
  * _container-runtime-engine_: For example docker, containerd, rocket.
  * _kubelet_: Responsible of managing everything in the node, report status, create containers, etc.
  * _kube-proxy_: responsible of the management of the communications for every container to be able to communicate with others containers.
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