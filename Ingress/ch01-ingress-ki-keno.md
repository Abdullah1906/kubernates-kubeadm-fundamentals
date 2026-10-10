# Chapter 1: Ingress কী এবং কেন লাগে?

> এই chapter-এ আমরা বুঝব: Service থাকতে Ingress কেন দরকার, আর দুটোর সম্পর্ক কী।

## 1.1 আগে Service-টা মনে করি

Kubernetes-এ Pod-এর IP বারবার বদলায় (Pod মরে গেলে নতুন Pod নতুন IP পায়)। তাই আমরা **Service** বানাই — এটা একটা স্থায়ী নাম + IP, যেটা পেছনের Pod গুলোকে load balance করে।

| Service type | কাজ | সমস্যা |
|---|---|---|
| `ClusterIP` | শুধু cluster-এর ভেতর থেকে access | বাইরে থেকে ঢোকা যায় না |
| `NodePort` | প্রতিটা Node-এ একটা port (30000–32767) খোলে | port মনে রাখা কঠিন, HTTPS/path routing নেই |
| `LoadBalancer` | Cloud provider থেকে আলাদা public IP আনে | **প্রতিটা Service = একটা আলাদা LoadBalancer = আলাদা খরচ** |

## 1.2 সমস্যাটা কোথায়?

আমাদের BMI app-এ আছে:

- `scaling-demo-web` (Angular frontend)
- `scaling-demo-api` (ASP.NET Core API)

দুটোকেই `LoadBalancer` দিলে দুটো public IP লাগবে। ১০টা service হলে ১০টা IP আর ১০টা বিল! তাছাড়া:

- `bmi.example.com` এ frontend আর `bmi.example.com/api` এ API — এটা Service দিয়ে করা যায় না।
- HTTPS certificate প্রতিটা service-এ আলাদা আলাদা বসাতে হয়।

## 1.3 Ingress কী?

**Ingress = HTTP/HTTPS traffic-এর জন্য routing নিয়মের তালিকা।**

উদাহরণ নিয়ম:

- `bmi.local/` → `scaling-demo-web` Service
- `bmi.local/api` → `scaling-demo-api` Service

Ingress দিয়ে আমরা পাই:

1. একটাই public entry point (একটাই IP)
2. Path-based routing (`/api`, `/admin`)
3. Host-based routing (`api.example.com`, `app.example.com`)
4. একজায়গায় HTTPS/TLS (certificate এক জায়গায়)
5. Redirect, rate limit, body size limit ইত্যাদি

## 1.4 Ingress vs Ingress Controller (খুব গুরুত্বপূর্ণ)

| | কী | উদাহরণ |
|---|---|---|
| **Ingress** | শুধু একটা YAML — "নিয়ম" | `kind: Ingress` |
| **Ingress Controller** | যে Pod নিয়মগুলো পড়ে আসলে traffic route করে | NGINX, Traefik, HAProxy |

> **Ingress YAML লিখলেই কাজ হয় না। Ingress Controller install করা না থাকলে Ingress কিছুই করবে না।**

উপমা: Ingress হলো "কোন বিভাগে কোন extension" লেখা কাগজ, আর Controller হলো রিসেপশনিস্ট যে কাগজ দেখে কল ট্রান্সফার করে।

## 1.5 Service আগে লাগে কেন?

Ingress **সরাসরি Pod-এ** traffic পাঠায় না। Ingress-এর `backend` হলো একটা **Service**। তাই ক্রম হলো:

```
Deployment (Pod) → Service (ClusterIP) → Ingress → বাইরের user
```

অর্থাৎ Service ছাড়া Ingress চলে না। আর Ingress ব্যবহার করলে Service-কে `ClusterIP` রাখলেই যথেষ্ট — আলাদা LoadBalancer লাগে না।

## 1.6 পুরো ছবি

```
                    Internet
                       │
              ┌────────▼────────┐
              │  একটাই Public IP │  (LoadBalancer — শুধু Controller-এর জন্য)
              └────────┬────────┘
                       │
          ┌────────────▼─────────────┐
          │   Ingress Controller     │  (NGINX Pod)
          │   Ingress নিয়ম পড়ে      │
          └──────┬───────────┬───────┘
          /      │           │      /api
      ┌──────────▼──┐   ┌────▼───────────┐
      │ web Service │   │ api Service    │   (ClusterIP)
      └──────┬──────┘   └────┬───────────┘
             │               │
         web Pods        api Pods
```

## 1.7 সারসংক্ষেপ

- Service = Pod-এর স্থায়ী ঠিকানা (cluster-এর ভেতরে)
- Ingress = বাইরে থেকে আসা HTTP(S) traffic-এর routing নিয়ম
- Ingress Controller = নিয়ম কার্যকর করা Pod
- Ingress-এর backend সবসময় Service

পরের chapter: [Chapter 2 — Ingress Controller install](ch02-controller-install.md)
