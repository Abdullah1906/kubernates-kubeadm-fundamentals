# Kubernetes Workloads & Services — বাংলা Production Grade Guide

এই গাইডে Kubernetes-এর গুরুত্বপূর্ণ Workload ও Networking resource সহজ ভাষায় এবং production-grade perspective থেকে আলোচনা করা হয়েছে।

যেগুলো এখানে শেখানো হবে:

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
* Readiness Probe
* Liveness Probe
* Startup Probe
* Rolling Update
* Rollback
* HPA
* Production Architecture

---

# 1. Kubernetes-এর পুরো ধারণা এক নজরে

Kubernetes-এ application চালানোর জন্য সবচেয়ে গুরুত্বপূর্ণ resource হলো:

```text
Deployment
DaemonSet
StatefulSet
```

আর Pod-গুলোর মধ্যে network communication করার জন্য:

```text
Service
```

সহজভাবে:

```text
                    Kubernetes Cluster
                           |
          +----------------+----------------+
          |                |                |
          v                v                v
     Deployment        DaemonSet       StatefulSet
          |                |                |
          v                v                v
        Pods             Pods             Pods
                                             |
                                            PVC
```

Networking:

```text
Client
   |
   v
Service
   |
   +--------+--------+
   |        |        |
   v        v        v
 Pod      Pod      Pod
```

---

# 2. Pod কী?

**Pod হলো Kubernetes-এর সবচেয়ে ছোট deployable unit।**

একটি Pod-এর মধ্যে এক বা একাধিক container থাকতে পারে।

সাধারণ application-এর ক্ষেত্রে:

```text
Pod
 |
 +--- .NET API Container
```

একাধিক tightly coupled container থাকলে:

```text
Pod
 |
 +--- Application Container
 |
 +--- Sidecar Container
```

একই Pod-এর container-গুলো:

* একই network namespace share করে
* একই Pod IP ব্যবহার করে
* volume share করতে পারে
* একই Pod lifecycle-এর অংশ

---

# 3. Pod-এর Basic Example

```yaml
apiVersion: v1
kind: Pod

metadata:
  name: bps-api-pod

spec:

  containers:

    - name: bps-api

      image: yourdockerhub/bps-api:1.0.0

      ports:
        - containerPort: 8080
```

Apply:

```bash
kubectl apply -f pod.yaml
```

Check:

```bash
kubectl get pods
```

তবে production-এ সাধারণত application Pod সরাসরি create করা হয় না।

এর পরিবর্তে ব্যবহার করা হয়:

```text
Deployment
DaemonSet
StatefulSet
```

---

# 4. Deployment কী?

Deployment হলো Kubernetes-এর সবচেয়ে বেশি ব্যবহৃত Workload Controller।

Deployment মূলত Stateless application-এর জন্য ব্যবহৃত হয়।

Deployment-এর সহজ প্রশ্ন:

> "আমার application-এর কয়টা copy সবসময় running থাকা উচিত?"

ধরো তোমার BPS API-এর 3টা instance দরকার:

```text
BPS API

Pod 1
Pod 2
Pod 3
```

তখন Deployment বলবে:

```text
আমার 3টা Pod সবসময় দরকার।
```

যদি কোনো Pod crash করে, Kubernetes নতুন Pod তৈরি করবে।

---

# 5. Deployment-এর Architecture

Deployment সরাসরি Pod-এর lifecycle manage করার জন্য ReplicaSet ব্যবহার করে।

```text
Deployment
     |
     v
ReplicaSet
     |
     +---------+---------+
     |         |         |
     v         v         v
   Pod 1     Pod 2     Pod 3
```

### Deployment-এর দায়িত্ব

* Application rollout
* Rolling update
* Rollback
* Replica management
* Version management

### ReplicaSet-এর দায়িত্ব

* Desired number of Pod maintain করা

---

# 6. Basic Deployment Example

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: bps-api

spec:

  replicas: 3

  selector:
    matchLabels:
      app: bps-api

  template:

    metadata:
      labels:
        app: bps-api

    spec:

      containers:

        - name: bps-api

          image: yourdockerhub/bps-api:1.0.0

          ports:
            - containerPort: 8080
```

Apply:

```bash
kubectl apply -f deployment.yaml
```

Deployment দেখো:

```bash
kubectl get deployments
```

Example:

```text
NAME       READY   UP-TO-DATE   AVAILABLE
bps-api    3/3     3            3
```

Pod দেখো:

```bash
kubectl get pods
```

Example:

```text
NAME                     READY   STATUS
bps-api-7d8f9c7c8b-x1    1/1     Running
bps-api-7d8f9c7c8b-x2    1/1     Running
bps-api-7d8f9c7c8b-x3    1/1     Running
```

---

# 7. ReplicaSet কী?

ReplicaSet-এর কাজ খুব সহজ:

> "Desired সংখ্যক Pod সবসময় running রাখো।"

যদি:

```yaml
replicas: 3
```

তাহলে:

```text
Desired = 3
Current = 3
```

ধরো একটি Pod crash করল:

```text
আগে:

