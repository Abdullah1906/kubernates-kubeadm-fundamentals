# Kubernetes Production-Grade Learning Guide

A practical, production-oriented Kubernetes learning repository covering **Kubernetes fundamentals, kubeadm cluster setup, workloads, services, HPA, database architecture, external database connectivity, firewall, networking, security, and troubleshooting**.

The goal of this repository is not only to learn `kubectl` commands, but to understand **how Kubernetes works internally and how these concepts are applied in real-world environments**.

---

## 📚 What This Repository Covers

This repository is divided into four major guides:

```text
                    Kubernetes Learning Path
                              │
              ┌───────────────┴───────────────┐
              │                               │
       Cluster Foundation               Kubernetes Core
              │                               │
        kubeadm + containerd         Pod / ReplicaSet / Deployment
        Master + Worker               DaemonSet / StatefulSet
        CNI + Networking              Service / HPA / Probes
              │                               │
              └───────────────┬───────────────┘
                              │
                    Production Architecture
                              │
                  ┌───────────┴───────────┐
                  │                       │
              Database              Networking/Security
                  │                       │
        StatefulSet / PVC          Firewall / NetworkPolicy
        External Database           RBAC / Secrets
        Read Replica                VPC / VPN / Connectivity
```

## kubeadm (Self-Hosted)
```bash
AWS ap-south-1 — devops-vpc (10.0.0.0/16)
┌─────────────────────────────────────────────────────┐
│  Public Subnet (ap-south-1a)                        │
│  ┌──────────────────────────────────────────────┐   │
│  │  Master Node (t3.medium, Ubuntu 22.04)       │   │
│  │  • kube-apiserver   :6443                    │   │
│  │  • etcd             :2379                    │   │
│  │  • kube-scheduler                            │   │
│  │  • kube-controller-manager                   │   │
│  │  • Calico CNI (podCIDR: 192.168.0.0/16)      │   │
│  └──────────────────────────────────────────────┘   │
│                                                     │
│  Private Subnets (1b / 1c)                          │
│  ┌──────────────────────┐  ┌──────────────────────┐ │
│  │  Worker Node 1       │  │  Worker Node 2       │ │
│  │  t3.medium           │  │  t3.medium           │ │
│  │  kubelet + containerd│  │  kubelet + containerd│ │
│  └──────────────────────┘  └──────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

---

# 📁 Repository Structure

Recommended structure:

```text
kubernetes-learning/
│
├── README.md
│
├── 01-kubernetes-cluster-setup/
│   ├── master-init-guide.md
│   └── worker-join-guide.md
│
├── 02-kubernetes-workloads-services/
│   └── kubernetes-workloads-services.md
│
└── 03-kubernetes-database-networking/
    └── kubernetes-database-external-connection.md
```

---

# 1️⃣ Kubernetes Cluster Setup

## `master-init-guide.md`

This guide covers building a Kubernetes cluster from the beginning using **kubeadm**.

### Main topics

* Ubuntu node preparation
* System update
* Swap disabling
* Kernel modules
* Linux `sysctl` configuration
* `containerd` installation
* Systemd cgroup configuration
* `kubeadm`
* `kubelet`
* `kubectl`
* Kubernetes v1.29 repository
* Control-plane initialization
* Kubernetes networking
* Calico CNI
* kubeconfig configuration
* Worker join command generation
* Cluster health verification
* Troubleshooting

### Architecture

```text
                 Kubernetes Cluster
                        │
                ┌───────┴───────┐
                │               │
          Control Plane      Worker Nodes
                │               │
          ┌─────┼─────┐     ┌───┴───┐
          │     │     │     │       │
       API Server etcd Scheduler  kubelet
                                  │
                              containerd
                                  │
                                 Pods
```

The guide also covers connectivity requirements such as the Kubernetes API server on port `6443`, node networking, CNI installation, and common kubeadm failures.

---

# 2️⃣ Worker Node Join

## `worker-join-guide.md`

After the control plane is initialized, this guide explains how to prepare and join worker nodes to the cluster.

### Main topics

* Worker node preparation
* Swap configuration
* Kernel configuration
* containerd
* kubeadm/kubelet/kubectl
* Master connectivity verification
* `kubeadm join`
* Join token
* Discovery CA hash
* Worker node verification
* Calico verification
* Test workload deployment
* Troubleshooting failed joins

### Worker join flow

```text
Worker Node
     │
     ├── Disable Swap
     │
     ├── Configure Kernel
     │
     ├── Install containerd
     │
     ├── Install kubeadm
     │
     ├── Install kubelet
     │
     ├── Install kubectl
     │
     ▼
Check Master:6443
     │
     ▼
kubeadm join
     │
     ▼
Kubernetes Control Plane
     │
     ▼
