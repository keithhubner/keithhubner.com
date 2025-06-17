---
title: "Deploying Supabase to Civo Kubernetes"
date: 2025-01-22
draft: true
tags: ["Civo", "Kubernetes", "Supabase"]
---



## Introduction

Supabase is an open-source alternative to Firebase, offering authentication, database, and storage capabilities. In this guide, we will deploy Supabase to a **Civo Kubernetes** cluster, leveraging Civo’s lightweight and cost-effective managed Kubernetes service.

## Prerequisites

Before we begin, ensure you have the following:

- A **Civo account** with Kubernetes enabled ([Sign up here](https://www.civo.com/))
- `kubectl` installed and configured for your Civo cluster
- The Civo CLI installed (`brew install civo` for macOS users)
- Helm installed (`brew install helm`)
- Docker installed
- Supabase CLI installed (`npm install -g supabase`)

## Step 1: Create a Kubernetes Cluster on Civo

### Using the Civo CLI

```sh
civo k3s create supabase-cluster --size g3.k3s.medium --nodes=3 --region LON1
```

### Retrieve Kubeconfig

```sh
civo k3s config supabase-cluster --save --switch
```

Verify the cluster is active:

```sh
kubectl get nodes
```

## Step 2: Install Supabase on Kubernetes

### Clone Supabase Kubernetes Helm Chart

```sh
git clone https://github.com/supabase-community/supabase-kubernetes.git
cd supabase-kubernetes
```

### Install Helm Chart

```sh
helm install supabase . --namespace supabase --create-namespace
```

Verify deployment:

```sh
kubectl get pods -n supabase
```

## Step 3: Exposing Supabase Services

To expose Supabase, create a Kubernetes Ingress. First, install Nginx Ingress Controller:

```sh
helm upgrade --install nginx-ingress ingress-nginx/ingress-nginx --namespace ingress --create-namespace
```

Then, apply the following Ingress configuration:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: supabase-ingress
  namespace: supabase
spec:
  rules:
  - host: supabase.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: supabase
            port:
              number: 80
```

Apply the Ingress:

```sh
kubectl apply -f supabase-ingress.yaml
```

## Step 4: Configuring DNS

Update your domain’s DNS settings to point to your Civo Kubernetes LoadBalancer IP:

```sh
kubectl get svc -n ingress
```

Set an `A` record for `supabase.yourdomain.com` pointing to the **EXTERNAL-IP**.

## Step 5: Verifying the Deployment

Once DNS propagates, access Supabase at `https://supabase.yourdomain.com`. 

Check logs for troubleshooting:

```sh
kubectl logs -n supabase -l app=supabase
```

## Conclusion

You have successfully deployed Supabase on **Civo Kubernetes**! This setup provides a scalable, self-hosted backend solution for your applications.

---

Do you have questions or improvements? Drop them in the comments below!
