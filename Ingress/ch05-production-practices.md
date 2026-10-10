# Chapter 5: Professional Grade Ingress ব্যবহার

> Lab-এ যা চলে, production-এ তা যথেষ্ট নয়। এই chapter-এ একটা checklist আর উদাহরণ দেওয়া হলো।

## 5.1 Production Checklist

| # | কী করবেন | কেন |
|---|---|---|
| 1 | Controller কে **একাধিক replica** (২–৩) চালান | একটা Pod মরলে সব বন্ধ হয়ে যাবে |
| 2 | `PodDisruptionBudget` দিন | node maintenance-এ সব replica একসাথে নামবে না |
| 3 | Controller-এ `resources` (requests/limits) দিন | অন্য workload চাপে পড়বে না |
| 4 | সবসময় `ingressClassName` লিখুন | ভুল controller নিয়ম নেবে না |
| 5 | HTTPS বাধ্যতামূলক, TLS auto-renew (cert-manager) | certificate মেয়াদ শেষ হয়ে outage এড়াতে |
| 6 | Backend Pod-এ `readinessProbe` | অপ্রস্তুত Pod-এ traffic যাবে না |
| 7 | Rate limit, body size, timeout ঠিক করুন | abuse ও hanging connection ঠেকাতে |
| 8 | Security header যোগ করুন | XSS/clickjacking ঝুঁকি কমে |
| 9 | Ingress YAML Git-এ রাখুন (GitOps) | কে কী বদলেছে ট্র্যাক হয় |
| 10 | Log ও metrics monitor করুন | সমস্যা আগে ধরা যায় |
| 11 | Namespace ও RBAC আলাদা রাখুন | এক team অন্য team-এর route নষ্ট করবে না |
| 12 | Internal API সরাসরি public করবেন না | attack surface কমানো |

## 5.2 Controller-এর জন্য `values.yaml` (Helm)

```yaml
# helm/ingress-nginx-values.yaml
controller:
  replicaCount: 2
  ingressClassResource:
    name: nginx
    default: true
  resources:
    requests: { cpu: 100m, memory: 128Mi }
    limits:   { cpu: 500m, memory: 512Mi }
  minAvailable: 1                # PodDisruptionBudget
  metrics:
    enabled: true                # Prometheus এর জন্য
  config:
    use-forwarded-headers: "true"
    hsts: "true"
    server-tokens: "false"       # NGINX version লুকানো
    proxy-body-size: "10m"
    log-format-upstream: >-
      $remote_addr - $request_id [$time_local] "$request"
      $status $body_bytes_sent $request_time $upstream_addr
  topologySpreadConstraints:
    - maxSkew: 1
      topologyKey: kubernetes.io/hostname
      whenUnsatisfiable: ScheduleAnyway
      labelSelector:
        matchLabels:
          app.kubernetes.io/name: ingress-nginx
```

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  -n ingress-nginx --create-namespace \
  -f helm/ingress-nginx-values.yaml
```

> আপনার Helm chart-এর version অনুযায়ী key-এর নাম সামান্য আলাদা হতে পারে। `helm show values ingress-nginx --repo https://kubernetes.github.io/ingress-nginx` দিয়ে দেখে নিন।

## 5.3 Production-মানের Ingress উদাহরণ

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scaling-demo
  namespace: scaling-demo
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod     # Chapter 8
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "5m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "30"
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
      more_set_headers "Referrer-Policy: strict-origin-when-cross-origin";
spec:
  ingressClassName: nginx
  tls:
    - hosts: [bmi.example.com]
      secretName: bmi-example-tls
  rules:
    - host: bmi.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service: { name: scaling-demo-api, port: { number: 80 } }
          - path: /
            pathType: Prefix
            backend:
              service: { name: scaling-demo-web, port: { number: 80 } }
```

> `configuration-snippet` কিছু cluster-এ (security কারণে) বন্ধ থাকে। বন্ধ থাকলে header গুলো frontend-এর nginx.conf-এ বা controller-এর `add-headers` ConfigMap দিয়ে দিন।

## 5.4 Backend Pod-কে প্রস্তুত রাখা

```yaml
readinessProbe:
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 5
  periodSeconds: 5
livenessProbe:
  httpGet: { path: /health, port: 8080 }
  initialDelaySeconds: 15
  periodSeconds: 10
```

ASP.NET Core-এ:

```csharp
builder.Services.AddHealthChecks();
app.MapHealthChecks("/health");
```

**Rolling update-এ downtime ছাড়া** deploy করতে readinessProbe অপরিহার্য — না থাকলে নতুন Pod তৈরি হওয়ামাত্র traffic পাবে, অথচ app তখনও চালু হয়নি।

## 5.5 Real client IP

Ingress-এর পেছনে থাকলে API-তে `RemoteIpAddress` হবে Ingress Pod-এর IP। আসল IP পেতে:

```csharp
builder.Services.Configure<ForwardedHeadersOptions>(o =>
{
    o.ForwardedHeaders = ForwardedHeaders.XForwardedFor | ForwardedHeaders.XForwardedProto;
    o.KnownNetworks.Clear();   // lab এর জন্য; production এ নির্দিষ্ট network দিন
    o.KnownProxies.Clear();
});
app.UseForwardedHeaders();
```

এটা না করলে `Request.IsHttps` ভুল আসবে এবং redirect loop হতে পারে।

## 5.6 একাধিক Environment

| Environment | Namespace | Host | Issuer |
|---|---|---|---|
| dev | `scaling-demo-dev` | `dev.bmi.example.com` | self-signed / staging |
| staging | `scaling-demo-stg` | `stg.bmi.example.com` | Let's Encrypt staging |
| prod | `scaling-demo` | `bmi.example.com` | Let's Encrypt prod |

একটা Controller সব namespace-এর Ingress পড়ে, তাই একটা IP-তে সব environment চালানো যায়।

## 5.7 Monitoring

Controller-এ `metrics.enabled=true` দিলে Prometheus metric পাবেন (Chapter 4-এর Prometheus + Grafana lab-এর সাথে জোড়া যায়)। গুরুত্বপূর্ণ metric:

- `nginx_ingress_controller_requests` — status code অনুযায়ী request সংখ্যা
- `nginx_ingress_controller_request_duration_seconds` — latency
- `nginx_ingress_controller_ssl_expire_time_seconds` — certificate কবে শেষ হবে (এটার alert রাখুন!)

## 5.8 সারসংক্ষেপ

- Controller: ২+ replica, resources, PDB, metrics
- Ingress: HTTPS, redirect, timeout, rate limit, header
- Backend: probe, forwarded headers
- সব কিছু YAML + Git-এ

পরের chapter: [Chapter 6 — TLS, Server Certificate, Client Certificate (তত্ত্ব)](ch06-tls-theory.md)