Worker → Ready
```

The guide also includes practical troubleshooting for:

* Port `6443` connectivity
* Expired kubeadm tokens
* Certificate issues
* `NotReady` workers
* Calico problems
* kubelet problems
* container runtime problems
* incorrect master IP
* previous kubeadm state

---

# 3️⃣ Kubernetes Workloads & Services

## `kubernetes-workloads-services.md`

This guide focuses on the Kubernetes resources used to deploy and operate applications.

### Core Kubernetes resources

```text
Pod
 │
 ▼
ReplicaSet
 │
 ▼
Deployment
 │
 ▼
Service
 │
 ├── ClusterIP
 ├── NodePort
 └── LoadBalancer
```

It also covers stateful and node-level workloads:

```text
Deployment
    │
    └── Stateless Application

DaemonSet
    │
    └── One Pod per eligible Node

StatefulSet
    │
    └── Stable Identity + Persistent Storage
```

### Topics covered

* Pod
* ReplicaSet
* Deployment
* DaemonSet
* StatefulSet
* Service
* ClusterIP
* NodePort
* LoadBalancer
* Kubernetes DNS
* ConfigMap
* Secret
* Resource Requests
* Resource Limits
* Readiness Probe
* Liveness Probe
* Startup Probe
* Rolling Update
* Rollback
* HPA
* PersistentVolume
* PVC
* StorageClass
* PodDisruptionBudget
* NetworkPolicy
* RBAC
* ServiceAccount
* Production architecture
* Kubernetes commands
* Troubleshooting

---

## 🔄 Deployment and ReplicaSet

A very important relationship:

```text
Deployment
     │
     ▼
ReplicaSet
     │
     ├── Pod
     ├── Pod
     └── Pod
```

The Deployment manages application releases and updates, while the ReplicaSet maintains the desired number of Pods.

---

# 4️⃣ Horizontal Pod Autoscaler

HPA allows Kubernetes to automatically adjust the number of application Pods based on resource utilization or other supported metrics.

```text
             HPA
              │
              ▼
         Deployment
              │
       ┌──────┼──────┐
       ▼      ▼      ▼
      Pod    Pod    Pod
```

Example:

```text
Low traffic
     ↓
3 Pods
     ↓
Traffic increases
     ↓
5 Pods
     ↓
Traffic increases again
     ↓
8 Pods
```

When traffic decreases:

```text
8 Pods
   ↓
5 Pods
   ↓
3 Pods
```

The guide explains the relationship between:

```text
HPA
 ↓
Deployment
 ↓
ReplicaSet
 ↓
Pods
```

---

# 5️⃣ Kubernetes Database Architecture

## `kubernetes-database-external-connection.md`

This guide focuses on one of the most important production topics:

> **How should a database be deployed and connected to Kubernetes applications?**

It explains why databases should not be treated exactly like stateless application Pods.

---

## Stateless vs Stateful

### Application

```text
Deployment
    │
    ├── API Pod
    ├── API Pod
    └── API Pod
```

Application Pods can normally be scaled horizontally.

```bash
kubectl scale deployment api --replicas=3
```

### Database

```text
StatefulSet
    │
    ├── db-0 → PVC
    ├── db-1 → PVC
    └── db-2 → PVC
```

A database requires consideration of:

* Persistent storage
* Replication
* Consistency
* Failover
* Backup
* Recovery
* Quorum
* Monitoring

Therefore:

> **StatefulSet replicas do not automatically create a highly available database cluster.**

---

# 6️⃣ External Database

For production systems, the database is often kept outside the Kubernetes cluster.

```text
                    Kubernetes Cluster
                           │
                    ┌──────┴──────┐
                    │             │
                  API Pod       API Pod
                    │             │
                    └──────┬──────┘
                           │
                           │
                           ▼
                    External Database
                           │
                 ┌─────────┴─────────┐
                 │                   │
             SQL Server          PostgreSQL
```

Possible external database environments:

* Dedicated VM
* Physical server
* AWS managed database
* Azure managed database
* Other managed database services

---

# 7️⃣ External Database Connection Methods

The database guide explains three common approaches.

### Method 1 — Secret + Environment Variable

```text
Application Pod
      │
      ▼
Kubernetes Secret
      │
      ▼
Connection String
      │
      ▼
External Database
```

### Method 2 — ExternalName Service

```text
Application
     │
     ▼
sql-external
     │
     ▼
db.mycompany.com
```

### Method 3 — Service + Endpoints

```text
Application
     │
     ▼
sql-external
     │
     ▼
Endpoints
     │
     ▼
192.168.x.x:1433
```

This allows the application to use a stable Kubernetes service name instead of directly depending on a database IP.

---

# 8️⃣ Database Security

The database guide also covers production security principles.

### Credentials

Database passwords should not be hardcoded inside:

```text
Source Code
Deployment YAML
Dockerfile
GitHub Repository
```

Instead:

```text
Kubernetes Secret
       │
       ▼