Pod 1
Pod 2
Pod 3

Pod 2 Crash

Pod 1
Pod 3
```

ReplicaSet বুঝবে:

```text
Desired = 3
Current = 2
```

তখন নতুন Pod তৈরি করবে:

```text
Pod 1
Pod 3
Pod 4
```

অর্থাৎ:

```text
ReplicaSet = Desired Pod count maintain করে
```

সাধারণত Deployment ব্যবহার করার সময় ReplicaSet manually manage করার প্রয়োজন নেই।

Deployment নিজেই ReplicaSet manage করে।

---

# 8. Deployment Rolling Update

Deployment-এর সবচেয়ে গুরুত্বপূর্ণ feature-এর একটি হলো:

> **Rolling Update**

ধরো বর্তমানে BPS API চলছে:

```text
bps-api:1.0.0

Pod 1 → 1.0.0
Pod 2 → 1.0.0
Pod 3 → 1.0.0
```

এখন নতুন version:

```text
bps-api:2.0.0
```

deploy করলে Kubernetes ধীরে ধীরে পুরোনো Pod replace করতে পারে।

### Step 1

```text
1.0.0
1.0.0
1.0.0
```

### Step 2

```text
1.0.0
1.0.0
2.0.0
```

### Step 3

```text
1.0.0
2.0.0
2.0.0
```

### Step 4

```text
2.0.0
2.0.0
2.0.0
```

এটাই:

```text
Rolling Update
```

এর সুবিধা:

* পুরো application একসাথে বন্ধ করতে হয় না
* নতুন version ধীরে ধীরে deploy করা যায়
* সমস্যা হলে rollback করা যায়

---

# 9. Production-Grade Deployment

Production environment-এ শুধু:

```yaml
replicas: 3
```

দেওয়া যথেষ্ট নয়।

সাধারণত বিবেচনা করতে হয়:

```text
Resource requests
Resource limits
Readiness Probe
Liveness Probe
Startup Probe
Rolling Update
Revision history
Multiple replicas
```

Example:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: bps-api
  namespace: bps

spec:

  replicas: 3

  strategy:

    type: RollingUpdate

    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1

  revisionHistoryLimit: 5

  selector:
    matchLabels:
      app: bps-api

  template:

    metadata:
      labels:
        app: bps-api

    spec:

      containers:

        - name: bps-api

          image: yourdockerhub/bps-api:1.0.0

          ports:
            - containerPort: 8080

          resources:

            requests:
              cpu: "250m"
              memory: "256Mi"

            limits:
              cpu: "1000m"
              memory: "512Mi"

          readinessProbe:

            httpGet:
              path: /health
              port: 8080

            initialDelaySeconds: 10
            periodSeconds: 5

          livenessProbe:

            httpGet:
              path: /health
              port: 8080

            initialDelaySeconds: 30
            periodSeconds: 10

          startupProbe:

            httpGet:
              path: /health
              port: 8080

            failureThreshold: 30
            periodSeconds: 5
```

---

# 10. Resource Requests এবং Limits

Production Kubernetes-এ resource management খুব গুরুত্বপূর্ণ।

Example:

```yaml
resources:

  requests:
    cpu: "250m"
    memory: "256Mi"

  limits:
    cpu: "1000m"
    memory: "512Mi"
```

---

## 10.1 Requests

`requests` মানে:

> Kubernetes-এর কাছে container-এর জন্য অন্তত কত resource প্রয়োজন।

Example:

```text
CPU    = 250m
Memory = 256Mi
```

Kubernetes Pod schedule করার সময় এই resource requirement বিবেচনা করে।

---

## 10.2 Limits

`limits` মানে:

> Container সর্বোচ্চ কত resource ব্যবহার করতে পারবে।

Example:

```text
CPU    = 1000m
Memory = 512Mi
```

---

# 11. Health Probes

Kubernetes application health check করার জন্য তিনটি গুরুত্বপূর্ণ Probe আছে:

```text
Readiness Probe
Liveness Probe
Startup Probe
```

---

# 12. Readiness Probe

Readiness Probe-এর প্রশ্ন:

> "এই Pod কি এখন traffic নেওয়ার জন্য প্রস্তুত?"

ধরো:

```text
Pod Running
```

কিন্তু application এখনও startup করছে।

তাহলে:

```text
Pod = Running
Application = Not Ready
```

এই অবস্থায় Service ওই Pod-এ normal traffic পাঠানো বন্ধ রাখতে পারে।

Example:

```yaml
readinessProbe:

  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 10
  periodSeconds: 5
```

Architecture:

```text
Service
   |
   +---- Pod 1 → Ready
   |
   +---- Pod 2 → Not Ready
   |
   +---- Pod 3 → Ready
```

