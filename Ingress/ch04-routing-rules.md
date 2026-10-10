# Chapter 4: Routing নিয়ম — Host, Path, Rewrite, Annotation

## 4.1 pathType তিন রকম

| pathType | আচরণ | উদাহরণ (`path: /api`) |
|---|---|---|
| `Prefix` | `/` দিয়ে ভাগ করা prefix মেলে | `/api`, `/api/bmi` মেলে; `/apixyz` মেলে না |
| `Exact` | হুবহু মিলতে হবে | শুধু `/api` মেলে, `/api/bmi` না |
| `ImplementationSpecific` | Controller যেভাবে ঠিক করে | সাধারণত regex-এর জন্য (এড়িয়ে চলুন) |

সাধারণত `Prefix` ব্যবহার করুন; `Exact` ব্যবহার করুন নির্দিষ্ট endpoint-এর জন্য।

## 4.2 Host-based routing

একই cluster-এ আলাদা subdomain:

```yaml
spec:
  ingressClassName: nginx
  rules:
    - host: app.bmi.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: scaling-demo-web, port: { number: 80 } }
    - host: api.bmi.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service: { name: scaling-demo-api, port: { number: 80 } }
```

`hosts` ফাইলে দুটো নামই যোগ করতে হবে:

```
127.0.0.1  app.bmi.local api.bmi.local
```

কখন কোনটা? 

- **Path-based** (`/api`) — ছোট প্রজেক্ট, একই domain, CORS নেই। 
- **Host-based** (`api.`) — বড় প্রজেক্ট, আলাদা team/আলাদা certificate/আলাদা policy।

## 4.3 Rewrite — path বদলে পাঠানো

ধরুন API নিজে `/api` চেনে না, চেনে `/bmi`; কিন্তু বাইরে থেকে আমরা `/api/bmi` চাই। তখন `/api` অংশটা কেটে পাঠাতে হয়:

```yaml
metadata:
  annotations:
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
    - host: bmi.local
      http:
        paths:
          - path: /api(/|$)(.*)
            pathType: ImplementationSpecific
            backend:
              service: { name: scaling-demo-api, port: { number: 80 } }
```

ফলাফল: `/api/bmi/calc` → API পায় `/bmi/calc`।

> **পরামর্শ:** Rewrite এড়াতে পারলে এড়ান। সহজ উপায়: ASP.NET Core API-তে `app.UsePathBase("/api")` বা controller route-এর শুরুতে `api/` রাখুন। কম জটিলতা = কম বাগ।

**সতর্কতা:** একই Ingress-এ rewrite annotation দিলে সেটা ওই Ingress-এর **সব path-এ** প্রযোজ্য হয়। তাই rewrite লাগা path আলাদা Ingress-এ রাখুন (frontend-এর `/` কে rewrite-এর আওতায় ফেলবেন না)।

## 4.4 দরকারি Annotation (ingress-nginx)

```yaml
metadata:
  annotations:
    # বড় body (ফাইল আপলোড) — default মাত্র 1MB
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"

    # timeout (সেকেন্ড)
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "5"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # HTTP → HTTPS redirect (TLS থাকলে default on)
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # rate limit: প্রতি IP থেকে প্রতি সেকেন্ডে ১০ request
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # নির্দিষ্ট IP-র জন্যই অনুমতি
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,203.0.113.5/32"
```

> Annotation-এর নাম controller ভেদে আলাদা। Traefik/HAProxy নিলে তাদের docs দেখুন।

## 4.5 Default backend

কোনো নিয়ম না মিললে কোথায় যাবে:

```yaml
spec:
  ingressClassName: nginx
  defaultBackend:
    service:
      name: scaling-demo-web
      port: { number: 80 }
```

Angular SPA-র জন্য (refresh দিলে `/result` এ 404 না আসতে) frontend container-এর nginx-এ `try_files $uri /index.html;` রাখুন — এটা Ingress নয়, web container-এর কাজ।

## 4.6 Ingress নিয়ে debug

```bash
kubectl describe ingress scaling-demo -n scaling-demo
kubectl get endpoints -n scaling-demo
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=50
```

`describe` এর `Backends` এ IP:port দেখা না গেলে Service-এর selector/label ভুল।

পরের chapter: [Chapter 5 — Professional grade ব্যবহার](ch05-production-practices.md)
