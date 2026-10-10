# Ingress ও TLS — Chapter সূচি

Kubernetes-এ Ingress, HTTPS, Server Certificate ও Client Certificate (mTLS) — সহজ বাংলায়, `scaling-demo` (BMI app) উদাহরণ সহ।

| Chapter | বিষয় |
|---|---|
| [01](ch01-ingress-ki-keno.md) | Ingress কী এবং Service থাকতে কেন লাগে |
| [02](ch02-controller-install.md) | Ingress Controller install |
| [03](ch03-first-ingress.md) | প্রথম Ingress — BMI app দিয়ে |
| [04](ch04-routing-rules.md) | Host, Path, Rewrite, Annotation |
| [05](ch05-production-practices.md) | Professional grade ব্যবহার |
| [06](ch06-tls-theory.md) | TLS, Server Certificate, Client Certificate (তত্ত্ব) |
| [07](ch07-https-on-ingress.md) | Ingress-এ HTTPS (নিজস্ব CA + openssl) |
| [08](ch08-cert-manager.md) | cert-manager ও Let's Encrypt |
| [09](ch09-client-certificate-mtls.md) | Client Certificate ও mTLS |
| [10](ch10-troubleshooting.md) | Troubleshooting ও Cheat Sheet |

## সতর্কতা

`certs/` ফোল্ডারের কোনো `.key` ফাইল GitHub-এ commit করবেন না। `.gitignore`-এ যোগ করুন:

```
certs/
*.key
*.pfx
```