Traffic যাবে:

```text
Pod 1
Pod 3
```

Pod 2-তে যাবে না যতক্ষণ না সেটি Ready হয়।

---

# 13. Liveness Probe

Liveness Probe-এর প্রশ্ন:

> "Application এখনও জীবিত এবং কাজ করার মতো অবস্থায় আছে?"

ধরো application hang করেছে:

```text
Application
     |
     v
Liveness Failed
     |
     v
Container Restart
```

Example:

```yaml
livenessProbe:

  httpGet:
    path: /health
    port: 8080

  initialDelaySeconds: 30
  periodSeconds: 10
```

---

# 14. Startup Probe

Startup Probe slow-starting application-এর জন্য গুরুত্বপূর্ণ।

ধরো application start হতে 1-2 মিনিট সময় নেয়।

তখন startupProbe Kubernetes-কে জানায়:

> "Application-টাকে startup করার জন্য কিছু সময় দাও।"

Example:

```yaml
startupProbe:

  httpGet:
    path: /health
    port: 8080

  failureThreshold: 30
  periodSeconds: 5
```

Flow:

```text
Container Start
      |
      v
Startup Probe
      |
      v
Application Ready
      |
      v
Readiness/Liveness
```

---

# 15. Deployment কোথায় ব্যবহার করব?

Deployment সাধারণত Stateless application-এর জন্য ব্যবহার করা হয়।

Examples:

```text
ASP.NET Core Web API
.NET Microservice
Angular + Nginx
React
Node.js API
Java REST API
Python API
```

যদি application-এর প্রতিটি Pod একইভাবে কাজ করতে পারে এবং Pod-এর আলাদা identity দরকার না হয়, Deployment সাধারণত উপযুক্ত।

---

# 16. DaemonSet কী?

DaemonSet-এর মূল concept:

> **প্রত্যেক eligible Kubernetes Node-এ একটি নির্দিষ্ট Pod চালানো।**

ধরো তোমার cluster:

```text
Master
Worker-1
Worker-2
Worker-3
```

DaemonSet:

```text
Worker-1 → Agent Pod
Worker-2 → Agent Pod
Worker-3 → Agent Pod
```

এখন নতুন Node যোগ হলো:

```text
Worker-4
```

DaemonSet automatically সেখানে Pod চালাবে:

```text
Worker-4 → Agent Pod
```

---

# 17. DaemonSet কোথায় ব্যবহার হয়?

DaemonSet সাধারণত Node-level service-এর জন্য ব্যবহার করা হয়।

Examples:

```text
Log Collector
Monitoring Agent
Node Exporter
Security Agent
Storage Agent
Networking Agent
```

Popular examples:

```text
Fluent Bit
Fluentd
Filebeat
Node Exporter
```

---

# 18. DaemonSet Example

```yaml
apiVersion: apps/v1
kind: DaemonSet

metadata:
  name: node-log-agent
  namespace: monitoring

spec:

  selector:
    matchLabels:
      app: node-log-agent

  template:

    metadata:
      labels:
        app: node-log-agent

    spec:

      containers:

        - name: log-agent

          image: fluent/fluent-bit:latest

          resources:

            requests:
              cpu: "100m"
              memory: "100Mi"

            limits:
              cpu: "500m"
              memory: "300Mi"
```

Check:

```bash
kubectl get daemonsets -n monitoring
```

Example:

```text
DESIRED   CURRENT   READY
3         3         3
```

যদি 3টি eligible Node থাকে:

```text
3 Nodes
3 DaemonSet Pods
```

---

# 19. Deployment বনাম DaemonSet

## Deployment

তুমি বলছো:

> "আমার 5টা application replica দরকার।"

```yaml
replicas: 5
```

Kubernetes Node অনুযায়ী Pod distribute করতে পারে।

Example:

```text
Node 1 → Pod
Node 1 → Pod

Node 2 → Pod
Node 2 → Pod

Node 3 → Pod
```

---

## DaemonSet

তুমি বলছো:

> "প্রত্যেক eligible Node-এ 1টা করে Pod চাই।"

```text
Node 1 → Pod
Node 2 → Pod
Node 3 → Pod
```

এখানে সাধারণত manually:

```yaml
replicas: 3
```

দেওয়া হয় না।

কারণ Pod-এর সংখ্যা Node-এর সংখ্যা অনুযায়ী নির্ধারিত হয়।

---

# 20. StatefulSet কী?

StatefulSet হলো Stateful application-এর জন্য Kubernetes controller।

সহজভাবে:

> "আমার Pod-এর identity এবং storage গুরুত্বপূর্ণ।"

Example:

```text
db-0
db-1
db-2
```

এখানে Pod-এর নাম predictable এবং stable।

Deployment-এর ক্ষেত্রে সাধারণত Pod নাম random হয়:

