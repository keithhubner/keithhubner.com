---
title: "🚀 Deploying Plausible to Civo Kubernetes 🌍"
date: 2025-01-22
draft: false
tags: ["Civo", "Kubernetes"]
---

🎯 **Welcome to the Ultimate Guide on Deploying Plausible to Civo Kubernetes!**

In this guide, we'll walk through deploying **Plausible Analytics**—a lightweight, open-source, privacy-friendly web analytics tool—on a **Civo Kubernetes** cluster. By the end, you'll have a fully functional Plausible instance running securely with **🔒 Let's Encrypt SSL**, **🐘 PostgreSQL**, and **📊 ClickHouse**.

---

## 📌 Prerequisites 🛠️

Before we begin, ensure you have the following set up:

✅ **Civo Account**: [Sign up for Civo](https://www.civo.com/signup) and ensure you have Kubernetes access.  
✅ **Civo CLI**: [Install the Civo CLI](https://www.civo.com/docs/cli) and configure it.  
✅ **kubectl**: [Install Kubernetes CLI](https://kubernetes.io/docs/tasks/tools/) for cluster management.  
✅ **Helm**: [Install Helm](https://helm.sh/docs/intro/install/) to deploy packages on Kubernetes.  
✅ **🌐 Domain Name**: A domain for hosting Plausible.  
✅ **✉️ Email Address**: Required for Let's Encrypt SSL certificates.  

---

## 🚀 Step 1: Create Your Civo Kubernetes Cluster 🌟

To create a cluster, run the following command:

```bash
civo k8s create plausible-cluster --nodes=3 --size=g4s.kube.medium --applications "cert-manager, metrics-server,traefik2-nodeport" --wait --save --merge
```

🔹 **Explanation:**
- `--nodes=3`: Creates a 3-node cluster.
- `--size=g4s.kube.medium`: Specifies node size.
- `--wait --save --merge`: Waits for deployment, saves config, and merges it with `kubectl`.

Once the cluster is ready, verify:
```bash
kubectl get nodes
```

---

## 🏗️ Step 2: Set Up a Namespace 🎯

Namespaces help organize resources. Let's create one for **Plausible**:

```bash
kubectl create namespace plausible
```

Verify:
```bash
kubectl get namespaces
```

---

## 🔒 Step 3: Set Up Secrets and ConfigMap

### 🔐 Securely Store Database Credentials

Create a `plausible-secrets.yaml` file with the contents below, replacing `secure-random-password` with a secure, random password:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: plausible-config
  namespace: plausible
type: Opaque
stringData:
  POSTGRES_USER: plausible
  POSTGRES_PASSWORD: secure-random-password
  POSTGRES_DB: plausible_db
```

Apply with:

```bash
kubectl apply -f plausible-secrets.yaml
```

### 📝 Configure Plausible Settings

Create a `plausible-configmap.yaml` file with the contents below, replacing `YOUR_DOMAIN` with your domain name, and `GENERATED_KEY` with a random string (you can use the command `openssl rand -base64 48` to generate one):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plausible-configmap
  namespace: plausible
data:
  BASE_URL: "https://YOUR_DOMAIN"
  SECRET_KEY_BASE: "GENERATED_KEY"
  CLICKHOUSE_DATABASE_URL: "http://plausible-clickhouse:8123/plausible_events_db"
```

Apply with:

```bash
kubectl apply -f plausible-configmap.yaml
```

---

## 🏅 Step 4: Configure Let's Encrypt Issuers

Create an `issuer.yaml` file, replacing `your-email@example.com` with a legitimate email address.

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: your-email@example.com
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-staging-key
    solvers:
    - http01:
        ingress:
          class: traefik
---
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
    - http01:
        ingress:
          class: traefik
```

Apply with:

```bash
kubectl apply -f issuer.yaml
```

---

## 🌍 Step 5: Deploy Ingress for External Access

Create `plausible-ingress.yaml` with the contents below, replacing `YOUR_DOMAIN` with your actual domain name.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: plausible-ingress
  namespace: plausible
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  rules:
  - host: YOUR_DOMAIN
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: plausible
            port:
              number: 80
  tls:
  - hosts:
    - YOUR_DOMAIN
    secretName: plausible-tls
```

Apply with:

```bash
kubectl apply -f plausible-ingress.yaml
```

🔹 **Create the DNS record for the ingress above. After confirming DNS setup, switch to production issuer:**

```bash
kubectl patch ingress plausible-ingress -n plausible --type='json' -p='[{"op": "replace", "path": "/metadata/annotations/cert-manager.io~1cluster-issuer", "value": "letsencrypt-production"}]'
```

---

## 🛠️ Step 6: Deploy PostgreSQL, ClickHouse, and Plausible

### 🐘 Deploy PostgreSQL

Create `postgres-deployment.yaml` with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible-postgresql
  namespace: plausible
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: postgresql
        image: postgres:14
        envFrom:
        - secretRef:
            name: plausible-config
```

Apply with:

```bash
kubectl apply -f postgres-deployment.yaml
```

### 📊 Deploy ClickHouse

Create `clickhouse-deployment.yaml` with:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible-clickhouse
  namespace: plausible
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:24.3.3.102-alpine
```

Apply with:

```bash
kubectl apply -f clickhouse-deployment.yaml
```

### 📈 Deploy Plausible

Create `plausible-deployment.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible
  namespace: plausible
spec:
  replicas: 1
  template:
    spec:
      containers:
      - name: plausible
        image: ghcr.io/plausible/community-edition:v2.1.4
        envFrom:
        - secretRef:
            name: plausible-config
```

Apply with:

```bash
kubectl apply -f plausible-deployment.yaml
```

🎉 **Done!** Your Plausible Analytics instance is now live on Civo Kubernetes! 🚀