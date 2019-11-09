# Kubernetes

## Core Concepts

### Cluster Architecture Overview

There are 2 types of nodes:
* _Worker Nodes_: they just run containers
** _container-runtime-engine_: For example docker, containerd, rocket.
** _kubelet_: Responsible of managing everything in the node, report status, create containers, etc.
** _kube-proxy_: responsible of the management of the communications for every container to be able to communicate with others containers.
* _Master Nodes_: control plane of Kubernetes cluster. Contains:
** _ETCD_: KV store
** _kube-scheduler_: Entity that decides which container has to run where.
** _node-controller_: manages nodes, status of the nodes, adding new ones, etc.
** _replication-controller_: responsible of the number of containers of each cluster, etc.
** _kube-apiserver_: exposes the API of k8s that is used by the rest of the components.

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




## Command reference

### Get system PODs
````
kubectl get pods -n kube-system
NAMESPACE NAME
kube-system coredns-78fcdf6894-prwvl kube-system coredns-78fcdf6894-vqd9w kube-         -master
kube-system kube-apiserver-master kube-system kube-controller-manager-master kube-system kube-proxy-f6k26
kube-system kube-proxy-hnzsw
kube-system kube-scheduler-master kube-system weave-net-924k8
kube-system weave-net-hzfcz
READY STATUS RESTARTS AGE 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 1/1 Running 0 1h 2/2 Running 1 1h 2/2 Running 1 1h
````
### Execute command on POD
````
kubectl exec etcd-master –n kube-system etcdctl get / --prefix –keys-only
/registry/apiregistration.k8s.io/apiservices/v1. /registry/apiregistration.k8s.io/apiservices/v1.apps /registry/apiregistration.k8s.io/apiservices/v1.authentication.k8s.io /registry/apiregistration.k8s.io/apiservices/v1.authorization.k8s.io /registry/apiregistration.k8s.io/apiservices/v1.autoscaling /registry/apiregistration.k8s.io/apiservices/v1.batch /registry/apiregistration.k8s.io/apiservices/v1.networking.k8s.io /registry/apiregistration.k8s.io/apiservices/v1.rbac.authorization.k8s.io /registry/apiregistration.k8s.io/apiservices/v1.storage.k8s.io /registry/apiregistration.k8s.io/apiservices/v1beta1.admissionregistration.k8s.io
 
````