Application
```

For larger production environments, secrets can be integrated with external secret-management systems.

---

# 9️⃣ Firewall & Network Connectivity

A very important production concept covered in the database guide is:

```text
Firewall ≠ Network Connectivity
```

They solve different problems.

### Firewall

Controls:

> **Who is allowed to connect?**

Example:

```text
Worker Node IP
      │
      ▼
DB Firewall
      │
      ├── Allowed → Port 1433
      │
      └── Others → Blocked
```

### Network Connectivity

Controls:

> **Can the Kubernetes network actually reach the database network?**

Possible solutions:

```text
VPC Peering
      │
      OR
      │
VPN
      │
      OR
      │
Direct Connect
```

---

# 🔐 Production Security Model

A production Kubernetes environment should ideally have multiple security layers.

```text
                    Internet
                       │
                       ▼
                Load Balancer
                       │
                       ▼
                    Ingress
                       │
                       ▼
                   Service
                       │
              ┌────────┴────────┐
              │                 │
              ▼                 ▼
           API Pod           API Pod
              │                 │
              └────────┬────────┘
                       │
                 NetworkPolicy
                       │
                       ▼
                 External DB
                       │
                 DB Firewall
                       │
                       ▼
                Database Server
```

Additional security controls:

```text
RBAC
 │
ServiceAccount
 │
Secrets
 │
NetworkPolicy
 │
Firewall / Security Group
 │
TLS
 │
Database Permissions
```

---

# 🚀 Complete Learning Flow

These four files can be studied in the following order:

```text
                    START
                      │
                      ▼
          1. Kubernetes Cluster Setup
                      │
                      ▼
             kubeadm Control Plane
                      │
                      ▼
              Worker Node Join
                      │
                      ▼
             Calico / Networking
                      │
                      ▼
          2. Kubernetes Workloads
                      │
              ┌───────┴───────┐
              ▼               ▼
          Deployment       DaemonSet
              │
              ▼
          ReplicaSet
              │
              ▼
             Pods
              │
              ▼
           Service
              │
              ▼
             HPA
              │
              ▼
       Production Workload
                      │
                      ▼
          3. Database Architecture
                      │
              ┌───────┴────────┐
              ▼                ▼
        Internal DB        External DB
              │                │
        StatefulSet        VM / Managed DB
              │                │
             PVC              │
              │                │
              └───────┬────────┘
                      ▼
             4. Network & Security
                      │
              ┌───────┼────────┐
              ▼       ▼        ▼
           Secret  Firewall  NetworkPolicy
                      │
                      ▼
               Production
```

---

# 🏗️ Production Kubernetes Mental Model

The main concepts from all four guides can be combined into this architecture:

```text
                         Internet
                            │
                            ▼
                   Cloud Load Balancer
                            │
                            ▼
                         Ingress
                            │
                            ▼
                       API Service
                            │
             ┌──────────────┼──────────────┐
             │              │              │
             ▼              ▼              ▼
          API Pod        API Pod        API Pod
             │              │              │
             └──────────────┼──────────────┘
                            │
                      Deployment
                            │
                        ReplicaSet
                            │
                           HPA
                            │
                            ▼
                  NetworkPolicy / Egress
                            │
                            ▼
                    External Database
                            │
                       DB Firewall
                            │
                            ▼
                     SQL Server / DB
```

Node-level infrastructure:

```text
Worker Node 1
    │
    ├── API Pod
    ├── API Pod
    └── DaemonSet Pod

Worker Node 2
    │
    ├── API Pod
    ├── API Pod
    └── DaemonSet Pod

Worker Node 3
    │
    ├── API Pod
    ├── API Pod
    └── DaemonSet Pod
```

---

# 🧰 Important Kubernetes Commands

Some frequently used commands covered throughout the guides:

```bash
kubectl get nodes
```

```bash
kubectl get pods -o wide
```

```bash
kubectl get deployments
```

```bash
kubectl get replicasets
```

```bash
kubectl get daemonsets
```

```bash
kubectl get statefulsets
```

```bash
kubectl get services
```

```bash
kubectl get endpoints
```

```bash
kubectl get endpointslices
```

```bash
kubectl describe pod <pod-name>
```

```bash
kubectl logs <pod-name>
```

```bash
kubectl rollout status deployment/<deployment-name>
```

```bash
kubectl rollout history deployment/<deployment-name>
```

```bash
kubectl rollout undo deployment/<deployment-name>
```

---

# 📋 Production Checklist

Before considering a Kubernetes application production-ready, review:

```text
[ ] Kubernetes cluster health
[ ] Multiple application replicas
[ ] Deployment strategy
[ ] Resource requests
[ ] Resource limits
[ ] Readiness Probe
[ ] Liveness Probe
[ ] Startup Probe
[ ] HPA
[ ] Rolling Update
[ ] Rollback
[ ] Service
[ ] Ingress / Gateway
[ ] ConfigMap
[ ] Secret
[ ] ServiceAccount
[ ] RBAC
[ ] NetworkPolicy
[ ] PodDisruptionBudget
[ ] PersistentVolume / PVC
[ ] StorageClass
[ ] Database backup
[ ] Database recovery strategy
[ ] Database firewall
[ ] Network connectivity
[ ] TLS
[ ] Monitoring
[ ] Logging
[ ] Metrics
[ ] Disaster Recovery
```

---

# 🎯 Recommended Learning Order

If you are learning Kubernetes from zero, follow this sequence:

```text
1. Pod
      ↓
