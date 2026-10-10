# Chapter 10: Troubleshooting ও Cheat Sheet

## 10.1 সমস্যা ধরার ক্রম (উপর থেকে নিচে)

```
Client ─► DNS/hosts ─► Ingress Controller ─► Ingress নিয়ম ─► Service ─► Endpoints ─► Pod ─► App
```

প্রতিটা ধাপ একে একে পরীক্ষা করুন।

## 10.2 ধাপে ধাপে যাচাই

```bash
# ১. Controller চলছে?
kubectl get pods -n ingress-nginx

# ২. Ingress ঠিক আছে? (Address ও Backends দেখুন)
kubectl get ingress -A
kubectl describe ingress scaling-demo -n scaling-demo

# ৩. Service-এর পেছনে Pod আছে?  (খালি হলে selector/label ভুল)
kubectl get endpoints -n scaling-demo

# ৪. Pod চলছে ও ready?
kubectl get pods -n scaling-demo
kubectl logs deploy/scaling-demo-api -n scaling-demo --tail=50

# ৫. Cluster-এর ভেতর থেকে Service কাজ করে?
kubectl run tmp --rm -it --image=curlimages/curl -n scaling-demo -- \
  curl -i http://scaling-demo-api/health

# ৬. Controller log
kubectl logs -n ingress-nginx deploy/ingress-nginx-controller --tail=100
```

## 10.3 Error কোড অনুযায়ী

| Error | সম্ভাব্য কারণ |
|---|---|
| **404** (NGINX-এর নিজের) | host/path মেলেনি, `hosts` ফাইল ভুল, `ingressClassName` ভুল |
| **404** (আপনার app-এর) | Ingress ঠিক, কিন্তু app path চেনে না → rewrite/`UsePathBase` দেখুন |
| **502 Bad Gateway** | Pod নেই/crash, `targetPort` ভুল, port নম্বর মেলেনি |
| **503 Service Unavailable** | Endpoints খালি (readiness fail বা label mismatch) |
| **504 Gateway Timeout** | App ধীর — `proxy-read-timeout` বাড়ান বা app ঠিক করুন |
| **413 Request Entity Too Large** | `proxy-body-size` বাড়ান |
| **400 No required SSL certificate** | mTLS-এ client certificate দেননি |
| **308 loop / ERR_TOO_MANY_REDIRECTS** | app ও Ingress দুজনেই redirect করছে → forwarded headers ঠিক করুন |
| **Fake Certificate** | TLS Secret নাম/namespace ভুল |

## 10.4 Certificate যাচাই কমান্ড

```bash
# certificate-এর তথ্য
openssl x509 -in server.crt -noout -subject -issuer -dates -ext subjectAltName

# মেয়াদ কবে শেষ
openssl x509 -in server.crt -noout -enddate

# chain যাচাই
openssl verify -CAfile ca.crt server.crt

# সার্ভার আসলে কোন certificate দিচ্ছে
openssl s_client -connect localhost:443 -servername bmi.local </dev/null

# key আর certificate মেলে কিনা (দুটোর hash এক হতে হবে)
openssl x509 -in server.crt -noout -modulus | openssl md5
openssl rsa  -in server.key -noout -modulus | openssl md5

# Secret থেকে certificate বের করে দেখা
kubectl get secret bmi-local-tls -n scaling-demo -o jsonpath='{.data.tls\.crt}' \
  | base64 -d | openssl x509 -noout -subject -dates
```

## 10.5 Cheat Sheet

| কাজ | কমান্ড/নিয়ম |
|---|---|
| Controller install | `helm upgrade --install ingress-nginx ingress-nginx --repo https://kubernetes.github.io/ingress-nginx -n ingress-nginx --create-namespace` |
| Ingress তালিকা | `kubectl get ingress -A` |
| TLS Secret | `kubectl create secret tls NAME --cert=server.crt --key=server.key -n NS` |
| CA Secret (mTLS) | `kubectl create secret generic client-ca --from-file=ca.crt=ca.crt -n NS` |
| Service type | Ingress-এর পেছনে `ClusterIP` |
| Path match | সাধারণত `Prefix` |
| HTTPS | `spec.tls[].secretName` |
| Auto certificate | cert-manager + `cert-manager.io/cluster-issuer` annotation |
| mTLS | `auth-tls-verify-client: "on"` + `auth-tls-secret` |

## 10.6 সম্পূর্ণ ছবি (এক নজরে)

```
Browser/curl
   │  https://bmi.local         (server certificate যাচাই — Ch 6,7)
   │  + client certificate      (শুধু mTLS হলে — Ch 9)
   ▼
[ LoadBalancer — একটাই IP ]
   ▼
[ Ingress Controller (NGINX) ]   ← TLS এখানেই শেষ (termination)
   │   নিয়ম পড়ে: host + path    (Ch 3,4)
   ├── /api ─► Service scaling-demo-api ─► api Pods
   └── /    ─► Service scaling-demo-web ─► web Pods
        ▲
   cert-manager Secret-এ certificate রাখে ও renew করে (Ch 8)
```

## 10.7 শেখার ক্রম (মনে রাখার জন্য)

1. Pod → Service → Ingress কেন (Ch 1)
2. Controller install (Ch 2)
3. প্রথম Ingress (Ch 3)
4. Routing নিয়ম (Ch 4)
5. Production অভ্যাস (Ch 5)
6. TLS তত্ত্ব (Ch 6)
7. HTTPS চালু (Ch 7)
8. Auto certificate (Ch 8)
9. mTLS (Ch 9)

## 10.8 পরবর্তী ধাপ

- Gateway API (Ingress-এর নতুন রূপ) শেখা
- Service mesh (Istio/Linkerd) ও Pod-to-Pod mTLS
- Ingress-এর পেছনে HPA (আপনার আগের Kubernetes HPA chapter-এর সাথে জুড়ে load test করা)
