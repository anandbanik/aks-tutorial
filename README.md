# Azure Kubernetes Services

## Kubernetes

### About Kubernetes
Kubernetes (k8s) is an open-source system for automating deployment, scaling, and management of containerized applications. It groups containers that make up an application into logical units for easy management and discovery.

### Features
1. Start, stop, update, and manage a cluster of machines running containers in a consistent and maintainable way.
2. Particularly suited for horizontally scaleable, stateless, or 'microservices' application architectures.
3. Additional functionality to make containers easier to use in a cluster (reachability and discovery).
4. Kubernetes does NOT and will not expose all of the 'features' of the docker command line.

### Components

1. Master:- It typically consists of
a) kube-apiserver
b) kube-scheduler
c) kube-controller-manager
d) etcd
e) kube-proxy