2. ReplicaSet
      ↓
3. Deployment
      ↓
4. Service
      ↓
5. ClusterIP
      ↓
6. NodePort
      ↓
7. LoadBalancer
      ↓
8. Ingress
      ↓
9. ConfigMap + Secret
      ↓
10. Health Probes
      ↓
11. Resource Requests/Limits
      ↓
12. HPA
      ↓
13. Rolling Update
      ↓
14. Rollback
      ↓
15. DaemonSet
      ↓
16. StatefulSet
      ↓
17. PVC / StorageClass
      ↓
18. PodDisruptionBudget
      ↓
19. NetworkPolicy
      ↓
20. Database Architecture
      ↓
21. External Database
      ↓
22. Firewall & Network Connectivity
      ↓
23. RBAC & Security
      ↓
24. Production Deployment
```

---

# 🧠 Key Takeaways

### Application

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

### Node-level workload

```text
DaemonSet
    ↓
One Pod per eligible Node
```

### Stateful workload

```text
StatefulSet
    ↓
Stable Pod Identity
    ↓
PVC
```

### Networking

```text
Ingress
   ↓
Service
   ↓
Pods
```

### Autoscaling

```text
HPA
 ↓
Deployment
 ↓
ReplicaSet
 ↓
Pods
```

### Database

```text
Application
     ↓
Service / Secret
     ↓
External Database
     ↓
Firewall
```

### Security

```text
RBAC
 +
Secrets
 +
NetworkPolicy
 +
Firewall
 +
Database Permissions
```

---

# 📖 Guide Summary

| Guide                                        | Main Focus                                                                       |
| -------------------------------------------- | -------------------------------------------------------------------------------- |
| `master-init-guide.md`                       | Kubernetes control-plane setup with kubeadm                                      |
| `worker-join-guide.md`                       | Worker preparation and cluster joining                                           |
| `kubernetes-workloads-services.md`           | Pods, ReplicaSet, Deployment, DaemonSet, StatefulSet, Services, HPA              |
| `kubernetes-database-external-connection.md` | Database scaling, external DB, Secrets, NetworkPolicy, Firewall and connectivity |

---

# 👨‍💻 Purpose of This Repository

This repository is intended as a **hands-on Kubernetes reference and learning lab** for developers and DevOps engineers who want to move from basic Kubernetes concepts toward production-oriented infrastructure design.

The focus is on understanding not only **what command to run**, but also:

* Why a Kubernetes resource is used
* How Kubernetes resources interact
* How workloads scale
* How applications communicate
* How stateful workloads differ from stateless workloads
* How databases should be designed around Kubernetes
* How external databases connect to Pods
* How firewall and network routing work
* How secrets and RBAC improve security
* How to troubleshoot real cluster problems
* How these concepts fit into a production architecture

---

## ⭐ Final Mental Model

```text
                 KUBERNETES
                     │
       ┌─────────────┼─────────────┐
       │             │             │
       ▼             ▼             ▼
   Workloads      Networking    Storage
       │             │             │
       ▼             ▼             ▼
 Deployment       Service        PVC
 DaemonSet        Ingress        StorageClass
 StatefulSet
       │
       ▼
    ReplicaSet
       │
       ▼
      Pods
       │
       ▼
      HPA
       │
       ▼
 Production Application
       │
       ▼
 External Database
       │
       ▼
 Firewall + NetworkPolicy
       │
       ▼
 Secure Production Architecture
```

> **Learn the resource. Understand its controller. Understand its networking. Understand its failure mode. Then understand how it behaves in production.**

---

## 📌 Status

**Learning / Practice Repository**

Focused on:

```text
Kubernetes
kubeadm
containerd
Docker
Linux
Networking
DevOps
Cloud Infrastructure
Database Architecture
Production Deployment
```

---

⭐ If this repository helps you understand Kubernetes, consider giving it a star.



### Here is step file for master node and worker node for kubeadm. 
### master-init-guide.md for master node and worker-join-guide.md for worker node.
### class-12
