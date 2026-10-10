# Chapter 9: Client Certificate ও mTLS (Ingress-এ)

> লক্ষ্য: শুধু যাদের কাছে আমাদের CA-র সই করা **client certificate** আছে, তারাই `https://bmi.local` এ ঢুকতে পারবে।

## 9.1 কখন লাগে

- Admin panel বা internal API, যেখানে শুধু নির্দিষ্ট মানুষ/সার্ভিস ঢুকবে
- Service-to-service communication (পাসওয়ার্ড/token ছাড়া)
- Partner/ব্যাংক integration

> সাধারণ public website-এ mTLS লাগে না। সাধারণ user-দের browser-এ certificate বসানো কঠিন।

## 9.2 ধাপ ১: Client certificate বানানো

Chapter 7-এর `certs/` ফোল্ডারে থাকুন (`ca.key`, `ca.crt` থাকতে হবে)।

```bash
# client key
openssl genrsa -out client.key 2048

# CSR (CN = ব্যবহারকারীর নাম)
MSYS_NO_PATHCONV=1 openssl req -new -key client.key \
  -subj "/CN=abdullah/O=BMI-Admins" -out client.csr
```

`client.ext`:

```
basicConstraints=CA:FALSE
keyUsage=digitalSignature
extendedKeyUsage=clientAuth
```

> Server certificate-এ ছিল `serverAuth`, এখানে `clientAuth` — এটাই মূল পার্থক্য।

সই করা:

```bash
openssl x509 -req -in client.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out client.crt -days 365 -sha256 -extfile client.ext

openssl verify -CAfile ca.crt client.crt       # OK আসবে
```

## 9.3 ধাপ ২: CA-র certificate কে Secret-এ রাখা

Ingress-কে জানাতে হবে "কোন CA-র সই করা client কে মানব"। এখানে শুধু **`ca.crt`** (প্রকাশ্য অংশ) দিন, `ca.key` নয়।

```bash
kubectl create secret generic client-ca \
  --from-file=ca.crt=ca.crt \
  -n scaling-demo
```

> Secret-এর ভেতরের key-এর নাম অবশ্যই `ca.crt` হতে হবে।

## 9.4 ধাপ ৩: Ingress-এ mTLS annotation

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: scaling-demo-admin
  namespace: scaling-demo
  annotations:
    nginx.ingress.kubernetes.io/auth-tls-verify-client: "on"
    nginx.ingress.kubernetes.io/auth-tls-secret: "scaling-demo/client-ca"   # <namespace>/<secret>
    nginx.ingress.kubernetes.io/auth-tls-verify-depth: "1"
    nginx.ingress.kubernetes.io/auth-tls-error-page: "https://bmi.local/cert-required"
    nginx.ingress.kubernetes.io/auth-tls-pass-certificate-to-upstream: "true"
spec:
  ingressClassName: nginx
  tls:
    - hosts: [bmi.local]
      secretName: bmi-local-tls
  rules:
    - host: bmi.local
      http:
        paths:
          - path: /admin
            pathType: Prefix
            backend:
              service: { name: scaling-demo-api, port: { number: 80 } }
