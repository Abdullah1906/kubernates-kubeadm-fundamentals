# Chapter 3: প্রথম Ingress — BMI App দিয়ে ধাপে ধাপে

> লক্ষ্য: একটাই ঠিকানা `http://bmi.local` থেকে frontend (`/`) আর API (`/api`) দুটোই চালানো।

## 3.1 আমরা কী বানাচ্ছি

```
http://bmi.local/      → scaling-demo-web  (Angular, port 80)
http://bmi.local/api   → scaling-demo-api  (ASP.NET Core, port 8080)
```

> নিচের image নাম (`yourname/scaling-demo-web:1.0` ইত্যাদি) আপনার নিজের image দিয়ে বদলে নিন।

## 3.2 ধাপ ১: Namespace

```yaml
# k8s/ingress/00-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: scaling-demo
```

## 3.3 ধাপ ২: Deployment + Service (ClusterIP)

```yaml
# k8s/ingress/10-api.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaling-demo-api
  namespace: scaling-demo
spec:
  replicas: 2
  selector:
    matchLabels: { app: scaling-demo-api }
  template:
    metadata:
      labels: { app: scaling-demo-api }
    spec:
      containers:
        - name: api
          image: yourname/scaling-demo-api:1.0
          ports:
            - containerPort: 8080
          readinessProbe:
            httpGet: { path: /health, port: 8080 }
---
apiVersion: v1
kind: Service
metadata:
  name: scaling-demo-api
  namespace: scaling-demo
spec:
  type: ClusterIP          # LoadBalancer লাগবে না!
  selector: { app: scaling-demo-api }
  ports:
    - port: 80
      targetPort: 8080
```

```yaml
# k8s/ingress/20-web.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaling-demo-web
  namespace: scaling-demo
spec:
  replicas: 2
  selector:
    matchLabels: { app: scaling-demo-web }
  template:
    metadata:
      labels: { app: scaling-demo-web }
    spec:
      containers:
        - name: web
          image: yourname/scaling-demo-web:1.0
          ports:
            - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: scaling-demo-web
  namespace: scaling-demo
spec:
  type: ClusterIP
  selector: { app: scaling-demo-web }
  ports:
    - port: 80
      targetPort: 80
```

> `/health` endpoint আপনার API-তে না থাকলে probe-টা সরিয়ে দিন বা `app.MapHealthChecks("/health")` যোগ করুন।

## 3.4 ধাপ ৩: Ingress

```yaml
# k8s/ingress/30-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scaling-demo
  namespace: scaling-demo
spec:
  ingressClassName: nginx
  rules:
    - host: bmi.local
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: scaling-demo-api
                port:
                  number: 80
          - path: /
            pathType: Prefix
            backend:
              service:
                name: scaling-demo-web
                port:
                  number: 80
```

ফাইলের অংশগুলো বুঝি:

| অংশ | অর্থ |
|---|---|
| `ingressClassName: nginx` | কোন Controller এই নিয়ম পড়বে |
| `host: bmi.local` | এই domain-এর request-এর জন্য নিয়ম |
| `path: /api` | `/api` দিয়ে শুরু হলে API Service-এ |
| `pathType: Prefix` | prefix মিললেই হবে (`/api/bmi` ও মিলবে) |
| `backend.service` | **Service-এর নাম ও port** (Pod নয়) |

NGINX সবচেয়ে **লম্বা/নির্দিষ্ট path আগে** মেলায়, তাই `/api` আগে মেলে, বাকি সব `/`-তে যায়।

## 3.5 ধাপ ৪: Apply করা

```bash
kubectl apply -f k8s/ingress/
kubectl get all -n scaling-demo
kubectl get ingress -n scaling-demo
```

`ADDRESS` কলামে IP/`localhost` আসতে এক-দুই মিনিট লাগতে পারে।

## 3.6 ধাপ ৫: `bmi.local` কে নিজের পিসিতে চেনানো

DNS নেই, তাই `hosts` ফাইলে লিখতে হবে।

**Windows** (Notepad কে *Run as administrator* করে খুলুন):

```
C:\Windows\System32\drivers\etc\hosts
```

শেষে যোগ করুন:

```
127.0.0.1  bmi.local
```

**Linux/macOS:**

```bash
echo "127.0.0.1 bmi.local" | sudo tee -a /etc/hosts
```

> minikube-তে `127.0.0.1` এর বদলে `minikube ip` এর IP দিন (tunnel ব্যবহার করলে `127.0.0.1`)।

## 3.7 ধাপ ৬: পরীক্ষা

```bash
curl -i http://bmi.local/
curl -i http://bmi.local/api/health
```

hosts ফাইল না বদলে দ্রুত পরীক্ষা করতে চাইলে Host header দিয়ে:

```bash
curl -i -H "Host: bmi.local" http://localhost/api/health
```

Browser-এ `http://bmi.local` খুললে Angular app দেখা যাবে।

## 3.8 Angular-এ API URL

Angular থেকে API call করার সময় **relative URL** ব্যবহার করুন:

```ts
// environment.ts
export const environment = { apiUrl: '/api' };
```

এতে একই origin থেকে request যায়, তাই **CORS সমস্যাই হয় না**। এটা Ingress ব্যবহারের একটা বড় সুবিধা।

## 3.9 সারসংক্ষেপ

1. Service গুলো `ClusterIP` রাখুন
2. `Ingress` এ host + path → Service map করুন
3. `hosts` ফাইলে domain চেনান
4. `curl`/browser দিয়ে পরীক্ষা করুন

পরের chapter: [Chapter 4 — Path, Host, Rewrite, Annotation](ch04-routing-rules.md)
