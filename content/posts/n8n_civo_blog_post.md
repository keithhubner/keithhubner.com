---
title: "Deploy n8n to Civo Kubernetes"
date: 2025-06-17T10:00:00Z
draft: false
tags: ["kubernetes", "n8n", "civo", "automation", "devops", "cloud"]
categories: ["tutorials", "kubernetes"]
author: "Keith Hubner"
description: "Learn how to deploy n8n workflow automation to a Civo Kubernetes cluster with this comprehensive step-by-step guide. Includes setup for new and existing clusters."
summary: "A complete guide to deploying n8n on Civo Kubernetes, covering everything from cluster setup to production considerations with hands-on examples."
---

---

n8n is a powerful workflow automation tool that helps you connect different services and automate repetitive tasks. When combined with Kubernetes on Civo's developer-friendly cloud platform, you get a scalable, reliable automation solution that can grow with your needs.

In this guide, we'll walk through deploying n8n to a Civo Kubernetes cluster, whether you're starting fresh or working with an existing cluster.

## What You'll Need

Before we begin, make sure you have:

- A Civo account (sign up at [civo.com](https://civo.com) if you don't have one, you should!)
- `kubectl` installed on your local machine
- `civo` CLI tool installed
- Basic familiarity with Kubernetes concepts

## Step 1: Set Up Your Civo Kubernetes Cluster

### For New Clusters

First, let's create a new Kubernetes cluster on Civo. We'll use a medium-sized cluster that's perfect for running n8n:

```bash
# Install Civo CLI if you haven't already
curl -sL https://civo.com/get | sh

# Login to your Civo account
civo apikey save my-key YOUR_API_KEY_HERE

# Create a new Kubernetes cluster
civo kubernetes create n8n-cluster \
  --size g4s.kube.medium \
  --nodes 2 \
  --wait
```

This creates a 2-node cluster with sufficient resources for n8n and room to grow.

### For Existing Clusters

If you already have a Civo Kubernetes cluster, simply get your kubeconfig:

```bash
# List your clusters
civo kubernetes list

# Download the kubeconfig for your cluster
civo kubernetes config n8n-cluster --save
```

Verify your connection:

```bash
kubectl get nodes
```

You should see your cluster nodes listed and ready.

## Step 2: Create a Dedicated Namespace

Let's create a dedicated namespace for n8n to keep things organized:

```bash
kubectl create namespace n8n

# Set this as your default namespace for convenience
kubectl config set-context --current --namespace=n8n
```

## Step 3: Set Up Persistent Storage

n8n needs persistent storage to save your workflows, credentials, and execution data. We'll create a PersistentVolumeClaim:

```yaml
# Create a file called n8n-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: n8n-data
  namespace: n8n
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: civo-volume
```

Apply the PVC:

```bash
kubectl apply -f n8n-pvc.yaml
```

## Step 4: Configure n8n Environment Variables

Create a ConfigMap for n8n configuration:

```yaml
# Create n8n-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: n8n-config
  namespace: n8n
data:
  N8N_HOST: "n8n.yourdomain.com" # Replace with your domain
  N8N_PORT: "5678"
  N8N_PROTOCOL: "https"
  WEBHOOK_URL: "https://n8n.yourdomain.com/"
  GENERIC_TIMEZONE: "UTC"
  N8N_LOG_LEVEL: "info"
```

For sensitive data like database credentials, create a Secret:

```yaml
# Create n8n-secrets.yaml
apiVersion: v1
kind: Secret
metadata:
  name: n8n-secrets
  namespace: n8n
type: Opaque
stringData:
  N8N_BASIC_AUTH_USER: "admin"
  N8N_BASIC_AUTH_PASSWORD: "your-secure-password-here"
  N8N_ENCRYPTION_KEY: "your-encryption-key-here" # Generate a random 32-character string
```

Apply both configurations:

```bash
kubectl apply -f n8n-config.yaml
kubectl apply -f n8n-secrets.yaml
```

## Step 5: Deploy n8n

Now let's create the main n8n deployment:

```yaml
# Create n8n-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: n8n
  namespace: n8n
  labels:
    app: n8n
spec:
  replicas: 1
  selector:
    matchLabels:
      app: n8n
  template:
    metadata:
      labels:
        app: n8n
    spec:
      # Fix permissions for the mounted volume
      securityContext:
        fsGroup: 1000
        runAsUser: 1000
        runAsGroup: 1000
      # Use an init container to fix permissions if needed
      initContainers:
        - name: volume-permissions
          image: busybox:1.35
          command:
            - sh
            - -c
            - |
              chown -R 1000:1000 /home/node/.n8n
              chmod -R 755 /home/node/.n8n
          volumeMounts:
            - name: n8n-data
              mountPath: /home/node/.n8n
          securityContext:
            runAsUser: 0
      containers:
        - name: n8n
          image: n8nio/n8n:latest
          ports:
            - containerPort: 5678
          envFrom:
            - configMapRef:
                name: n8n-config
            - secretRef:
                name: n8n-secrets
          volumeMounts:
            - name: n8n-data
              mountPath: /home/node/.n8n
          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "500m"
          # Give the init container time to fix permissions
          livenessProbe:
            httpGet:
              path: /healthz
              port: 5678
            initialDelaySeconds: 60
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: 5678
            initialDelaySeconds: 30
            periodSeconds: 5
          securityContext:
            runAsUser: 1000
            runAsGroup: 1000
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: false
      volumes:
        - name: n8n-data
          persistentVolumeClaim:
            claimName: n8n-data
```

Deploy n8n:

```bash
kubectl apply -f n8n-deployment.yaml
```

## Step 6: Expose n8n with a Service

Create a service to expose n8n within the cluster:

```yaml
# Create n8n-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: n8n-service
  namespace: n8n
spec:
  selector:
    app: n8n
  ports:
    - port: 80
      targetPort: 5678
      protocol: TCP
  type: ClusterIP
```

Apply the service:

```bash
kubectl apply -f n8n-service.yaml
```

## Step 7: Install and Configure cert-manager

Before setting up ingress, we need cert-manager to handle SSL certificates automatically. cert-manager will obtain Let's Encrypt certificates for our domain.

Install cert-manager using Helm or kubectl:

```bash
# Add the Jetstack Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.12.0 \
  --set installCRDs=true
```

Alternatively, you can install with kubectl:

```bash
# Install cert-manager CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.crds.yaml

# Create namespace
kubectl create namespace cert-manager

# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.12.0/cert-manager.yaml
```

Wait for cert-manager to be ready:

```bash
kubectl wait --for=condition=ready pod -l app.kubernetes.io/instance=cert-manager -n cert-manager --timeout=300s
```

Now create a ClusterIssuer for Let's Encrypt:

```yaml
# Create letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    # Replace with your email address
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: traefik
```

Apply the ClusterIssuer:

```bash
kubectl apply -f letsencrypt-issuer.yaml
```

Verify cert-manager is working:

```bash
# Check cert-manager pods
kubectl get pods -n cert-manager

# Check the ClusterIssuer
kubectl get clusterissuer letsencrypt-prod
```

## Step 8: Set Up Ingress for External Access

To access n8n from the internet, we need an ingress controller. Civo clusters come with Traefik pre-installed, so we can use that:

```yaml
# Create n8n-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: n8n-ingress
  namespace: n8n
  annotations:
    kubernetes.io/ingress.class: "traefik"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    traefik.ingress.kubernetes.io/redirect-entry-point: https
spec:
  tls:
    - hosts:
        - n8n.yourdomain.com
      secretName: n8n-tls
  rules:
    - host: n8n.yourdomain.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: n8n-service
                port:
                  number: 80
```

Apply the ingress:

```bash
kubectl apply -f n8n-ingress.yaml
```

## Step 9: Configure DNS

Point your domain to your Civo cluster's load balancer IP. First, get the external IP:

```bash
kubectl get svc -n kube-system traefik
```

Look for the `EXTERNAL-IP` and create an A record in your DNS provider pointing `n8n.yourdomain.com` to this IP.

## Step 10: Verify Your Deployment

Check that everything is running correctly:

```bash
# Check pod status
kubectl get pods -n n8n

# Check service
kubectl get svc -n n8n

# Check ingress
kubectl get ingress -n n8n

# Check certificate (should show Ready=True after a few minutes)
kubectl get certificate -n n8n

# View logs if needed
kubectl logs -f deployment/n8n -n n8n
```

You should see output similar to:

```
NAME   READY   STATUS    RESTARTS   AGE
n8n-xxx   1/1     Running   0          2m
```

For the certificate, you should see:

```
NAME      READY   SECRET    AGE
n8n-tls   True    n8n-tls   2m
```

## Step 11: Access n8n

Once your DNS has propagated (this can take a few minutes), you can access n8n at `https://n8n.yourdomain.com`. The SSL certificate should be automatically provisioned by cert-manager within a few minutes.

You'll be prompted to log in using the credentials you set in the Secret (admin/your-secure-password-here by default).

## Creating Your First Workflow

Now that n8n is running, let's create a simple workflow to test everything:

1. **Access the n8n interface** at your domain
2. **Click "New Workflow"**
3. **Add a Manual Trigger node** - this will be your starting point
4. **Add an HTTP Request node** - configure it to make a GET request to `https://api.github.com/users/octocat`
5. **Add a Set node** - use this to format the response data
6. **Connect the nodes** by dragging from the output of one to the input of the next
7. **Click "Execute Workflow"** to test it

This simple workflow demonstrates n8n's ability to fetch data from APIs and process it.

## Production Considerations

For production deployments, consider these additional steps:

### Database Configuration

Instead of using SQLite (the default), configure n8n to use PostgreSQL for better performance:

```yaml
# Add to your n8n-config ConfigMap
DB_TYPE: "postgresdb"
DB_POSTGRESDB_HOST: "your-postgres-host"
DB_POSTGRESDB_PORT: "5432"
DB_POSTGRESDB_DATABASE: "n8n"
```

### Resource Scaling

Monitor your n8n instance and adjust resources as needed:

```bash
# Scale up if needed
kubectl scale deployment n8n --replicas=2 -n n8n

# Update resource limits
kubectl patch deployment n8n -n n8n -p '{"spec":{"template":{"spec":{"containers":[{"name":"n8n","resources":{"limits":{"memory":"2Gi","cpu":"1"}}}]}}}}'
```

### Backup Strategy

Set up regular backups of your persistent volume and export your workflows:

```bash
# Create a backup job
kubectl create job n8n-backup --from=cronjob/n8n-backup -n n8n
```

### Monitoring

Add monitoring to track n8n's performance:

```yaml
# Add to your deployment
- name: metrics
  port: 9464
  targetPort: 9464
```

## Troubleshooting Common Issues

### Pod Won't Start - Permission Issues

The most common issue is permission problems with the persistent volume. Check the logs:

```bash
kubectl logs deployment/n8n -n n8n
```

If you see `EACCES: permission denied` errors, the deployment above includes fixes for this. However, if you're still having issues, you can manually fix permissions:

```bash
# Delete the current deployment
kubectl delete deployment n8n -n n8n

# Create a one-time job to fix permissions
kubectl apply -f - <<EOF
apiVersion: batch/v1
kind: Job
metadata:
  name: fix-n8n-permissions
  namespace: n8n
spec:
  template:
    spec:
      containers:
      - name: fix-permissions
        image: busybox:1.35
        command:
        - sh
        - -c
        - |
          echo "Fixing permissions for /home/node/.n8n"
          chown -R 1000:1000 /home/node/.n8n
          chmod -R 755 /home/node/.n8n
          echo "Permissions fixed successfully"
        volumeMounts:
        - name: n8n-data
          mountPath: /home/node/.n8n
        securityContext:
          runAsUser: 0
      volumes:
      - name: n8n-data
        persistentVolumeClaim:
          claimName: n8n-data
      restartPolicy: Never
EOF

# Wait for the job to complete
kubectl wait --for=condition=complete job/fix-n8n-permissions -n n8n --timeout=300s

# Clean up the job
kubectl delete job fix-n8n-permissions -n n8n

# Redeploy n8n
kubectl apply -f n8n-deployment.yaml
```

### Pod Won't Start - Other Issues

For other startup issues:

```bash
kubectl logs deployment/n8n -n n8n
```

Common issues include:

- Incorrect environment variables
- Storage mount problems
- Resource constraints

### SSL Certificate Issues

If the certificate isn't working:

```bash
# Check certificate status
kubectl describe certificate n8n-tls -n n8n

# Check cert-manager logs
kubectl logs -n cert-manager deployment/cert-manager

# Check certificate request
kubectl get certificaterequest -n n8n
```

Common certificate issues:

- DNS not pointing to the correct IP
- Rate limiting from Let's Encrypt (wait an hour and try again)
- Incorrect email in ClusterIssuer

### Can't Access via Domain

Verify your setup:

```bash
# Check ingress
kubectl describe ingress n8n-ingress -n n8n

# Check if cert-manager is working
kubectl get certificates -n n8n

# Test internal connectivity
kubectl port-forward svc/n8n-service 8080:80 -n n8n
```

### Workflows Not Persisting

Ensure your PVC is properly mounted:

```bash
kubectl describe pvc n8n-data -n n8n
```

## Next Steps

Now that you have n8n running on Civo Kubernetes, you can:

- **Explore n8n's extensive node library** - connect to hundreds of services
- **Set up webhook workflows** - trigger automation from external services
- **Create scheduled workflows** - automate recurring tasks
- **Build complex multi-step automations** - chain together multiple services
- **Integrate with your existing tools** - connect databases, APIs, and services

## Conclusion

You now have a fully functional n8n instance running on Civo Kubernetes! This setup provides a scalable foundation for your automation needs, with the flexibility to grow as your requirements evolve.

The combination of n8n's powerful workflow capabilities and Civo's developer-friendly Kubernetes platform gives you a robust automation solution that's both easy to manage and cost-effective.

Remember to regularly update your n8n deployment, monitor resource usage, and back up your workflows to ensure a smooth operation.

Happy automating!