```

annotation-এর অর্থ:

| Annotation | কাজ |
|---|---|
| `auth-tls-verify-client` | `on` = client certificate বাধ্যতামূলক; `optional` = থাকলে যাচাই, না থাকলে ঢুকতে দেবে; `off` = বন্ধ |
| `auth-tls-secret` | কোন CA দিয়ে যাচাই (`namespace/secret`) |
| `auth-tls-verify-depth` | chain কত গভীর পর্যন্ত যাচাই (আমাদের ১টি CA, তাই 1) |
| `auth-tls-pass-certificate-to-upstream` | Client certificate API-কে `ssl-client-cert` header-এ পাঠায় |

> `auth-tls-error-page` ঐচ্ছিক। না দিলে NGINX-এর ডিফল্ট ৪০০ error আসে।

**গুরুত্বপূর্ণ সীমাবদ্ধতা:** client certificate যাচাই NGINX-এ **host (server block) পর্যায়ে** হয়, path পর্যায়ে নয়। অর্থাৎ একই host-এর কোনো একটা Ingress-এ `verify-client: on` দিলে সেই host-এর সবকিছুতেই আসলে handshake-এ certificate চাইবে। তাই practical উপায় হলো **admin-এর জন্য আলাদা host** ব্যবহার করা (যেমন `admin.bmi.local`) এবং `verify-client: optional` ব্যবহার করে path অনুযায়ী যাচাই করা। নিচে দেখুন।

## 9.5 ভালো নকশা: আলাদা host

```
https://bmi.local         → সবার জন্য, certificate লাগে না
https://admin.bmi.local   → mTLS বাধ্যতামূলক
```

- `hosts` ফাইলে `admin.bmi.local` যোগ করুন
- Server certificate-এ SAN-এ `*.bmi.local` আছে (Chapter 7), তাই একই certificate চলবে
- `admin.bmi.local` এর Ingress-এ ওপরের annotation গুলো দিন, `host: admin.bmi.local` করুন, `tls.hosts` এও এটা রাখুন

## 9.6 ধাপ ৪: পরীক্ষা

**Certificate ছাড়া** (ব্যর্থ হওয়া উচিত):

```bash
curl --cacert ca.crt https://admin.bmi.local/
# 400 Bad Request — No required SSL certificate was sent
```

**Certificate সহ** (সফল):

```bash
curl --cacert ca.crt --cert client.crt --key client.key https://admin.bmi.local/
```

`hosts` ফাইল না বদলে:

```bash
curl --resolve admin.bmi.local:443:127.0.0.1 \
  --cacert ca.crt --cert client.crt --key client.key \
  https://admin.bmi.local/
```

**ভুল CA-র certificate দিয়ে** (অন্য কোনো self-signed) ব্যর্থ হবে — `400 The SSL certificate error`।

## 9.7 Browser-এ client certificate ব্যবহার

Browser `.pfx` চায়:

```bash
openssl pkcs12 -export -out client.pfx \
  -inkey client.key -in client.crt -certfile ca.crt
# পাসওয়ার্ড চাইবে — একটা দিন
```

**Windows:** `client.pfx` ডাবল-ক্লিক → *Current User* → পাসওয়ার্ড দিন → Personal store-এ বসান। Browser restart করুন; `https://admin.bmi.local` খুললে certificate বাছার dialog আসবে।

## 9.8 API-তে client-এর পরিচয় পড়া

`pass-certificate-to-upstream: "true"` দিলে API পায় header `ssl-client-cert` (URL-encoded PEM)। ASP.NET Core-এ:

```csharp
app.MapGet("/admin/whoami", (HttpContext ctx) =>
{
    var raw = ctx.Request.Headers["ssl-client-cert"].ToString();
    if (string.IsNullOrEmpty(raw)) return Results.Unauthorized();

    var pem = Uri.UnescapeDataString(raw);
    var cert = System.Security.Cryptography.X509Certificates
                 .X509Certificate2.CreateFromPem(pem);
    return Results.Ok(new { cert.Subject, cert.Thumbprint, cert.NotAfter });
});
```

> **নিরাপত্তা:** এই header শুধু Ingress থেকে এলেই বিশ্বাস করুন। API সরাসরি বাইরে খোলা থাকলে কেউ নকল header পাঠাতে পারবে। তাই API Service `ClusterIP` রাখুন।

## 9.9 Certificate বাতিল (Revocation)

Client certificate চুরি/হারিয়ে গেলে কী করবেন?

- ছোট lab: নতুন CA বানিয়ে সবাইকে নতুন certificate দিন, `client-ca` Secret বদলান
- Production: ছোট মেয়াদ (৩০–৯০ দিন), CRL বা `auth-tls-crl` ব্যবহার (বিস্তারিত controller docs-এ), অথবা cert-manager দিয়ে স্বয়ংক্রিয় renew

## 9.10 Service-to-service mTLS (Ingress-এর বাইরে)

Pod থেকে Pod-এর মধ্যে mTLS দরকার হলে সাধারণত **service mesh** (Istio, Linkerd) নেওয়া হয়, যারা sidecar দিয়ে স্বয়ংক্রিয়ভাবে certificate দেয় ও renew করে। এটা আলাদা বিষয়; পরবর্তী chapter-এ আলোচনা করা যেতে পারে।

## 9.11 সারসংক্ষেপ

```
client.key ──► client.csr ──(ca দিয়ে সই)──► client.crt  (extendedKeyUsage=clientAuth)
ca.crt ──► Secret "client-ca" ──► Ingress annotation auth-tls-secret
Client: curl --cert client.crt --key client.key
```

পরের chapter: [Chapter 10 — Troubleshooting ও Cheat Sheet](ch10-troubleshooting.md)
