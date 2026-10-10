# Chapter 2: Ingress Controller Install করা

> Ingress YAML লিখার আগে **Controller** লাগবে (Chapter 1 দেখুন)। এই chapter-এ আমরা NGINX Ingress Controller install করব।

## 2.1 আগে যা লাগবে

- একটা চলমান Kubernetes cluster (Docker Desktop / minikube / kind — যেটা আপনার আছে)
- `kubectl` এবং `helm` install করা
- যাচাই:

```bash
kubectl get nodes
helm version
```

## 2.2 একটা গুরুত্বপূর্ণ খবর (২০২৬)

Kubernetes community-র `ingress-nginx` প্রজেক্ট ২০২৬-এর মার্চ থেকে retire হওয়ার ঘোষণা আছে, অর্থাৎ আর নতুন feature/security patch আসবে না। তাই:

- **শেখার জন্য / lab-এ** — `ingress-nginx` এখনও ঠিক আছে, আর Ingress-এর concept সব controller-এ একই।
- **নতুন production-এ** — Traefik, HAProxy Ingress, F5 NGINX Ingress Controller অথবা **Gateway API** (Ingress-এর উত্তরসূরি) বিবেচনা করুন।

> সঠিক বর্তমান অবস্থা আপনার নিজে একবার যাচাই করে নিন (kubernetes.io blog / GitHub repo)। এই doc-এর বাকি অংশে `ingressClassName: nginx` ব্যবহার করা হয়েছে; অন্য controller নিলে শুধু class name আর annotation বদলাবে।

## 2.3 পদ্ধতি A: Helm দিয়ে (সব cluster-এ কাজ করে, সবচেয়ে ভালো)

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx --create-namespace
```

Install হয়েছে কিনা দেখুন:

```bash
kubectl get pods -n ingress-nginx
kubectl get svc  -n ingress-nginx
kubectl get ingressclass
```

প্রত্যাশিত ফল:

- `ingress-nginx-controller-xxxxx` Pod → `Running`
- `ingress-nginx-controller` Service → type `LoadBalancer`
- `IngressClass` → নাম `nginx`

## 2.4 পদ্ধতি B: আপনার cluster অনুযায়ী

### Docker Desktop Kubernetes

Helm দিয়ে install করলে Service-এর `EXTERNAL-IP` হবে `localhost`। অর্থাৎ browser-এ সরাসরি `http://localhost` কাজ করবে।

### minikube

```bash
minikube addons enable ingress
kubectl get pods -n ingress-nginx
```

তারপর আলাদা terminal-এ `minikube tunnel` চালু রাখুন (Windows/macOS-এ Docker driver হলে দরকার)।

### kind

kind-এ cluster বানানোর সময়ই port mapping দিতে হয়:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
      - containerPort: 443
        hostPort: 443
```

```bash
kind create cluster --config kind-config.yaml
```

তারপর kind-এর জন্য ingress-nginx-এর manifest apply করুন (ingress-nginx docs-এর "kind" অংশে দেওয়া আছে)।

## 2.5 Controller কাজ করছে কিনা যাচাই

```bash
curl -i http://localhost
```

`404 Not Found` (NGINX-এর) আসলে বুঝবেন **Controller ঠিকঠাক চলছে**, শুধু এখনও কোনো Ingress নিয়ম নেই। এটাই সঠিক ফল।

## 2.6 সারসংক্ষেপ

| ধাপ | কমান্ড |
|---|---|
| Controller install | `helm upgrade --install ingress-nginx ...` |
| Pod দেখা | `kubectl get pods -n ingress-nginx` |
| IngressClass দেখা | `kubectl get ingressclass` |
| পরীক্ষা | `curl -i http://localhost` → NGINX 404 |

পরের chapter: [Chapter 3 — প্রথম Ingress (BMI app)](ch03-first-ingress.md)