```text
api-7d8f9c7c8b-x1
api-7d8f9c7c8b-x2
```

কিন্তু StatefulSet:

```text
db-0
db-1
db-2
```

---

# 21. StatefulSet-এর ৩টি গুরুত্বপূর্ণ feature

## 21.1 Stable Identity

```text
db-0
db-1
db-2
```

Pod restart হলেও identity একই থাকে।

---

## 21.2 Stable Network Identity

StatefulSet Pod-এর predictable network identity থাকতে পারে।

---

## 21.3 Stable Storage

প্রতিটি Pod-এর জন্য আলাদা PersistentVolumeClaim ব্যবহার করা যায়।

```text
db-0 → PVC-0
db-1 → PVC-1
db-2 → PVC-2
```

---

# 22. StatefulSet Architecture

```text
StatefulSet
     |
     +---- db-0
     |      |
     |      +---- PVC
     |
     +---- db-1
     |      |
     |      +---- PVC
     |
     +---- db-2
            |
            +---- PVC
```

---

# 23. StatefulSet Example

```yaml
apiVersion: apps/v1
kind: StatefulSet

metadata:
  name: mysql

spec:

  serviceName: mysql

  replicas: 3

  selector:
    matchLabels:
      app: mysql

  template:

    metadata:
      labels:
        app: mysql

    spec:

      containers:

        - name: mysql

          image: mysql:8.0

          env:

            - name: MYSQL_ROOT_PASSWORD
              value: "change-me"

          ports:

            - containerPort: 3306

          volumeMounts:

            - name: mysql-data
              mountPath: /var/lib/mysql

  volumeClaimTemplates:

    - metadata:
        name: mysql-data

      spec:

        accessModes:
          - ReadWriteOnce

        resources:

          requests:
            storage: 20Gi
```

এতে ধারণাগতভাবে:

```text
mysql-0
   |
   +---- mysql-data-mysql-0

mysql-1
   |
   +---- mysql-data-mysql-1

mysql-2
   |
   +---- mysql-data-mysql-2
```

---

# 24. গুরুত্বপূর্ণ StatefulSet Production Warning

একটি খুব গুরুত্বপূর্ণ বিষয়:

> **StatefulSet ব্যবহার করলেই Database automatically Highly Available হয়ে যায় না।**

যেমন:

```text
StatefulSet
replicas: 3
```

এর মানে এই নয় যে automatically:

```text
3-node database cluster
+
Replication
+
Automatic Failover
+
Backup
+
Disaster Recovery
```

হয়ে গেছে।

Database-এর জন্য আলাদাভাবে design করতে হয়:

```text
Replication
Failover
Quorum
Backup
Recovery
Storage
Consistency
Monitoring
```

Production environment-এ অনেক ক্ষেত্রে managed database ব্যবহার করা হয়, যেমন:

```text
Amazon RDS
Amazon Aurora
Azure Database
Google Cloud SQL
Managed PostgreSQL
```

কোনটা ব্যবহার করবে সেটা architecture requirement-এর উপর নির্ভর করে।

---

# 25. Deployment বনাম StatefulSet

## Deployment

ব্যবহার করো যখন:

```text
Pod identity গুরুত্বপূর্ণ নয়
Pod-specific storage প্রয়োজন নেই
Application stateless
```

Example:

```text
.NET API
Frontend
Microservice
```

Pods:

```text
api-x7sd8
api-92kd8
api-7sd82
```

---

## StatefulSet

ব্যবহার করো যখন:

```text
Pod identity গুরুত্বপূর্ণ
Stable network identity দরকার
Persistent storage প্রয়োজন
Replica-specific state দরকার
```

Example:

```text
db-0
db-1
db-2
```

---

# 26. Service কী?

Service Kubernetes-এর একটি গুরুত্বপূর্ণ Networking resource।

Service-এর মূল কাজ:

> **Pods-এর জন্য একটি stable network endpoint দেওয়া।**

ধরো:

```text
Pod 1 → 10.244.1.5
Pod 2 → 10.244.2.7
Pod 3 → 10.244.3.9
```

Pod 2 crash করল।

নতুন Pod হয়তো পেল:

```text
10.244.2.25
```

অর্থাৎ Pod IP পরিবর্তন হতে পারে।

তাই client-এর উচিত নয় সরাসরি Pod IP ব্যবহার করা।

এখানেই Service দরকার।

---

# 27. Service Architecture

```text
                 Service
              bps-api-service
                     |
          +----------+----------+
          |          |          |
          v          v          v
        Pod 1      Pod 2      Pod 3
```

Service-এর IP/endpoint stable থাকতে পারে, যদিও backend Pods পরিবর্তিত হয়।

---

# 28. Service কীভাবে Pod খুঁজে পায়?

Service **Label Selector** ব্যবহার করে।

Deployment-এর Pod:

```yaml
labels:
  app: bps-api
```

Service:

```yaml
selector:
  app: bps-api
```

মানে:

> `app=bps-api` label যেসব Pod-এর আছে, তাদের কাছে traffic পাঠাও।

Flow:

```text
Deployment Pod
      |
      | label: app=bps-api
      v
Service selector
      |
      v
Matching Pods
```

---

# 29. ClusterIP Service

ClusterIP হলো Service-এর default type।

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: bps-api-service

spec:

  type: ClusterIP

  selector:
    app: bps-api

  ports:

    - port: 80
      targetPort: 8080
```

এখানে:

```text
Service Port = 80
Pod Port     = 8080
```

Traffic:

```text
Client
   |
   v
bps-api-service:80
   |
   +---- Pod:8080
   +---- Pod:8080
   +---- Pod:8080
```

ClusterIP সাধারণত cluster-এর ভিতরের communication-এর জন্য ব্যবহৃত হয়।

---

# 30. NodePort

NodePort Kubernetes Node-এর একটি port-এর মাধ্যমে Service expose করে।

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: bps-api

spec:

  type: NodePort

  selector:
    app: bps-api

  ports:

    - port: 80
      targetPort: 8080
      nodePort: 30080
```

এখন:

```text
NodeIP:30080
```

থেকে Service access করা যায়।

Architecture:

```text
Client
   |
   v
NodeIP:30080
   |
   v
Service
   |
   +---- Pod
   +---- Pod
   +---- Pod
```

NodePort useful:

```text
Development
Testing
Kubernetes Lab
Simple external access
```

Production public traffic-এর জন্য direct NodePort সাধারণত preferred architecture নয়।

---

# 31. LoadBalancer

Cloud environment-এ Service:

```yaml
type: LoadBalancer
```

দিলে cloud provider external load balancer provision করতে পারে।

Example:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: bps-api

spec:

  type: LoadBalancer

  selector:
    app: bps-api

  ports:

    - port: 80
      targetPort: 8080
```

Architecture:

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Kubernetes Service
   |
   +---- Pod
   +---- Pod
   +---- Pod
```

AWS-এর মতো cloud environment-এ Kubernetes Service cloud load-balancing infrastructure-এর সাথে integrate করতে পারে।

---

# 32. Service Types

| Service Type | কাজ                             |
| ------------ | ------------------------------- |
| ClusterIP    | Cluster-এর ভিতরে communication  |
| NodePort     | Node-এর port দিয়ে expose করা    |
| LoadBalancer | External cloud load balancer    |
| ExternalName | External service-এর DNS mapping |

---

# 33. ClusterIP বনাম NodePort বনাম LoadBalancer

## ClusterIP

```text
Internal Client
      |
      v
Service
      |
      v
Pods
```

---

## NodePort

```text
Client
   |
   v
NodeIP:NodePort
   |
   v
Service
   |
   v
Pods
```

---

## LoadBalancer

```text
Internet
   |
   v
Cloud Load Balancer
   |
   v
Service
   |
   v
Pods
```

---

# 34. Deployment + Service একসাথে

Deployment:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: bps-api

spec:

  replicas: 3

  selector:
    matchLabels:
      app: bps-api

  template:

    metadata:
      labels:
        app: bps-api

    spec:

      containers:

        - name: bps-api

          image: yourdockerhub/bps-api:1.0.0

          ports:
            - containerPort: 8080
```

Service:

```yaml
apiVersion: v1
kind: Service

metadata:
  name: bps-api

spec:

  type: ClusterIP

  selector:
    app: bps-api

  ports:

    - name: http
      port: 80
      targetPort: 8080
```

এখানে সবচেয়ে গুরুত্বপূর্ণ connection:

```text
Deployment
     |
     v
Pods
     |
     | app=bps-api
     v
Service Selector
     |
     v
Service
```

---

# 35. Service Pod তৈরি করে না

এটা খুব গুরুত্বপূর্ণ।

Deployment:

```text
Deployment
     |
     +---- Pod তৈরি/Manage করে
```

Service:

```text
Service
   |
   +---- Pod খুঁজে
   |
   +---- Stable network endpoint দেয়
   |
   +---- Pod-এ traffic পাঠায়
```

তাই:

```text
Service != Deployment
Service != Pod
```

---

# 36. Service Endpoint

Service-এর backend হিসেবে কোন Pod আছে সেটা দেখতে:

```bash
kubectl get endpoints
```

Modern Kubernetes cluster-এ:

```bash
kubectl get endpointslices
```

Example:

```text
bps-api

10.244.1.10:8080
10.244.2.20:8080
10.244.3.30:8080
```

এগুলো backend Pod-এর network endpoints।

---

# 37. Common Service Problem — Selector Mismatch

Deployment:

```yaml
labels:
  app: bps-api
```

কিন্তু Service:

```yaml
selector:
  app: api
