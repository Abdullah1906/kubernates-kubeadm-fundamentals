# Chapter 7: Ingress-এ HTTPS চালু করা (নিজস্ব CA + openssl)

> লক্ষ্য: `https://bmi.local` চালু করা। আমরা নিজেরা একটা CA বানাব, তারপর তা দিয়ে server certificate সই করব। এই CA-ই Chapter 9-এ client certificate-এও কাজে লাগবে।

## 7.1 সরঞ্জাম

`openssl` লাগবে। Windows-এ **Git Bash** (Git for Windows-এর সাথে আসে) বা WSL ব্যবহার করুন।

> **Git Bash-এর সতর্কতা:** `-subj "/CN=..."` কে Git Bash ভুল করে Windows path বানিয়ে ফেলে। কমান্ডের আগে `MSYS_NO_PATHCONV=1` বসান, অথবা `-subj "//CN=..."` লিখুন (WSL/Linux-এ দরকার নেই)।

```bash
mkdir -p certs && cd certs
```

> `certs/` ফোল্ডার `.gitignore`-এ রাখুন। **কোনো `.key` ফাইল GitHub-এ push করবেন না।**

## 7.2 ধাপ ১: নিজস্ব CA বানানো

```bash
# CA-র private key
openssl genrsa -out ca.key 4096

# CA-র certificate (১০ বছর)
MSYS_NO_PATHCONV=1 openssl req -x509 -new -nodes -key ca.key -sha256 \
  -days 3650 -subj "/CN=BMI Lab CA" -out ca.crt
```

ফলাফল: `ca.key` (গোপন), `ca.crt` (প্রকাশ্য)।

## 7.3 ধাপ ২: Server key + CSR

```bash
openssl genrsa -out server.key 2048

MSYS_NO_PATHCONV=1 openssl req -new -key server.key \
  -subj "/CN=bmi.local" -out server.csr
```

## 7.4 ধাপ ৩: CA দিয়ে server certificate সই (SAN সহ)

`server.ext` ফাইল বানান:

```
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt_names

[alt_names]
DNS.1=bmi.local
DNS.2=*.bmi.local
```

এবার সই করুন:

```bash
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 365 -sha256 -extfile server.ext
```

যাচাই:

```bash
openssl x509 -in server.crt -noout -text | grep -A1 "Alternative Name"
openssl verify -CAfile ca.crt server.crt     # server.crt: OK আসবে
```

## 7.5 ধাপ ৪: Kubernetes Secret বানানো

```bash
kubectl create secret tls bmi-local-tls \
  --cert=server.crt --key=server.key \
  -n scaling-demo
```

> Secret অবশ্যই **Ingress-এর একই namespace**-এ থাকতে হবে।

Secret দেখা:

```bash
kubectl get secret bmi-local-tls -n scaling-demo
```

(`kubernetes.io/tls` টাইপ হওয়া উচিত।)

## 7.6 ধাপ ৫: Ingress-এ `tls` অংশ যোগ

Chapter 3-এর Ingress-এ শুধু `tls:` ব্লকটা যোগ করুন:

```yaml
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - bmi.local
      secretName: bmi-local-tls
  rules:
    - host: bmi.local
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

```bash
kubectl apply -f k8s/ingress/30-ingress.yaml
```

## 7.7 ধাপ ৬: পরীক্ষা

```bash
# আমাদের CA বিশ্বাস করিয়ে — সঠিক পদ্ধতি
curl --cacert certs/ca.crt https://bmi.local/

# শুধু দ্রুত দেখতে (certificate যাচাই বাদ) — অভ্যাস করবেন না
curl -k https://bmi.local/

# HTTP → HTTPS redirect দেখা
curl -i http://bmi.local/        # 308 Permanent Redirect আসবে
```

কোন certificate দেওয়া হচ্ছে দেখতে:

```bash
openssl s_client -connect localhost:443 -servername bmi.local </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates
```

`-servername` লাগে কারণ Ingress **SNI** (কোন domain চাওয়া হচ্ছে) দেখে certificate বাছে।

## 7.8 Browser-এ warning না চাইলে

আমাদের CA ওই কম্পিউটার চেনে না, তাই warning আসে। `ca.crt` কে trust করান:

**Windows:** `ca.crt` ডাবল-ক্লিক → *Install Certificate* → *Local Machine* → *Place all certificates in the following store* → **Trusted Root Certification Authorities** → Finish। Browser restart করুন।

**Linux (Ubuntu):**

```bash
sudo cp certs/ca.crt /usr/local/share/ca-certificates/bmi-lab-ca.crt
sudo update-ca-certificates
```

> শুধু নিজের lab মেশিনেই এটা করুন। অন্যের/অফিসের মেশিনে নিজস্ব Root CA বসানো ঝুঁকিপূর্ণ।

## 7.9 সাধারণ ভুল

| সমস্যা | কারণ |
|---|---|
| Browser এ `Kubernetes Ingress Controller Fake Certificate` | Secret নাম ভুল/ভুল namespace, বা `tls.hosts` মেলেনি |
| `NET::ERR_CERT_COMMON_NAME_INVALID` | SAN এ domain নেই |
| `unable to get local issuer certificate` | `--cacert` দেননি / CA trust করাননি |
| redirect loop | App নিজেও HTTPS-এ redirect করছে অথচ forwarded header পড়ছে না (Chapter 5.5) |

## 7.10 সারসংক্ষেপ

```
ca.key + ca.crt  ──signs──►  server.crt (SAN: bmi.local)
server.crt + server.key  ──►  Secret (tls)  ──►  Ingress spec.tls
```

পরের chapter: [Chapter 8 — cert-manager ও Let's Encrypt (production)](ch08-cert-manager.md)
