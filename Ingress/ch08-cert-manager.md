# Chapter 8: cert-manager ও Let's Encrypt — Certificate স্বয়ংক্রিয়ভাবে

> Chapter 7-এ আমরা হাতে certificate বানিয়েছি। Production-এ certificate ৯০ দিনে মেয়াদ শেষ হয় (Let's Encrypt), তাই হাতে renew করা চলে না। **cert-manager** এটা নিজে নিজে করে।

## 8.1 cert-manager কী করে

```
Ingress (annotation) ─► cert-manager ─► CA (Let's Encrypt) ─► Certificate
                                                   │
                                      Secret-এ রাখে ◄┘  (মেয়াদ শেষের আগেই নিজে renew)
```

আপনি শুধু বলেন "এই domain-এর certificate চাই", বাকিটা cert-manager করে।

## 8.2 Install

```bash
helm upgrade --install cert-manager cert-manager \
  --repo https://charts.jetstack.io \
  --namespace cert-manager --create-namespace \
  --set crds.enabled=true
```

> পুরনো chart-এ flag ছিল `installCRDs=true`; নতুনে `crds.enabled=true`। কাজ না করলে chart-এর docs দেখুন।

যাচাই:

```bash
kubectl get pods -n cert-manager      # ৩টা Pod: cert-manager, cainjector, webhook
```

## 8.3 Issuer ও ClusterIssuer

- **Issuer** — একটা namespace-এর জন্য
- **ClusterIssuer** — পুরো cluster-এর জন্য (সাধারণত এটাই নেওয়া হয়)

## 8.4 Lab-এ: নিজস্ব CA দিয়ে (domain/ইন্টারনেট ছাড়াই)

Chapter 7-এর `ca.crt`/`ca.key` cert-manager-কে দিন, সে সব certificate বানিয়ে দেবে।

```bash
kubectl create secret tls bmi-lab-ca \
  --cert=certs/ca.crt --key=certs/ca.key \
  -n cert-manager
```

```yaml
# k8s/ingress/cert-manager/clusterissuer-lab.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: bmi-lab-ca
spec:
  ca:
    secretName: bmi-lab-ca     # ClusterIssuer-এর ক্ষেত্রে cert-manager namespace-এর Secret
```

Ingress-এ শুধু annotation:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: bmi-lab-ca
spec:
  tls:
    - hosts: [bmi.local]
      secretName: bmi-local-tls      # cert-manager এই Secret নিজে বানাবে
```

দেখুন:

```bash
kubectl get certificate -n scaling-demo
kubectl describe certificate bmi-local-tls -n scaling-demo
```

`READY = True` হলে সফল।

## 8.5 Production: Let's Encrypt (সত্যিকারের domain লাগবে)

### পূর্বশর্ত

1. আপনার নিজের **domain** (যেমন `bmi.example.com`)
2. DNS-এ domain → Ingress-এর **public IP** (A record)
3. Port **80** ইন্টারনেট থেকে খোলা (HTTP-01 challenge এর জন্য)
4. Cluster-এ বাইরে থেকে আসা traffic Ingress-এ পৌঁছায়

> `bmi.local` এর জন্য Let's Encrypt কাজ করে না, কারণ এটা public domain নয়।

### আগে Staging দিয়ে পরীক্ষা

Production-এ রেট-লিমিট আছে। ভুল করলে কিছুক্ষণ ব্লক হয়ে যাবেন। তাই আগে staging:

```yaml
# clusterissuer-letsencrypt.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: you@example.com
    privateKeySecretRef:
      name: letsencrypt-prod-key
    solvers:
      - http01:
          ingress:
            ingressClassName: nginx
```

```bash
kubectl apply -f clusterissuer-letsencrypt.yaml
kubectl get clusterissuer
```

Ingress-এ:

```yaml
metadata:
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging   # পরে letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
    - hosts: [bmi.example.com]
      secretName: bmi-example-tls
```

Staging সফল হলে (browser এ untrusted দেখাবে, এটা স্বাভাবিক) annotation বদলে `letsencrypt-prod` করুন এবং পুরনো secret মুছে দিন:

```bash
kubectl delete secret bmi-example-tls -n scaling-demo
```

## 8.6 কীভাবে কাজ করে (HTTP-01)

1. Ingress-এ annotation দেখে cert-manager একটা `Certificate` বানায়
2. Let's Encrypt বলে "প্রমাণ করো `bmi.example.com` তোমার"
3. cert-manager একটা অস্থায়ী Ingress/Pod বানিয়ে `/.well-known/acme-challenge/...` এ উত্তর রাখে
4. Let's Encrypt ইন্টারনেট থেকে ওই URL দেখে যাচাই করে
5. Certificate ইস্যু হয়ে Secret-এ জমা হয়; Ingress সেটা ব্যবহার শুরু করে
6. মেয়াদ শেষের ~৩০ দিন আগে নিজে renew

## 8.7 Wildcard (`*.example.com`) লাগলে

HTTP-01 দিয়ে হয় না; **DNS-01** লাগে (DNS provider-এর API token দিতে হয়, যেমন Cloudflare/Route53)। প্রথমে HTTP-01 দিয়ে শুরু করুন।

## 8.8 Troubleshooting

```bash
kubectl get certificate,certificaterequest,order,challenge -A
kubectl describe challenge -n scaling-demo
kubectl logs -n cert-manager deploy/cert-manager --tail=50
```

| সমস্যা | কারণ |
|---|---|
| Challenge `pending` থাকে | DNS ভুল IP-তে / port 80 বন্ধ / firewall |
| `too many certificates` | Let's Encrypt রেট-লিমিট — staging ব্যবহার করুন |
| Secret তৈরি হয় না | Annotation নাম ভুল, ClusterIssuer নাম মেলেনি |

## 8.9 সারসংক্ষেপ

| পরিবেশ | Issuer |
|---|---|
| Local lab | নিজস্ব CA (`ca` issuer) |
| Test | `letsencrypt-staging` |
| Production | `letsencrypt-prod` |

পরের chapter: [Chapter 9 — Client Certificate ও mTLS](ch09-client-certificate-mtls.md)