```

এখানে match করছে না।

Result:

```text
Service
   |
   X
No matching Pods
```

Check:

```bash
kubectl get endpoints bps-api
```

যদি endpoint না থাকে, প্রথমে check করো:

```text
Service selector
        vs
Pod labels
```

---

# 38. Kubernetes DNS

Kubernetes Service-এর জন্য automatically DNS record তৈরি করে।

ধরো:

```text
Service = bps-api
Namespace = bps
```

একই namespace থেকে:

```text
http://bps-api
```

অন্য namespace থেকে:

```text
http://bps-api.bps.svc.cluster.local
```

DNS structure:

```text
service.namespace.svc.cluster.local
```

Example:

```text
bps-api.bps.svc.cluster.local
```

এটা Kubernetes-এর microservice communication-এর জন্য খুব গুরুত্বপূর্ণ।

---

# 39. Frontend থেকে Backend Communication

ধরো:

```text
Namespace = bps
Service = bps-api
Port = 80
```

Frontend Pod থেকে:

```text
http://bps-api
```

call করা যায়।

Architecture:

```text
Frontend Pod
     |
     | HTTP
     v
bps-api Service
     |
     +--------+--------+
     |        |        |
     v        v        v
   API 1    API 2    API 3
```

Frontend সরাসরি:

```text
10.244.x.x
```

Pod IP ব্যবহার করবে না।

Service ব্যবহার করবে।

---

# 40. Deployment + Service + HPA

HPA Deployment-এর replica সংখ্যা dynamically পরিবর্তন করতে পারে।

Architecture:

```text
                    Service
                       |
          +------------+------------+
          |            |            |
          v            v            v
        Pod 1        Pod 2        Pod 3
          ^            ^            ^
          |            |            |
          +------------+------------+
                       |
                      HPA
                       |
                  CPU / Memory
```

ধরো শুরুতে:

```text
3 Pods
```

CPU বেশি হয়ে গেলে:

```text
3
 ↓
5
 ↓
8
```

CPU কমে গেলে:

```text
8
 ↓
5
 ↓
3
```

HPA desired replica সংখ্যা পরিবর্তন করে, আর Deployment সেই অনুযায়ী Pod manage করে।

---

# 41. Production Kubernetes Architecture

একটি production application-এর architecture এমন হতে পারে:

```text
                         Internet
                            |
                            v
                  Cloud Load Balancer
                            |
                            v
                         Ingress
                            |
                            v
                     API Service
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          API Pod         API Pod         API Pod
             ^              ^              ^
             |              |              |
             +--------------+--------------+
                            |
                       Deployment
                            |
                         ReplicaSet
```

---

# 42. HPA Architecture

```text
                    HPA
                     |
                     v
                Deployment
                     |
              +------+------+------+
              |      |      |      |
              v      v      v      v
             Pod    Pod    Pod    Pod
```

HPA সাধারণত CPU/Memory বা অন্যান্য metrics-এর উপর ভিত্তি করে replica সংখ্যা পরিবর্তন করে।

---

# 43. DaemonSet Architecture

ধরো 3টি Worker Node:

```text
Worker 1
   |
   +--- Log Agent

Worker 2
   |
   +--- Log Agent

Worker 3
   |
   +--- Log Agent
```

এটাই DaemonSet-এর মূল ধারণা।

---

# 44. StatefulSet Architecture

```text
StatefulSet
    |
    +--- db-0
    |      |
    |      +--- PVC
    |
    +--- db-1
    |      |
    |      +--- PVC
    |
    +--- db-2
           |
           +--- PVC
```

---

# 45. সবগুলো একসাথে

একটি production Kubernetes environment-এ conceptually:

```text
                         Internet
                            |
                            v
                   Cloud Load Balancer
                            |
                            v
                         Ingress
                            |
                            v
                       API Service
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          API Pod         API Pod         API Pod
             ^              ^              ^
             |              |              |
             +--------------+--------------+
                            |
                       Deployment
                            |
                        ReplicaSet
```

Node-level:

```text
Node 1 → DaemonSet Pod
Node 2 → DaemonSet Pod
Node 3 → DaemonSet Pod
```

Stateful workload:

```text
StatefulSet

db-0 → PVC
db-1 → PVC
db-2 → PVC
```

Autoscaling:

```text
HPA
 |
 v
Deployment
 |
 v
Pods
```

---

# 46. BPS Application-এর Example Architecture

BPS-এর মতো application-এর জন্য একটি সম্ভাব্য architecture:

```text
                         Internet
                            |
                            v
                   AWS Load Balancer
                            |
                            v
                         Ingress
                            |
                            v
                    bps-api Service
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          API Pod         API Pod         API Pod
             |              |              |
             +--------------+--------------+
                            |
                            v
                       SQL Server
