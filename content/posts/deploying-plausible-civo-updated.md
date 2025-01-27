---
title: "Deploying Plausible to Civo NEW"
date: 2025-01-22
draft: false
tags: ["Civo", "Kubernetes"]
---
Deploying Plausible Analytics on a Kubernetes cluster can significantly enhance your ability to manage and scale your analytics infrastructure. This guide will walk you through the process of deploying Plausible on Civo Kubernetes, leveraging various Kubernetes resources and tools to ensure a smooth and efficient setup.

### Table of Contents
- [Encode the values in base64](#encode-the-values-in-base64)
- [Create the secret.yaml file with the encoded values](#create-the-secretyaml-file-with-the-encoded-values)
    - [ConfigMap](#configmap)
    - [Creating PVCs](#creating-pvcs)
    - [Clickhouse DeploymentmName: clickhouse-pvc](#clickhouse-deploymentmname-clickhouse-pvc)
    - [The Plausible Application DeploymentetRef:](#the-plausible-application-deploymentetref)
  - [Additional Recommendations and Insights](#additional-recommendations-and-insights)
    - [Choosing Civo Node SizesThe node size `g4s.kube.medium` specified in the cluster creation is suitable for most small to medium workloads. However, if you anticipate higher traffic or need more processing power, you can explore other Civo node sizes. Ensure your cluster has sufficient resources to handle database workloads and analytics processing.](#choosing-civo-node-sizesthe-node-size-g4skubemedium-specified-in-the-cluster-creation-is-suitable-for-most-small-to-medium-workloads-however-if-you-anticipate-higher-traffic-or-need-more-processing-power-you-can-explore-other-civo-node-sizes-ensure-your-cluster-has-sufficient-resources-to-handle-database-workloads-and-analytics-processing)
    - [Using Managed Databases](#using-managed-databases)
    - [Traefik Configuration](#traefik-configuration)
    - [Securing Secrets](#securing-secrets)
    - [Monitoring and Logging](#monitoring-and-logging)
    - [Backups and Disaster Recovery](#backups-and-disaster-recovery)
    - [Testing and Staging Environment](#testing-and-staging-environment)

### Prerequisites

Before you begin, ensure you have the following:

- **Basic Knowledge of Kubernetes**: Familiarity with Kubernetes concepts such as pods, deployments, services, and namespaces.
- **Civo Account**: A registered account on Civo with access to create Kubernetes clusters.
- **kubectl**: The Kubernetes command-line tool installed and configured to interact with your Civo cluster.
- **Helm**: The package manager for Kubernetes installed on your local machine.
- **Domain Name**: A domain name for accessing your Plausible instance, with DNS records configured to point to your cluster's ingress controller.

### Required Tools

- **kubectl**: For managing Kubernetes clusters. [Install kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- **Helm**: For deploying applications on Kubernetes. [Install Helm](https://helm.sh/docs/intro/install/)
- **Civo CLI**: For managing Civo resources. [Install Civo CLI](https://www.civo.com/docs/cli)
- **OpenSSL**: For generating secure keys and secrets. [Install OpenSSL](https://www.openssl.org/source/)

By the end of this guide, you will have a fully functional instance of Plausible Analytics running on your Civo Kubernetes cluster, complete with SSL certificates, database configurations, and ingress setup.

## Create Cluster

You can specify a name for the cluster after the create command or leave this blank and one will be generated for you:

```bash
civo k8s create my-cluster --nodes=3 --size=g4s.kube.medium --wait --save --merge
```

```bash
civo k8s create --nodes=3 --size=g4s.kube.medium --wait --save --merge
```

## Create Namespace

```bash
kubectl create namespace plausible
```

## Verify

```bash
kubectl get nodes
```

## Cert Manager

Add the Jetstack Helm repository and update it:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

Install Cert Manager:

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true
```

Verify Cert Manager installation:

```bash
kubectl get pods --namespace cert-manager
```

## Issuers for Staging and Production

Create issuers for staging and production:

```bash
cat <<EOF > issuer.yaml
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
EOF
```

Apply the issuer configuration:

```bash
kubectl apply -f issuer.yaml
```

## Deployment

The deployment has been broken down into the various resources, however there is no reason you can't just create a single deployment.yaml file, you can see an example of this in the github repo for this project.

Let's start by creating the directory for our files and then opening in a code editor:

```bash

mkdir civo-plausible
cd civo-plausible

```

### Namespace

Create a file called `namespace.yaml` with the following content:

```bash
cat <<EOF > namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: plausible
EOF
```

### Secret

Replace the example values with your own:

# Encode the values in base64
```bash
POSTGRES_USER_BASE64=$(echo -n "POSTGRES_USER" | base64) 
POSTGRES_PASSWORD_BASE64=$(echo -n "POSTGRES_PASSWORD" | base64) 
POSTGRES_DB_BASE64=$(echo -n "POSTGRES_DB" | base64) 
DATABASE_URL_BASE64=$(echo -n "DATABASE_URL" | base64) 
```

# Create the secret.yaml file with the encoded values
```bash
cat <<EOF > secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: plausible-config
  namespace: plausible
type: Opaque
data:
  POSTGRES_USER: $POSTGRES_USER_BASE64
  POSTGRES_PASSWORD: $POSTGRES_PASSWORD_BASE64
  POSTGRES_DB: $POSTGRES_DB_BASE64
  DATABASE_URL: $DATABASE_URL_BASE64
EOF
```
### ConfigMap

Generate a random `SECRET_KEY_BASE` and encode it in base64:

```bash
SECRET_KEY_BASE=$(openssl rand -base64 48 | base64)
```

Use the generated value and your domain:

```bash
cat <<EOF > configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plausible-configmap
  namespace: plausible
data:
  BASE_URL: "https://YOUR_DOMAIN"
  SECRET_KEY_BASE: "$SECRET_KEY_BASE"
  CLICKHOUSE_DATABASE_URL: "http://plausible-clickhouse:8123/plausible_events_db"
EOF
```
### Creating PVCs



```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: plausible
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: clickhouse-pvc
  namespace: plausible
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
```-config
OSTGRES_USER


Deploy a PostgreSQL pod:
fig
```yamlOSTGRES_PASSWORD
apiVersion: apps/v1DB
kind: Deployment
metadata:
  name: plausible-postgresqlle-config
  namespace: plausibleOSTGRES_DB
spec:
  replicas: 1/data/pgdata
  selector:
    matchLabels:var/lib/postgresql/data
      app: plausible-postgresql
  template:
    metadata:
      labels::
        app: plausible-postgresqlmName: postgres-pvc
    spec:
      containers:
      - name: postgresql
        image: postgres:14
        ports:Deploy ClickHouse:
        - containerPort: 5432
        env:```yaml
        - name: POSTGRES_USER1
          valueFrom:kind: Deployment
            secretKeyRef:a:
              name: plausible-configlickhouse
              key: POSTGRES_USERusible
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: plausible-configatchLabels:
              key: POSTGRES_PASSWORDausible-clickhouse
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: plausible-config: plausible-clickhouse
              key: POSTGRES_DB
        - name: PGDATAers:
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:mage: clickhouse/clickhouse-server:24.3.3.102-alpine
        - mountPath: /var/lib/postgresql/data
          name: postgres-data 8123
      volumes:
      - name: postgres-datatPath: /var/lib/clickhouse
        persistentVolumeClaim:ta
          claimName: postgres-pvc
```

### Clickhouse DeploymentmName: clickhouse-pvc

Deploy ClickHouse:

```yaml
apiVersion: apps/v1Deploy a simple mail server:
kind: Deployment
metadata:```yaml
  name: plausible-clickhouse
  namespace: plausiblekind: Deployment
spec:a:
  replicas: 1ail
  selector:usible
    matchLabels:
      app: plausible-clickhouse
  template:
    metadata:atchLabels:
      labels:ausible-mail
        app: plausible-clickhouse
    spec:
      containers:
      - name: clickhouse: plausible-mail
        image: clickhouse/clickhouse-server:24.3.3.102-alpine
        ports:ers:
        - containerPort: 8123
        volumeMounts:mage: bytemark/smtp
        - mountPath: /var/lib/clickhouse
          name: clickhouse-datarPort: 25
      volumes:
      - name: clickhouse-data
        persistentVolumeClaim:on Deployment
          claimName: clickhouse-pvc
```Deploy the Plausible application:

### Simple Mail Server```yaml

Deploy a simple mail server:kind: Deployment
a:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible-mail
  namespace: plausible
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plausible-mail
  template:
    metadata:
      labels:
        app: plausible-mail
    spec:
      containers:
      - name: mail
        image: bytemark/smtp
        ports:
        - containerPort: 25
```

### The Plausible Application DeploymentetRef:
onfig
Deploy the Plausible application:MapRef:
ausible-configmap
```yaml
apiVersion: apps/v1
kind: Deployment, Plausible, and Mail
metadata:
  name: plausibleCreate services for each component:
  namespace: plausible
spec:```yaml
  replicas: 1
  selector:kind: Service
    matchLabels:a:
      app: plausibleble-postgresql
  template:plausible
    metadata:
      labels:
        app: plausible
    spec:argetPort: 5432
      initContainers:or:
      - name: wait-for-postgresqlible-postgresql
        image: busybox
        command: ['sh', '-c', 'until nc -z plausible-postgresql 5432; do echo "Waiting for PostgreSQL..."; sleep 2; done'] v1
      containers:
      - name: plausibleadata:
        image: ghcr.io/plausible/community-edition:v2.1.4ble-clickhouse
        command: ["/bin/sh", "-c", "/entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"]plausible
        ports:
        - containerPort: 8000
        envFrom:
        - secretRef:argetPort: 8123
            name: plausible-configor:
        - configMapRef:ible-clickhouse
            name: plausible-configmap
``` v1

### Services for Postgres, Clickhouse, Plausible, and Mailadata:
ble-mail
Create services for each component:plausible

```yaml
apiVersion: v1
kind: ServiceargetPort: 25
metadata:or:
  name: plausible-postgresqlusible-mail
  namespace: plausible
spec: v1
  ports:
  - port: 5432adata:
    targetPort: 5432ble
  selector:plausible
    app: plausible-postgresql
---
apiVersion: v1
kind: ServiceargetPort: 8000
metadata:or:
  name: plausible-clickhouseusible
  namespace: plausible
spec:
  ports:
  - port: 8123
    targetPort: 8123Deploy the ingress. Comment out the production issuer until the domain is ready:
  selector:
    app: plausible-clickhouse```yaml
---
apiVersion: v1kind: Ingress
kind: Servicea:
metadata:
  name: plausible-mailplausible
  namespace: plausibleions:
spec:ster-issuer: letsencrypt-production
  ports:uster-issuer: letsencrypt-staging
  - port: 25gress.kubernetes.io/redirect-entry-point: https
    targetPort: 25
  selector:
    app: plausible-mail
---ttp:
apiVersion: v1ths:
kind: Service
metadata:athType: Prefix
  name: plausibleend:
  namespace: plausiblece:
spec:ble
  ports::
  - port: 80er: 80
    targetPort: 8000
  selector:
    app: plausible
```cretName: plausible-tls

### Ingress
 and Insights
Deploy the ingress. Comment out the production issuer until the domain is ready:
### Choosing Civo Node Sizes
```yaml
apiVersion: networking.k8s.io/v1The node size `g4s.kube.medium` specified in the cluster creation is suitable for most small to medium workloads. However, if you anticipate higher traffic or need more processing power, you can explore other Civo node sizes. Ensure your cluster has sufficient resources to handle database workloads and analytics processing.
kind: Ingress
metadata:### Using Managed Databases
  name: plausible-ingress
  namespace: plausibleWhile this guide demonstrates deploying PostgreSQL and ClickHouse directly in the cluster, consider using managed database services for production environments. Managed services offer automated backups, scaling, and monitoring, reducing operational overhead.
  annotations:
    # cert-manager.io/cluster-issuer: letsencrypt-production### Traefik Configuration
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/redirect-entry-point: httpsTraefik is used as the ingress controller in this setup. If you have not installed Traefik in your cluster, you will need to deploy it before applying the ingress configuration. The official Traefik Helm chart provides an easy way to set it up.
spec:
  rules:### Securing Secrets
  - host: YOUR_DOMAIN
    http:Secrets are stored in Kubernetes but are only base64-encoded by default. For enhanced security, consider integrating Kubernetes with a secret management tool like HashiCorp Vault or AWS Secrets Manager to encrypt and manage your secrets.
      paths:
      - path: /### Monitoring and Logging
        pathType: Prefix
        backend:Deploy monitoring and logging tools such as Prometheus, Grafana, and Loki. These tools help you gain insights into your cluster's performance and diagnose issues effectively.
          service:
            name: plausible### Backups and Disaster Recovery
            port:
              number: 80Regularly back up your PostgreSQL and ClickHouse databases. Use tools like Velero for cluster backups or database-specific tools for targeted backups.
  tls:
  - hosts:### Testing and Staging Environment
    - YOUR_DOMAIN
    secretName: plausible-tlsBefore switching to production, thoroughly test your deployment in a staging environment. Use the staging Let's Encrypt issuer to avoid hitting rate limits and to test TLS configurations.
```
---
## Additional Recommendations and Insights
By following the steps outlined above and implementing these additional recommendations, you can deploy a robust and reliable instance of Plausible Analytics on Civo Kubernetes. With privacy-first analytics becoming increasingly important, hosting your instance of Plausible gives you complete control over your data and infrastructure.
### Choosing Civo Node SizesThe node size `g4s.kube.medium` specified in the cluster creation is suitable for most small to medium workloads. However, if you anticipate higher traffic or need more processing power, you can explore other Civo node sizes. Ensure your cluster has sufficient resources to handle database workloads and analytics processing.

### Using Managed Databases

While this guide demonstrates deploying PostgreSQL and ClickHouse directly in the cluster, consider using managed database services for production environments. Managed services offer automated backups, scaling, and monitoring, reducing operational overhead.

### Traefik Configuration

Traefik is used as the ingress controller in this setup. If you have not installed Traefik in your cluster, you will need to deploy it before applying the ingress configuration. The official Traefik Helm chart provides an easy way to set it up.

### Securing Secrets

Secrets are stored in Kubernetes but are only base64-encoded by default. For enhanced security, consider integrating Kubernetes with a secret management tool like HashiCorp Vault or AWS Secrets Manager to encrypt and manage your secrets.

### Monitoring and Logging

Deploy monitoring and logging tools such as Prometheus, Grafana, and Loki. These tools help you gain insights into your cluster's performance and diagnose issues effectively.

### Backups and Disaster Recovery

Regularly back up your PostgreSQL and ClickHouse databases. Use tools like Velero for cluster backups or database-specific tools for targeted backups.

### Testing and Staging Environment

Before switching to production, thoroughly test your deployment in a staging environment. Use the staging Let's Encrypt issuer to avoid hitting rate limits and to test TLS configurations.

---

By following the steps outlined above and implementing these additional recommendations, you can deploy a robust and reliable instance of Plausible Analytics on Civo Kubernetes. With privacy-first analytics becoming increasingly important, hosting your instance of Plausible gives you complete control over your data and infrastructure.
