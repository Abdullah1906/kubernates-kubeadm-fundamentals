# Chapter 6: TLS, Server Certificate ও Client Certificate — সহজ বাংলায়

> আগে তত্ত্বটা বুঝে নিই, তারপর পরের chapter-গুলোতে হাতে-কলমে করব।

## 6.1 HTTP vs HTTPS

- **HTTP** — ডেটা সাধারণ লেখার মতো যায়। রাস্তায় যে কেউ পড়তে বা বদলাতে পারে।
- **HTTPS** = HTTP + **TLS** (পুরনো নাম SSL)। ডেটা encrypted হয়ে যায়।

TLS তিনটা জিনিস দেয়:

1. **গোপনীয়তা (Encryption)** — মাঝপথে কেউ পড়তে পারবে না
2. **অখণ্ডতা (Integrity)** — মাঝপথে কেউ বদলাতে পারবে না
3. **পরিচয় যাচাই (Authentication)** — আপনি সত্যিই `bmi.example.com` এর সাথেই কথা বলছেন

তৃতীয়টার জন্যই **certificate** লাগে।

## 6.2 Certificate কী?

Certificate হলো একটা **ডিজিটাল পরিচয়পত্র**। এতে থাকে:

- কার পরিচয় (domain নাম — `bmi.example.com`)
- তার **public key**
- কে এটা যাচাই করে দিয়েছে (**CA**)
- মেয়াদ (কবে থেকে কবে)
- CA-র **digital signature**

জাতীয় পরিচয়পত্রের সাথে মিলিয়ে দেখুন: আপনার NID-তে আপনার তথ্য আছে এবং সরকারের সিলমোহর আছে। সবাই সরকারকে বিশ্বাস করে বলে NID-ও বিশ্বাস করে। Certificate-ও তেমন: CA-কে বিশ্বাস করলে CA-র সই করা certificate-ও বিশ্বাস করা হয়।

## 6.3 Public key ও Private key

| | Public key | Private key |
|---|---|---|
| কে জানে | সবাই | **শুধু মালিক** |
| কাজ | encrypt / signature যাচাই | decrypt / signature তৈরি |
| ফাইল | `.crt` / `.pem` (certificate-এর ভেতরে) | `.key` |

> **Private key কখনো কাউকে দেবেন না, Git-এ commit করবেন না।** Certificate (`.crt`) প্রকাশ্য, কিন্তু `.key` গোপন।

## 6.4 CA (Certificate Authority)

CA হলো বিশ্বস্ত প্রতিষ্ঠান যে certificate-এ সই করে।

- **Public CA** — Let's Encrypt, DigiCert, Sectigo। Browser/OS আগে থেকেই এদের বিশ্বাস করে।
- **Private (নিজস্ব) CA** — আপনি নিজে বানান (এই lab-এ যেমন `BMI Lab CA`)। শুধু যাদের কাছে আপনার CA-র certificate (`ca.crt`) আছে তারাই বিশ্বাস করবে।
- **Self-signed certificate** — নিজেই নিজের সই। Browser warning দেয়। শুধু dev/test-এ।

### Certificate chain

```
Root CA  ──signs──►  Intermediate CA  ──signs──►  Server certificate (bmi.example.com)
```

Client সার্ভারের certificate থেকে উপরে Root পর্যন্ত chain মিলিয়ে দেখে। Root তার trust store-এ থাকলেই বিশ্বাস করে।

## 6.5 Server Certificate

**সার্ভার নিজের পরিচয় প্রমাণ করে** client-এর কাছে।

- সাধারণ HTTPS-এ শুধু এটাই লাগে
- Browser যাচাই করে: (১) CA বিশ্বস্ত? (২) domain মিলছে? (৩) মেয়াদ আছে? 
- **SAN (Subject Alternative Name)** এ domain নাম থাকতে হবে। আধুনিক browser শুধু `CN` দেখে না।

## 6.6 Client Certificate

**Client নিজের পরিচয় প্রমাণ করে** সার্ভারের কাছে।

- পাসওয়ার্ডের বদলে (বা সাথে) certificate দিয়ে লগইন
- ব্যবহার: ব্যাংকের API, microservice-to-microservice, অফিসের VPN/internal tool, IoT device
- সার্ভার যাচাই করে: client-এর certificate কি আমার বিশ্বস্ত CA সই করেছে?

| | Server certificate | Client certificate |
|---|---|---|
| কে দেখায় | সার্ভার | Client (browser/app/curl) |
| কে যাচাই করে | Client | সার্ভার |
| কখন লাগে | সবসময় (HTTPS) | শুধু mTLS-এ |
| Extended Key Usage | `serverAuth` | `clientAuth` |

## 6.7 One-way TLS বনাম mTLS

```
One-way TLS (সাধারণ HTTPS)
  Client ── "তুমি কে?" ──► Server
  Client ◄── server certificate ── Server
  (শুধু client সার্ভারকে যাচাই করে)

Mutual TLS (mTLS)
  Client ── "তুমি কে?" ──► Server
  Client ◄── server certificate ── Server
  Client ◄── "তুমিও certificate দেখাও" ── Server
  Client ── client certificate ──► Server
  (দুজনেই দুজনকে যাচাই করে)
```

## 6.8 TLS Handshake (সরল ধাপ)

1. Client বলে: "আমি TLS চাই, আমার সমর্থিত cipher এগুলো।"
2. Server উত্তর দেয় + **server certificate** পাঠায়।
3. Client certificate যাচাই করে (CA, domain, মেয়াদ)।
4. (mTLS হলে) Server **client certificate চায়**, client পাঠায়, server যাচাই করে।
5. দুজনে মিলে একটা **session key** বানায়।
6. এরপর সব ডেটা ওই session key দিয়ে encrypt হয়ে যায় (দ্রুত symmetric encryption)।

## 6.9 ফাইলের ধরন

| Extension | কী |
|---|---|
| `.key` | Private key |
| `.csr` | Certificate Signing Request — CA-র কাছে "আমার certificate-এ সই করো" আবেদন |
| `.crt` / `.cer` / `.pem` | Certificate (প্রকাশ্য) |
| `.pfx` / `.p12` | Certificate + private key একসাথে, পাসওয়ার্ড দিয়ে সুরক্ষিত (Windows/browser-এ import করতে) |

## 6.10 Kubernetes-এ TLS কোথায় শেষ হয় (Termination)

| Mode | কোথায় decrypt | মন্তব্য |
|---|---|---|
| **TLS Termination** (সবচেয়ে প্রচলিত) | Ingress-এ | Ingress → Pod পর্যন্ত সাধারণ HTTP। সহজ, certificate এক জায়গায় |
| **Re-encryption** | Ingress-এ, আবার নতুন করে encrypt | Ingress → Pod-ও HTTPS। বেশি নিরাপদ |
| **Passthrough** | Pod-এ | Ingress শুধু TCP পাঠায়, নিজে দেখে না; path routing সম্ভব না |

আমরা **TLS Termination** দিয়ে শুরু করব।

## 6.11 সারসংক্ষেপ

- Certificate = CA-সই করা ডিজিটাল পরিচয়পত্র
- Server certificate: সার্ভারের পরিচয় (সবসময় লাগে)
- Client certificate: client-এর পরিচয় (শুধু mTLS-এ)
- Private key গোপন রাখুন
- Domain নাম **SAN**-এ রাখুন

পরের chapter: [Chapter 7 — Ingress-এ HTTPS (নিজস্ব CA + openssl)](ch07-https-on-ingress.md)