```

Kubernetes resources হতে পারে:

```text
Deployment
Service
Ingress
ConfigMap
Secret
HPA
PodDisruptionBudget
ServiceAccount
NetworkPolicy
```

Node-level resources:

```text
DaemonSet
```

Stateful workload প্রয়োজন হলে:

```text
StatefulSet + PVC
```

---

# 47. Production Kubernetes Resource Map

## Application Lifecycle

```text
Deployment
    |
    +--- ReplicaSet
             |
             +--- Pods
```

Stateful application:

```text
StatefulSet
    |
    +--- Stateful Pods
             |
             +--- PVC
```

Node-level application:

```text
DaemonSet
    |
    +--- One Pod per eligible Node
```

Networking:

```text
Ingress / Gateway
       |
       v
Service
       |
       v
Pods
```

Autoscaling:

```text
HPA
 |
 v
Deployment
 |
 v
Pods
```

---

# 48. Deployment বনাম DaemonSet বনাম StatefulSet

| বিষয়                      | Deployment            | DaemonSet          | StatefulSet              |
| ------------------------- | --------------------- | ------------------ | ------------------------ |
| মূল কাজ                   | Stateless application | Node-level Pod     | Stateful application     |
| Replica count             | Yes                   | Node অনুযায়ী       | Yes                      |
| Stable Pod identity       | না                    | মূল উদ্দেশ্য নয়    | Yes                      |
| Stable storage            | সাধারণত না            | প্রয়োজন অনুযায়ী    | Yes                      |
| Rolling update            | Yes                   | Yes                | Yes                      |
| সাধারণ ব্যবহার            | API/Frontend          | Logging/Monitoring | Database/Stateful system |
| Pod identity গুরুত্বপূর্ণ | না                    | সাধারণত না         | Yes                      |
| Replica-specific PVC      | Optional              | Optional           | Common                   |

---

# 49. Service বনাম Workload Controller

এটা খুব ভালোভাবে মনে রাখতে হবে।

## Workload Controllers

```text
Deployment
DaemonSet
StatefulSet
ReplicaSet
      |
      v
    Pods
```

এদের কাজ:

> Application workload manage করা।

---

## Networking

```text
Service
   |
   v
Pods
```

Service-এর কাজ:

> Pod-গুলোর জন্য stable network endpoint এবং traffic routing দেওয়া।

---

# 50. খুব সহজে মনে রাখার নিয়ম

## Deployment

> "আমার application-এর কয়েকটি copy দরকার।"

```text
API → 3 replicas
```

---

## DaemonSet

> "প্রত্যেক eligible Node-এ একটি Pod দরকার।"

```text
Node 1 → Agent
Node 2 → Agent
Node 3 → Agent
```

---

## StatefulSet

> "আমার Pod-এর identity এবং storage গুরুত্বপূর্ণ।"

```text
db-0
db-1
db-2
```

---

## Service

> "এই Pods-গুলোর কাছে stable network address দিয়ে traffic পাঠাতে হবে।"

```text
Client
  |
  v
Service
  |
  v
Pods
```

---

# 51. গুরুত্বপূর্ণ পার্থক্য

এই বিষয়গুলো কখনো এক করে ফেলবে না:

```text
Deployment ≠ Pod

Deployment ≠ Service

DaemonSet ≠ Service

StatefulSet ≠ Database

Service ≠ Pod
```

সঠিক ধারণা:

```text
Deployment
    ↓
ReplicaSet
    ↓
Pods
```

```text
Service
    ↓
Pods
```

```text
DaemonSet
    ↓
Pods
```

```text
StatefulSet
    ↓
Stateful Pods
    ↓
PVC
```

---

# 52. Production Checklist

Production Kubernetes workload deploy করার আগে সাধারণত নিচের বিষয়গুলো বিবেচনা করতে হবে:

```text
[ ] Deployment / StatefulSet / DaemonSet
[ ] Multiple replicas
[ ] Resource requests
[ ] Resource limits
[ ] Readiness Probe
[ ] Liveness Probe
[ ] Startup Probe
[ ] Rolling Update strategy
[ ] Rollback strategy
[ ] HPA
[ ] PodDisruptionBudget
[ ] ConfigMap
[ ] Secret
[ ] Service
[ ] Ingress / Gateway
[ ] NetworkPolicy
[ ] RBAC
[ ] ServiceAccount
[ ] PersistentVolume
[ ] StorageClass
[ ] Centralized Logging
[ ] Monitoring
[ ] Metrics
[ ] Backup
[ ] Disaster Recovery
[ ] Security Policies
```

---

# 53. গুরুত্বপূর্ণ Kubernetes Commands

## Deployment

```bash
kubectl get deployments
```

```bash
kubectl describe deployment bps-api
```

```bash
kubectl rollout status deployment/bps-api
```

```bash
kubectl rollout history deployment/bps-api
```

---

# 54. Deployment-এর Image Update

```bash
kubectl set image deployment/bps-api \
  bps-api=yourdockerhub/bps-api:2.0.0
