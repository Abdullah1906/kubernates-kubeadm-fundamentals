# Kubernetes-এ Database: Scale ও External Connection গাইড

Database কীভাবে scale হয়, External Database-এর সাথে App Pod কীভাবে যুক্ত হয়, এবং Firewall ও Network কীভাবে সাজাতে হয় — উদাহরণসহ ধাপে ধাপে বাংলা গাইড।

## সূচিপত্র

- [প্রশ্ন ১: App-এর মতো Database কি scale করা যায়?](#প্রশ্ন-১-app-এর-মতো-database-কি-replicas-দিয়ে-scale-করা-যায়)
- [প্রশ্ন ২: Database Cluster-এর ভেতরে, না বাইরে?](#প্রশ্ন-২-production-এ-database-cluster-এর-ভেতরে-রাখব-না-বাইরে)
- [প্রশ্ন ৩: Cluster-এর ভেতরে DB চালানোর নিয়ম](#প্রশ্ন-৩-db-যদি-kubernetes-এর-ভেতরেই-চালাতে-হয়-kubeadm)
- [প্রশ্ন ৪: আপনার প্রজেক্টে কোনটা বেছে নেবেন?](#প্রশ্ন-৪-আপনার-প্রজেক্টে-database-ভেতরে-থাকবে-নাকি-বাইরে)
- [প্রশ্ন ৫: External DB-এর সাথে connect করার ৩টি উপায়](#প্রশ্ন-৫-external-db-এর-সাথে-app-pod-কীভাবে-connect-হবে)
- [Production Setup: ধাপে ধাপে](#production-setup-ধাপে-ধাপে)
- [প্রশ্ন ৬: Firewall ও Network Connectivity](#প্রশ্ন-৬-firewall-ও-network-connectivity-কীভাবে-হ্যান্ডেল-করবেন)

---

## প্রশ্ন ১: App-এর মতো Database কি replicas দিয়ে scale করা যায়?

> [!IMPORTANT]
> **উত্তর: না।** App scale হয় `replicas` বাড়িয়ে, কিন্তু Database সেভাবে হয় না।

**App (API, Angular) — Stateless**
- নিজের ভেতরে কোনো ডেটা রাখে না।
- তাই ৩টা বা ১০টা pod চালালেও সমস্যা নেই।

**Database — Stateful**
- ডেটা disk-এ থাকে।
- একই ডেটা ৩টা আলাদা pod-এ আলাদা লিখলে conflict ও inconsistency হয়।

```bash
# App scale — সহজ ও নিরাপদ
kubectl scale deployment api --replicas=3

# Database-এ একই কাজ — ভুল উপায়
kubectl scale deployment mssql --replicas=3
# ৩টা pod একই volume ধরতে যাবে, অথবা ৩টা আলাদা খালি DB হবে
```

> [!TIP]
> Database-এর জন্য Deployment নয়, **StatefulSet** বা **DB Operator** লাগে (প্রশ্ন ৩ দেখুন)।

---

## প্রশ্ন ২: Production-এ Database Cluster-এর ভেতরে রাখব, না বাইরে?

> [!IMPORTANT]
> **উত্তর:** সাধারণত **বাইরে** রাখাই নিরাপদ (External Database)।

### কেন বাইরে রাখা ভালো?

- Pod যেকোনো সময় crash বা restart হতে পারে।
- Node down হলে storage সমস্যা হতে পারে।
- Backup, failover ও patching নিজে ম্যানেজ করতে হয়।

বাইরে রাখার উপায়: আলাদা VM/সার্ভার, অথবা Cloud managed service (AWS RDS, Azure SQL ইত্যাদি)।

### উদাহরণ A: DB-র hostname থাকলে (ExternalName Service)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sql-external
spec:
  type: ExternalName
  externalName: db.mycompany.com   # বাইরের DB সার্ভারের DNS
```

App-এর connection string-এ host হিসেবে `sql-external` লিখলেই হবে।

### উদাহরণ B: DB-র শুধু IP থাকলে (Service + Endpoints)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sql-external
spec:
  ports:
    - port: 1433
---
apiVersion: v1
kind: Endpoints
metadata:
  name: sql-external     # Service-এর নামের সাথে হুবহু মিলতে হবে
subsets:
  - addresses:
      - ip: 192.168.1.50   # বাইরের DB সার্ভারের IP
    ports:
      - port: 1433
```

### উদাহরণ C: Password Secret-এ রেখে App-এ পাঠানো

```bash
kubectl create secret generic db-secret \
  --from-literal=ConnectionStrings__Default="Server=sql-external,1433;Database=AppDb;User Id=sa;Password=YourStrong@Pass;TrustServerCertificate=True"
```

```yaml
# Deployment-এর container অংশে
env:
  - name: ConnectionStrings__Default
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: ConnectionStrings__Default
```

> [!NOTE]
> ASP.NET Core-এ `__` (double underscore) দিয়ে `appsettings.json`-এর `ConnectionStrings:Default` override হয়ে যায়।

---

## প্রশ্ন ৩: DB যদি Kubernetes-এর ভেতরেই চালাতে হয় (kubeadm)?

চারটি মূল নিয়ম মানতে হবে।

### ৩.১ Deployment-এর বদলে StatefulSet

StatefulSet-এ প্রতিটি pod-এর স্থায়ী নাম (`mssql-0`, `mssql-1`) ও নিজস্ব PVC থাকে। Pod restart হলেও ডেটা থেকে যায়।

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mssql
spec:
  clusterIP: None          # headless service
  selector:
    app: mssql
  ports:
    - port: 1433
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mssql
spec:
  serviceName: mssql
  replicas: 1
  selector:
    matchLabels:
      app: mssql
  template:
    metadata:
      labels:
        app: mssql
    spec:
      containers:
        - name: mssql
          image: mcr.microsoft.com/mssql/server:2022-latest
          ports:
            - containerPort: 1433
          env:
            - name: ACCEPT_EULA
              value: "Y"
            - name: MSSQL_SA_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: mssql-secret
                  key: sa-password
          resources:
            requests:
              cpu: "1"
              memory: 2Gi
            limits:
              cpu: "2"
              memory: 4Gi
          volumeMounts:
            - name: data
              mountPath: /var/opt/mssql
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: local-path
        resources:
          requests:
            storage: 20Gi
```

> [!NOTE]
> **kubeadm নোট:** kubeadm ক্লাস্টারে ডিফল্ট StorageClass থাকে না। PVC `Pending` থাকলে আগে একটা provisioner (যেমন local-path-provisioner বা NFS) ইনস্টল করতে হবে।

### ৩.২ Vertical Scaling (Scale Up) — সবচেয়ে সহজ ও প্রচলিত

Pod-এর সংখ্যা নয়, CPU/RAM বাড়ানো হয়।

```bash
kubectl patch statefulset mssql --type='json' -p='[
  {"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/memory","value":"8Gi"},
  {"op":"replace","path":"/spec/template/spec/containers/0/resources/limits/cpu","value":"4"}
]'
```

Pod restart হবে, তাই কম traffic-এর সময় করুন।

### ৩.৩ Read Replica — Read traffic scale করার উপায়

- **Primary (১টি):** write + read
- **Replica (একাধিক):** শুধু read (Primary থেকে ডেটা কপি হয়ে আসে)

```text
            ┌── Write ──► Primary (db-0)
   App ─────┤
            └── Read  ──► Replica (db-1, db-2)
```

Kubernetes-এ সাধারণত দুটো আলাদা Service থাকে:

```yaml
# Write-এর জন্য → শুধু primary
apiVersion: v1
kind: Service
metadata:
  name: db-write
spec:
  selector:
    app: db
    role: primary
  ports:
    - port: 5432
---
# Read-এর জন্য → replica-গুলো
apiVersion: v1
kind: Service
metadata:
  name: db-read
spec:
  selector:
    app: db
    role: replica
  ports:
    - port: 5432
```

.NET-এ দুটো connection string:

```json
{
  "ConnectionStrings": {
    "Write": "Host=db-write;Database=AppDb;...",
    "Read":  "Host=db-read;Database=AppDb;..."
  }
}
```

Report/list জাতীয় query **Read**-এ, insert/update **Write**-এ পাঠাতে হবে।

### ৩.৪ Database Operator ব্যবহার

Replication, failover ও backup হাতে করা কঠিন। Operator এগুলো অটোমেট করে।

- **PostgreSQL:** CloudNativePG, Zalando Postgres Operator, Patroni
- **MySQL:** MySQL Operator, Percona Operator
- **MongoDB:** MongoDB Community Operator

**উদাহরণ (CloudNativePG):** ৩টা instance — ১টা primary + ২টা replica, failover অটোমেটিক।

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Cluster
metadata:
  name: app-db
spec:
  instances: 3
  storage:
    size: 20Gi
```

> [!NOTE]
> **SQL Server নোট:** Kubernetes-এ Always On Availability Group সেটআপ করা বেশ জটিল। শেখার প্রজেক্টে সাধারণত `replicas: 1` StatefulSet অথবা বাইরের DB ব্যবহার করা হয়।

---

## প্রশ্ন ৪: আপনার প্রজেক্টে Database ভেতরে থাকবে, নাকি বাইরে?

এটা আপনার নিজের সিদ্ধান্তের বিষয়। বেছে নেওয়ার জন্য ছোট গাইড:

- **Real production, ডেটা গুরুত্বপূর্ণ** → **বাইরে** (আলাদা সার্ভার বা managed DB)
- **Learning / assignment / demo** → **ভেতরে**, StatefulSet + `replicas: 1` + PVC
- **Read traffic অনেক বেশি** → Read Replica বা Operator
- **Backup/failover নিজে ম্যানেজ করতে চান না** → Managed DB (RDS ইত্যাদি)

### সারসংক্ষেপ: App বনাম Database

**Workload type**
- App (Stateless): Deployment
- Database (Stateful): StatefulSet / Operator

**Scale পদ্ধতি**
- App: replicas বাড়ানো
- Database: Vertical scale, Read Replica

**Storage**
- App: দরকার নেই
- Database: PVC বাধ্যতামূলক

**Production সুপারিশ**
- App: Cluster-এর ভেতরে
- Database: সাধারণত Cluster-এর বাইরে

---

## প্রশ্ন ৫: External DB-এর সাথে App Pod কীভাবে connect হবে?

তিনটি উপায় আছে। তিনটিতেই App, Service বা Environment Variable-এর মাধ্যমে DB পায়।

### তিনটির পার্থক্য এক নজরে

**পদ্ধতি ১: Env variable + Secret (সরাসরি)**
- DB address থাকে Secret-এর connection string-এ
- আলাদা Service লাগে না
- Address বদলালে Secret বদলে pod restart করতে হয়
- Connection string-এ host: সরাসরি IP/hostname

**পদ্ধতি ২: ExternalName Service**
- DB address থাকে `externalName` (hostname)-এ
- আলাদা Service লাগে
- Address বদলালে Service YAML বদলান
- Connection string-এ host: `sql-external`

**পদ্ধতি ৩: Service + Endpoints**
- DB address থাকে Endpoints (IP)-এ
- আলাদা Service লাগে
- Address বদলালে Endpoints YAML বদলান
- Connection string-এ host: `sql-external`

### কোন ক্ষেত্রে কোনটা

- **Cloud managed DB (AWS RDS, Azure SQL), hostname থাকে** → ExternalName Service + Secret
- **On-prem বা আলাদা VM-এর DB, শুধু IP আছে** → Service + Endpoints + Secret
- **ছোট প্রজেক্ট, assignment, demo** → Env variable + Secret (সরাসরি)

### Production-এ কেন Service (পদ্ধতি ২ বা ৩) বেশি পছন্দ করা হয়?

- **এক জায়গায় address:** DB migrate হলে বা IP বদলালে শুধু Service/Endpoints বদলালেই হয়।
- **Pod restart লাগে না:** App-এর Secret ও Deployment ছুঁতে হয় না।
- **Address ও credential আলাদা:** Address থাকে Service-এ, password থাকে Secret-এ।
- **একাধিক app:** সবাই একই `sql-external` নাম ব্যবহার করতে পারে।

> [!IMPORTANT]
> **Production-এর নীতি:** "address আলাদা, password আলাদা।" Address থাকবে Service-এ, password থাকবে Secret-এ — code বা Deployment YAML-এ কখনো নয়।

### Secret-এর ওপরে আরও যা করা হয়

- Secret সরাসরি YAML-এ না রেখে External Secrets Operator, HashiCorp Vault বা AWS Secrets Manager থেকে আনা হয়।
- Secret-এ RBAC দিয়ে access সীমিত করা হয়।
- DB-র firewall-এ শুধু worker node-গুলোর IP allow করা হয়।

---

## Production Setup: ধাপে ধাপে

ধরুন DB-র hostname `db.mycompany.com`।

### Step 1: আলাদা namespace বানান

```bash
kubectl create namespace prod
```

### Step 2: ExternalName Service (`sql-external.yaml`)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sql-external
  namespace: prod
spec:
  type: ExternalName
  externalName: db.mycompany.com
```

শুধু IP থাকলে এর বদলে Service + Endpoints (উদাহরণ B) ব্যবহার করুন।

### Step 3: Credential Secret-এ রাখুন

```bash
kubectl create secret generic db-secret -n prod \
  --from-literal=ConnectionStrings__Default="Server=sql-external,1433;Database=AppDb;User Id=appuser;Password=StrongPass;Encrypt=True;TrustServerCertificate=False"
```

> [!WARNING]
> `sa` ব্যবহার না করে আলাদা limited-permission user (`appuser`) বানান। শুধু App-এর দরকারি table-এ access দিন।

### Step 4: Deployment-এ Secret inject করুন (`api-deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
        - name: api
          image: myrepo/api:1.0
          env:
            - name: ConnectionStrings__Default
              valueFrom:
                secretKeyRef:
                  name: db-secret
                  key: ConnectionStrings__Default
          resources:
            requests:
              cpu: 250m
              memory: 256Mi
            limits:
              cpu: 500m
              memory: 512Mi
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
```

### Step 5: NetworkPolicy দিয়ে egress সীমিত করুন

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-egress
  namespace: prod
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes: [Egress]
  egress:
    - to:
        - ipBlock:
            cidr: 192.168.1.50/32   # DB server-এর IP
      ports:
        - port: 1433
    - ports:                          # DNS-এর জন্য
        - port: 53
          protocol: UDP
```

> [!NOTE]
> NetworkPolicy কাজ করতে CNI plugin (Calico, Cilium) লাগে। kubeadm-এ Flannel থাকলে এটা enforce হবে না।

### Step 6: DB Firewall-এ শুধু Worker Node IP allow করুন

**কী?** DB server-এর firewall-এ নিয়ম দেওয়া যে শুধু আপনার worker node-গুলো port 1433-এ connect করতে পারবে, বাকি সবাই block।

**কেন দরকার?**
- DB port সবার জন্য খোলা থাকলে যে কেউ login চেষ্টা (brute force) করতে পারে।
- Password leak হলেও attacker বাইরে থেকে connect করতে পারবে না, কারণ তার IP allow করা নেই।

**Worker node IP কেন, pod IP কেন নয়?** Pod-এর IP বারবার বদলায়। তাছাড়া pod থেকে বাইরের সার্ভারে traffic যাওয়ার সময় সাধারণত node-এর IP দিয়েই বের হয় (SNAT), তাই DB server শুধু node-এর IP দেখে।

```bash
# Node-গুলোর IP বের করুন (INTERNAL-IP কলাম)
kubectl get nodes -o wide

# DB server-এ (Linux, ufw) — ধরুন 192.168.1.11 ও 192.168.1.12
sudo ufw allow from 192.168.1.11 to any port 1433 proto tcp
sudo ufw allow from 192.168.1.12 to any port 1433 proto tcp
sudo ufw deny 1433
```

- **Windows Server:** Windows Firewall → Inbound Rules → port 1433 → Scope → Remote IP address-এ শুধু node IP-গুলো।
- **AWS:** Security Group-এ inbound rule: port 1433, source = worker node-দের Security Group বা IP।

> [!NOTE]
> নতুন worker node যোগ করলে তার IP-ও allow করতে হবে, নইলে ওই node-এর pod DB-তে connect হবে না।

### Step 7: Secret RBAC দিয়ে সীমিত করুন

**কেন দরকার?** যার `prod` namespace-এ যথেষ্ট access আছে, সে এই command দিয়ে DB password পেয়ে যেতে পারে:

```bash
kubectl get secret db-secret -n prod -o jsonpath='{.data.ConnectionStrings__Default}' | base64 -d
```

> [!CAUTION]
> Secret শুধু base64 encoded, encrypted নয়। Access সীমিত না করলে developer, intern বা যেকোনো tool-এর service account password পড়ে ফেলতে পারে।

**১) Role — শুধু নির্দিষ্ট Secret পড়ার অনুমতি**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: db-secret-reader
  namespace: prod
rules:
  - apiGroups: [""]
    resources: ["secrets"]
    resourceNames: ["db-secret"]   # শুধু এই Secret
    verbs: ["get"]
```

**২) RoleBinding — যাকে দেবেন তার সাথে bind করুন**

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: db-secret-reader-binding
  namespace: prod
subjects:
  - kind: ServiceAccount
    name: api-sa
    namespace: prod
roleRef:
  kind: Role
  name: db-secret-reader
  apiGroup: rbac.authorization.k8s.io
```

**৩) যাচাই করুন**

```bash
kubectl auth can-i get secret/db-secret -n prod --as=system:serviceaccount:prod:api-sa
# yes

kubectl auth can-i get secret/db-secret -n prod --as=system:serviceaccount:prod:other-sa
# no
```

> [!TIP]
> App pod `secretKeyRef` দিয়ে env variable নিলে সেটা kubelet করে, তাই app-এর service account-এ আলাদা Secret-read permission না দিলেও চলে। এই RBAC মূলত মানুষ ও অন্য tool-দের আটকানোর জন্য।

### Step 6 ও 7 একসাথে কী সুরক্ষা দেয়

- **DB Firewall:** বাইরের কেউ সরাসরি DB-তে connect করা আটকায়।
- **Secret RBAC:** Cluster-এর ভেতরের অনধিকারী কারও password পড়া আটকায়।

---

## প্রশ্ন ৬: Firewall ও Network Connectivity কীভাবে হ্যান্ডেল করবেন?

এখানে আসলে দুটো আলাদা সমস্যা আছে। একটা বাড়ির উদাহরণে বোঝা যাক:

- **Firewall = বাড়ির গেটের দারোয়ান।** তালিকায় নাম না থাকলে ঢুকতে দেয় না। প্রশ্ন: *গেটে ঢুকতে দেবে কি?*
- **Network connectivity = বাড়ি পর্যন্ত যাওয়ার রাস্তা।** রাস্তা না থাকলে অনুমতি থাকলেও পৌঁছানো যায় না। প্রশ্ন: *রাস্তা আছে কি?*

App pod-এর DB-তে পৌঁছাতে **দুটোই** ঠিক থাকতে হবে।

### সমস্যা ১: Firewall — DB-র গেটে অনুমতি দেওয়া

DB server-কে বলে দিন: "আমার Kubernetes node-গুলোর IP থেকে আসা connection ঢুকতে দাও।" আগে DB-র port মনে রাখুন:

- **SQL Server:** `1433`
- **PostgreSQL:** `5432`
- **MySQL:** `3306`

#### ক) DB Cloud-এ থাকলে (AWS RDS)

Cloud-এ firewall-কে বলে **Security Group**:

1. AWS Console → RDS → আপনার DB → Security Group
2. Inbound rules → Add rule
3. Type বাছুন (MySQL / PostgreSQL / MS SQL), port নিজে বসে যাবে
4. Source-এ Kubernetes node-গুলোর IP বা subnet দিন (যেমন `192.168.10.0/24`)

> [!CAUTION]
> Source-এ `0.0.0.0/0` দেবেন না। এর মানে "পৃথিবীর সবাই ঢুকতে পারবে", যা ভয়ংকর।

#### খ) DB নিজের Physical server বা VM-এ থাকলে

```bash
# node IP 192.168.1.10 কে MySQL / PostgreSQL-এ ঢুকতে দাও
sudo ufw allow from 192.168.1.10 to any port 3306 proto tcp
sudo ufw allow from 192.168.1.10 to any port 5432 proto tcp
```

এর সাথে DB-কেও বলতে হবে যে সে বাইরের connection শুনবে (ডিফল্টে শুধু নিজের মেশিন থেকে নেয়):

- **MySQL:** `mysqld.cnf`-এ `bind-address = 0.0.0.0`
- **PostgreSQL:** `postgresql.conf`-এ `listen_addresses = '*'`, আর `pg_hba.conf`-এ node IP allow করতে হয়
- **Windows Server (SQL Server):** Windows Defender Firewall → Inbound Rules → New Rule → Port → TCP 1433 → Scope-এ node IP

### সমস্যা ২: Network Connectivity — রাস্তা তৈরি করা

Firewall খুললেও Kubernetes ক্লাস্টার আর DB আলাদা private network-এ থাকলে তারা একে অপরকে দেখতেই পায় না। তখন দুই network জুড়ে দিতে হয়।

**ক) দুটোই AWS-এ, কিন্তু আলাদা VPC-তে**
- সমাধান: **VPC Peering** + দুই দিকের Route Table-এ একে অপরের subnet যোগ করা।
- Traffic ইন্টারনেট দিয়ে না গিয়ে private IP দিয়ে যায়, তাই বেশি নিরাপদ।

**খ) DB অফিসের Physical server room-এ, ক্লাস্টার অন্য জায়গায়**
- সমাধান: **Site-to-Site VPN** (নিরাপদ টানেল), অথবা dedicated লাইন (AWS Direct Connect / Azure ExpressRoute)।
- ফলে DB-টা ক্লাস্টারের নিজের local network-এর মতো আচরণ করে।

**গ) ক্লাস্টার ও DB একই network-এ**
- আলাদা কিছু লাগে না, শুধু Firewall ঠিক করলেই চলে। **আপনার assignment-এ সম্ভবত এটাই হবে।**

### সব ঠিক আছে কিনা test করুন

ক্লাস্টারের যেকোনো node-এ ঢুকে চালান:

```bash
nc -zv <DB_IP> 1433    # SQL Server
nc -zv <DB_IP> 5432    # PostgreSQL
nc -zv <DB_IP> 3306    # MySQL
```

- **`succeeded` / `open`** → সব ঠিক আছে, রাস্তাও আছে, গেটও খোলা।
- **`timed out`** → সাধারণত রাস্তা নেই (routing/VPN), অথবা firewall চুপচাপ block করছে।
- **`refused`** → DB পর্যন্ত পৌঁছেছে, কিন্তু DB বাইরের connection নিচ্ছে না (`bind-address` ইত্যাদি দেখুন)।

### এক লাইনে সারাংশ

- **Firewall** (গেটে অনুমতি নেই) → DB-র firewall / Security Group-এ node IP allow করা।
- **Connectivity** (রাস্তাই নেই) → VPC Peering / VPN / Direct Connect।
