# Kubernetes Storage: PV, PVC & StorageClass — Complete Guide

> Kubernetes-এ persistent storage কীভাবে কাজ করে: PersistentVolume (PV), PersistentVolumeClaim (PVC), StorageClass, dynamic provisioning, StatefulSet এবং production best practices।

---

## Table of Contents

1. [Why Persistent Storage?](#chapter-1--why-persistent-storage)
2. [Core Concepts: PV, PVC, StorageClass](#chapter-2--core-concepts-pv-pvc-storageclass)
3. [PersistentVolume (PV)](#chapter-3--persistentvolume-pv)
4. [PersistentVolumeClaim (PVC)](#chapter-4--persistentvolumeclaim-pvc)
5. [StorageClass](#chapter-5--storageclass)
6. [Static vs Dynamic Provisioning](#chapter-6--static-vs-dynamic-provisioning)
7. [Access Modes](#chapter-7--access-modes)
8. [Reclaim Policy](#chapter-8--reclaim-policy)
9. [volumeBindingMode](#chapter-9--volumebindingmode)
10. [Hands-on: SQL Server with PVC (AWS EBS)](#chapter-10--hands-on-sql-server-with-pvc-aws-ebs)
11. [StatefulSet & volumeClaimTemplates](#chapter-11--statefulset--volumeclaimtemplates)
12. [PVC ↔ PV Matching Rules](#chapter-12--pvc--pv-matching-rules)
13. [Essential kubectl Commands & Troubleshooting](#chapter-13--essential-kubectl-commands--troubleshooting)
14. [Production Architecture & Best Practices](#chapter-14--production-architecture--best-practices)
15. [Interview Q&A](#chapter-15--interview-qa)
16. [Recommended Repository Structure](#chapter-16--recommended-repository-structure)

---

## Chapter 1 — Why Persistent Storage?

Kubernetes-এ একটি সাধারণ Pod-এর container filesystem **ephemeral**। Pod delete বা recreate হলে container-এর ভেতরের data হারিয়ে যায়।

উদাহরণ হিসেবে একটি application ধরা যাক:

```text
Angular  →  .NET API  →  SQL Server
```

SQL Server Pod-এর ভেতরে database file থাকে:

```text
SQL Server Pod
 └── /var/opt/mssql
       ├── BPS.mdf
       └── BPS_log.ldf
```

কোনো persistent storage না থাকলে:

```text
Pod deleted → Database files gone → DATA LOSS ❌
```

এই সমস্যার সমাধান হলো **PV, PVC এবং StorageClass**।

---

## Chapter 2 — Core Concepts: PV, PVC, StorageClass

```text
StorageClass   →  "কী ধরনের storage, কীভাবে তৈরি হবে?"
      ↓
PVC            →  "আমার 100Gi storage দরকার"
      ↓
PV             →  "এই হলো actual 100Gi storage"
      ↓
Storage Backend → EBS / Azure Disk / NFS / Ceph / ...
```

| Component | কাজ |
|---|---|
| **PV** | Cluster-level actual persistent storage resource |
| **PVC** | Application-এর storage request |
| **StorageClass** | Storage কীভাবে dynamically provision হবে তার policy/configuration |

> **মূল নিয়ম:** Pod সাধারণত PV-এর সাথে সরাসরি কথা বলে না।
> `Pod → PVC → PV → Storage Backend`

### সহজ উদাহরণ (বাসা ভাড়ার analogy)

- **PV** = বাড়িওয়ালার হাতে থাকা 1000 sq ft-এর বাসা (available resource)
- **PVC** = তোমার চাওয়া "আমার 700 sq ft বাসা দরকার" (request)
- Kubernetes দুটোকে match (bind) করে দেয়।

---

## Chapter 3 — PersistentVolume (PV)

PV হলো cluster-এর একটি storage resource। এটি Kubernetes-কে বলে: *"আমার কাছে এই storage available আছে।"*

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: bps-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: manual
  hostPath:
    path: /mnt/data/bps
```

| Field | অর্থ |
|---|---|
| `capacity.storage` | PV-এর size (এখানে 10 GB) |
| `accessModes` | কীভাবে mount করা যাবে |
| `persistentVolumeReclaimPolicy` | PVC delete হলে PV-এর কী হবে |
| `storageClassName` | কোন class-এর অন্তর্গত |
| `hostPath` | Node-এর local path |

> ⚠️ **Note:** `hostPath` শুধু **single-node learning/testing** (minikube, kind) এর জন্য। Production-এ ব্যবহার করা উচিত নয়, কারণ Pod অন্য node-এ গেলে data পাওয়া যাবে না এবং security risk আছে।

---

## Chapter 4 — PersistentVolumeClaim (PVC)

PVC হলো application-এর storage request।

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: bps-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: manual
```

এখানে application বলছে:

```text
Storage      = 5Gi
Access Mode  = ReadWriteOnce
StorageClass = manual
```

Kubernetes এই শর্ত পূরণ করে এমন একটি PV খুঁজে PVC-এর সাথে **bind** করে।

---

## Chapter 5 — StorageClass

StorageClass হলো Kubernetes-এর **storage provisioning policy**।

> কেউ storage চাইলে কোন ধরনের storage এবং কীভাবে সেটা তৈরি হবে — StorageClass সেটাই ঠিক করে।

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: bps-gp3
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  fsType: ext4
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

| Field | অর্থ |
|---|---|
| `provisioner` | কোন CSI driver storage তৈরি করবে (এখানে AWS EBS CSI) |
| `parameters` | Backend-specific setting (disk type, filesystem) |
| `reclaimPolicy` | Dynamic PV-এর reclaim policy |
| `volumeBindingMode` | কখন volume provision/bind হবে |
| `allowVolumeExpansion` | পরে PVC size বাড়ানো যাবে কিনা |

### CSI Driver কী?

**CSI (Container Storage Interface) Driver** হলো Kubernetes এবং storage provider-এর মধ্যে bridge। AWS EBS, EFS, Azure Disk, Ceph — সবার জন্য আলাদা CSI driver আছে।

---

## Chapter 6 — Static vs Dynamic Provisioning

### 6.1 Static Provisioning

Admin নিজে PV তৈরি করে, user PVC দিয়ে সেটা claim করে।

```text
Admin → Create PV → User creates PVC → PVC binds to PV → Pod
```

শেখার জন্য ভালো, কিন্তু বড় scale-এ manual PV তৈরি করা কঠিন।

### 6.2 Dynamic Provisioning

Production-এ সাধারণত এটাই ব্যবহার হয়।

```text
Developer → PVC → StorageClass → CSI Driver → Cloud Storage
                                                    ↓
                                      PV automatically created
                                                    ↓
                                              PVC = Bound → Pod
```

### 6.3 কেন Production-এ Dynamic ভালো?

৫০টা application থাকলে ৫০টা PV manually বানাতে হতো। Dynamic provisioning-এ:

- Automation
- Scalability
- Infrastructure consistency
- কম manual কাজ
- Kubernetes-native workflow

### 6.4 Comparison

| বিষয় | Static PV | Dynamic Provisioning |
|---|---|---|
| PV manually create | Yes | No |
| PVC | Yes | Yes |
| StorageClass | Optional | Usually Yes |
| Automation | Low | High |
| Scalability | Low | High |
| Production | Specific cases | Preferred |
| Cloud Kubernetes | Less common | Common |

---

## Chapter 7 — Access Modes

| Mode | Short | অর্থ | Common backend |
|---|---|---|---|
| ReadWriteOnce | RWO | একটি **node** থেকে read/write | AWS EBS |
| ReadOnlyMany | ROX | অনেক node থেকে read-only | NFS, EFS |
| ReadWriteMany | RWX | অনেক node থেকে read/write | NFS, AWS EFS, Azure Files, CephFS |
| ReadWriteOncePod | RWOP | শুধু **একটি Pod** থেকে read/write | CSI drivers (Kubernetes 1.29+ stable) |

### RWO vs RWX — কখন কোনটা?

- **একটি primary database Pod** (যেমন SQL Server) → সাধারণত **RWO** যথেষ্ট।
- **একাধিক application Pod একই shared file ব্যবহার করবে** → **RWX** দরকার হতে পারে।

```text
Pod 1 ─┐
Pod 2 ─┼──► Shared Files (RWX, e.g. EFS)
Pod 3 ─┘
```

---

## Chapter 8 — Reclaim Policy

PVC delete হলে PV এবং actual data-র কী হবে, সেটা ঠিক করে `persistentVolumeReclaimPolicy`।

### Retain

```text
PVC deleted → PV "Released" → Actual data remains
```

Database-এর জন্য নিরাপদ। তবে Released PV নতুন PVC-তে স্বয়ংক্রিয়ভাবে bind হয় না; manual cleanup (যেমন `claimRef` মুছে ফেলা) করতে হয়।

### Delete

```text
PVC deleted → PV deleted → EBS volume deleted
```

> ⚠️ Dynamically provisioned volume-এর **default reclaim policy হলো `Delete`**। Production database-এ `Delete` ব্যবহারের আগে খুব সতর্ক থাকতে হবে। StorageClass-এ স্পষ্টভাবে `reclaimPolicy: Retain` দেওয়া ভালো।

---

## Chapter 9 — volumeBindingMode

| Mode | আচরণ |
|---|---|
| `Immediate` | PVC তৈরি হলেই volume provision হয় |
| `WaitForFirstConsumer` | Pod কোন node/zone-এ schedule হবে সেটা ঠিক হওয়ার পর volume provision হয় |

```text
Immediate:              PVC → EBS created (zone অনুমান করে)
WaitForFirstConsumer:   PVC → Pod scheduling → Node/Zone selected → EBS provisioned
```

**Multi-AZ cloud environment-এ `WaitForFirstConsumer` প্রায় বাধ্যতামূলক**, কারণ EBS-এর মতো block storage zone-specific। ভুল zone-এ volume তৈরি হলে Pod schedule হতে পারে না।

---

## Chapter 10 — Hands-on: SQL Server with PVC (AWS EBS)

ধরা যাক BPS application AWS EKS-এ চলছে এবং SQL Server-এর persistent storage দরকার।

### Step 1 — PVC তৈরি

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: sqlserver-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: bps-gp3
  resources:
    requests:
      storage: 100Gi
```

### Step 2 — Kubernetes কী করে?

```text
PVC (storageClassName = bps-gp3)
   ↓
StorageClass
   ↓
EBS CSI Driver
   ↓
AWS EBS 100Gi
   ↓
PV automatically created
   ↓
PVC = Bound
```

তুমি manually কোনো PV বানাওনি।

### Step 3 — Pod-এ PVC ব্যবহার

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: sqlserver
spec:
  serviceName: sqlserver
  replicas: 1
  selector:
    matchLabels:
      app: sqlserver
  template:
    metadata:
      labels:
        app: sqlserver
    spec:
      securityContext:
        fsGroup: 10001          # SQL Server container-এর mssql user
      containers:
        - name: sqlserver
          image: mcr.microsoft.com/mssql/server:2022-latest
          env:
            - name: ACCEPT_EULA
              value: "Y"
            - name: MSSQL_SA_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: sqlserver-secret
                  key: password
          volumeMounts:
            - name: sql-data
              mountPath: /var/opt/mssql
      volumes:
        - name: sql-data
          persistentVolumeClaim:
            claimName: sqlserver-pvc
```

গুরুত্বপূর্ণ দুটি অংশ:

```yaml
volumeMounts:            # container-এর কোথায় mount হবে
  - name: sql-data
    mountPath: /var/opt/mssql

volumes:                 # কোন PVC ব্যবহার হবে
  - name: sql-data
    persistentVolumeClaim:
      claimName: sqlserver-pvc
```

> **Note:** SQL Server container non-root `mssql` user হিসেবে চলে, তাই volume-এ write permission-এর জন্য `fsGroup: 10001` দেওয়া হয়েছে। এটি না দিলে অনেক সময় permission error আসে।

### Complete Flow Diagram

```text
                 Kubernetes Cluster
                        │
                 ┌──────▼──────┐
                 │     Pod     │
                 │  SQL Server │
                 └──────┬──────┘
                   volumeMount
                 ┌──────▼──────┐
                 │     PVC     │
                 │   100 Gi    │
                 └──────┬──────┘
                      Bound
                 ┌──────▼──────┐
                 │      PV     │
                 │   100 Gi    │
                 └──────┬──────┘
                 provisioned by
                 ┌──────▼──────┐
                 │StorageClass │
                 │   bps-gp3   │
                 └──────┬──────┘
                    CSI Driver
                 ┌──────▼──────┐
                 │   AWS EBS   │
                 │   100 Gi    │
                 └─────────────┘
```

---

## Chapter 11 — StatefulSet & volumeClaimTemplates

Stateful application-এর জন্য Kubernetes-এ **StatefulSet** ব্যবহার করা হয়। `volumeClaimTemplates` দিয়ে StatefulSet **প্রতিটি Pod-এর জন্য আলাদা PVC** তৈরি করে।

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
        - name: postgres
          image: postgres:16
          volumeMounts:
            - name: postgres-data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: postgres-data
      spec:
        accessModes:
          - ReadWriteOnce
        storageClassName: bps-gp3
        resources:
          requests:
            storage: 50Gi
```

`replicas: 3` হলে:

```text
postgres-0 → postgres-data-postgres-0 → PV 0
postgres-1 → postgres-data-postgres-1 → PV 1
postgres-2 → postgres-data-postgres-2 → PV 2
```

> **Note:** শুধু `replicas: 3` দিলেই database replication হয় না। PostgreSQL-এর replication/HA আলাদাভাবে configure করতে হয় (বা Operator ব্যবহার করতে হয়)। এখানে উদ্দেশ্য শুধু per-Pod storage pattern বোঝানো।

---

## Chapter 12 — PVC ↔ PV Matching Rules

Kubernetes এই property গুলো দেখে PVC-কে PV-এর সাথে match করে:

- `storageClassName`
- `accessModes`
- `capacity` (PV ≥ PVC request)

### Match হবে ✅

```text
PVC: 10Gi, RWO, gp3
PV : 20Gi, RWO, gp3
→ Bound
```

### Match হবে না ❌

```text
PVC: 50Gi
PV : 20Gi
→ PVC Pending
```

---

## Chapter 13 — Essential kubectl Commands & Troubleshooting

### Commands

```bash
kubectl get pvc
kubectl get pv
kubectl get storageclass        # অথবা: kubectl get sc

kubectl describe pvc sqlserver-pvc
kubectl describe pv <pv-name>
```

Example output:

```text
NAME             STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS
sqlserver-pvc    Bound    pvc-xxx    100Gi      RWO            bps-gp3
```

### PVC `Pending` হলে

```bash
kubectl get pvc
kubectl describe pvc sqlserver-pvc     # শেষের Events অংশ দেখো
```

| সম্ভাব্য কারণ | কী চেক করবে |
|---|---|
| StorageClass ভুল/নেই | `kubectl get sc`, PVC-র `storageClassName` |
| CSI driver install নেই | `kubectl get pods -n kube-system` |
| Insufficient capacity | PV size বনাম PVC request |
| Unsupported access mode | Backend কোন mode support করে |
| Zone mismatch | `volumeBindingMode`, node zone |
| Provisioner issue | CSI controller Pod-এর log |
| `WaitForFirstConsumer` | Pod এখনো না চললে PVC Pending থাকা **স্বাভাবিক** |

---

## Chapter 14 — Production Architecture & Best Practices

### 14.1 Standard Production Pattern

```text
Developer → PVC → StorageClass → CSI Driver
                                     │
                    ┌────────────────┴───────────────┐
                    ▼                                ▼
                AWS EBS (RWO)                   AWS EFS (RWX)
                    │                                │
            Stateful workloads                  Shared files
            (SQL Server, DB)             (multiple API Pods)
```

### 14.2 Use Cases

| Use case | Storage |
|---|---|
| Database (SQL Server, PostgreSQL, MySQL, MongoDB) | Block storage (EBS) + RWO |
| File upload (PDF, image, document) | Object storage (S3/Blob/GCS) অগ্রাধিকার; shared filesystem লাগলে EFS/NFS/CephFS |
| Stateful apps (Kafka, Elasticsearch, Prometheus) | PVC + StatefulSet |

### 14.3 PV থাকলেই Database Production-Ready নয়

```text
Persistent Storage
  + Backup
  + Recovery
  + Replication / HA
  + Monitoring
  + Security
  + Disaster Recovery
```

### 14.4 In-cluster Database vs Managed Database

Production AWS environment-এ অনেক organization Kubernetes-এর ভেতরে database না চালিয়ে managed service ব্যবহার করে:

```text
EKS
 ├── Angular
 ├── .NET API
 ├── Redis
 └── RabbitMQ
        │
        ▼
       RDS  (PostgreSQL / MySQL / SQL Server)
```

কারণ managed service-এ backup, patching, HA, replication, monitoring, failover — এই operational burden কমে যায়।

### 14.5 Best Practices Checklist

- [ ] Production-এ dynamic provisioning (PVC + StorageClass + CSI) ব্যবহার করো
- [ ] Database-এর জন্য `reclaimPolicy: Retain` রাখো
- [ ] Multi-AZ cluster-এ `volumeBindingMode: WaitForFirstConsumer` ব্যবহার করো
- [ ] `allowVolumeExpansion: true` রাখো যাতে পরে size বাড়ানো যায়
- [ ] Stateful workload-এর জন্য `StatefulSet` + `volumeClaimTemplates` ব্যবহার করো
- [ ] `hostPath` production-এ ব্যবহার করো না
- [ ] Regular backup / snapshot (যেমন `VolumeSnapshot`) configure করো
- [ ] Secret (যেমন `MSSQL_SA_PASSWORD`) Kubernetes Secret বা external secret manager-এ রাখো
- [ ] Storage usage monitor করো

---

## Chapter 15 — Interview Q&A

**Q: What are PV, PVC and StorageClass in Kubernetes?**

> A **PersistentVolume (PV)** is a cluster-level storage resource that represents actual persistent storage.
> A **PersistentVolumeClaim (PVC)** is a request for storage made by an application.
> A **StorageClass** defines how storage should be dynamically provisioned, typically through a CSI driver such as AWS EBS or EFS CSI.
> In production environments, dynamic provisioning through PVC + StorageClass + CSI is generally preferred over manually creating PVs, because it provides automation, scalability, and better infrastructure management.

### Mental Model (মনে রাখার সূত্র)

```text
Pod
 │ uses
 ▼
PVC            = আমার কী storage দরকার?
 │ binds to
 ▼
PV             = actual storage resource
 │ provisioned by
 ▼
StorageClass   = storage কীভাবে তৈরি হবে?
 │ via
 ▼
CSI Driver     = Kubernetes ↔ storage provider-এর bridge
 ▼
Actual Storage
```

---

## Chapter 16 — Recommended Repository Structure

```text
kubernetes-storage/
│
├── README.md                      # এই guide
│
├── 01-static-pv/
│   ├── pv.yaml
│   └── pvc.yaml
│
├── 02-dynamic-provisioning/
│   ├── storageclass.yaml
│   └── pvc.yaml
│
├── 03-statefulset/
│   └── postgres-statefulset.yaml
│
└── 04-production/
    ├── storageclass-gp3.yaml
    ├── database-pvc.yaml
    └── statefulset.yaml
```

এভাবে সাজালে repository দেখেই বোঝা যাবে যে শুধু YAML মুখস্থ নয়, Kubernetes storage architecture-ও বোঝা হয়েছে।

---

## License

Educational / learning purposes.