```

তারপর:

```bash
kubectl rollout status deployment/bps-api
```

---

# 55. Rollback

যদি নতুন version-এ সমস্যা হয়:

```bash
kubectl rollout undo deployment/bps-api
```

নির্দিষ্ট revision-এ rollback:

```bash
kubectl rollout undo deployment/bps-api --to-revision=2
```

---

# 56. Pod Commands

সব Pod:

```bash
kubectl get pods
```

Node সহ Pod:

```bash
kubectl get pods -o wide
```

Pod-এর বিস্তারিত:

```bash
kubectl describe pod <pod-name>
```

Pod logs:

```bash
kubectl logs <pod-name>
```

---

# 57. DaemonSet Commands

```bash
kubectl get daemonsets
```

```bash
kubectl describe daemonset <daemonset-name>
```

---

# 58. StatefulSet Commands

```bash
kubectl get statefulsets
```

```bash
kubectl describe statefulset <statefulset-name>
```

---

# 59. Service Commands

সব Service:

```bash
kubectl get services
```

Service-এর বিস্তারিত:

```bash
kubectl describe service bps-api
```

Endpoints:

```bash
kubectl get endpoints
```

EndpointSlices:

```bash
kubectl get endpointslices
```

---

# 60. Final Mental Model

Kubernetes বোঝার জন্য নিচের architecture-টা মনে রাখো:

```text
                         USER
                           |
                           v
                    Ingress / Gateway
                           |
                           v
                        Service
                           |
              +------------+------------+
              |            |            |
              v            v            v
            Pod 1        Pod 2        Pod 3
              ^            ^            ^
              |            |            |
              +------------+------------+
                           |
                      Deployment
                           |
                       ReplicaSet
```

Node-level workload:

```text
Node 1 → DaemonSet Pod
Node 2 → DaemonSet Pod
Node 3 → DaemonSet Pod
```

Stateful workload:

```text
StatefulSet

db-0 → PVC
db-1 → PVC
db-2 → PVC
```

Autoscaling:

```text
HPA
 |
 v
Deployment
 |
 v
Pods
```

---

# 61. এক লাইনে সবকিছু

| Kubernetes Resource | সহজভাবে মনে রাখো                                        |
| ------------------- | ------------------------------------------------------- |
| Pod                 | Container চালানোর জায়গা                                 |
| ReplicaSet          | নির্দিষ্ট সংখ্যক Pod চালু রাখে                          |
| Deployment          | Stateless application-এর deployment ও update manage করে |
| DaemonSet           | প্রত্যেক eligible Node-এ Pod চালায়                      |
| StatefulSet         | Stable identity ও storage সহ stateful workload চালায়    |
| Service             | Pods-এর জন্য stable network endpoint দেয়                |
| ClusterIP           | Cluster-এর ভিতরের Service                               |
| NodePort            | Node-এর port দিয়ে Service expose করে                    |
| LoadBalancer        | Cloud-এর external Load Balancer ব্যবহার করে             |
| HPA                 | প্রয়োজন অনুযায়ী replica সংখ্যা বাড়ায়/কমানো              |

---

# 62. Kubernetes শেখার Recommended Order

Practical Kubernetes শেখার জন্য এই order follow করা ভালো:

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
20. Production Deployment
```

---

# 63. Final Summary

Kubernetes-এর এই চারটি resource-এর মূল ধারণা:

```text
Deployment
    ↓
"আমার application-এর কয়েকটি replica চালাও।"
```

```text
DaemonSet
    ↓
"প্রত্যেক eligible Node-এ একটি Pod চালাও।"
```

```text
StatefulSet
    ↓
"আমার Pod-এর identity এবং storage গুরুত্বপূর্ণ।"
```

```text
Service
    ↓
"এই Pods-গুলোর জন্য stable network endpoint দাও।"
```

সবকিছু একসাথে:

```text
                         Internet
                            |
                            v
                    Load Balancer
                            |
                            v
                         Ingress
                            |
                            v
                        Service
                            |
             +--------------+--------------+
             |              |              |
             v              v              v
          API Pod         API Pod         API Pod
             ^              ^              ^
             |              |              |
             +--------------+--------------+
                            |
                       Deployment
                            |
                        ReplicaSet


Node 1 ───── DaemonSet Pod
Node 2 ───── DaemonSet Pod
Node 3 ───── DaemonSet Pod


StatefulSet:

db-0 ───── PVC
db-1 ───── PVC
db-2 ───── PVC


HPA
 ↓
Deployment
 ↓
Pods
```

**এই mental model-টা পরিষ্কার থাকলে Kubernetes-এর পরের topics—Ingress, HPA, ConfigMap, Secret, PVC, StorageClass, PDB, NetworkPolicy এবং production deployment—অনেক সহজে বোঝা যাবে।**
