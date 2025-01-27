---
## Table of Contents
- [Deployment](#deployment)
  - [Namespace](#namespace)
  - [Secret](#secret)
  - [ConfigMap](#configmap)
  - [Creating PVCs](#creating-pvcs)
  - [Postgres](#postgres)
  - [Clickhouse deployment](#clickhouse-deployment)
  - [Simple Mail Server](#simple-mail-server)
  - [The Plausible Application Deployment](#the-plausible-application-deployment)
  - [Services for Postgres, Clickhouse, Plausible and Mail](#services-for-postgres-clickhouse-plausible-and-mail)
  - [Ingress](#ingress)
- [Additional Recommendations and Insights](#additional-recommendations-and-insights)
  - [Choosing Civo Node Sizes](#choosing-civo-node-sizes)
  - [Using Managed Databases](#using-managed-databases)
  - [Traefik Configuration](#traefik-configuration)
  - [Securing Secrets](#securing-secrets)
  - [Monitoring and Logging](#monitoring-and-logging)
  - [Backups and Disaster Recovery](#backups-and-disaster-recovery)
  - [Testing and Staging Environment](#testing-and-staging-environment)



Create Cluster:

> You can specify a name for the cluster after the create command or leave this blank and one will be generated for you:

```bash
civo k8s create my-cluster --nodes=3 --size=g4s.kube.medium --wait --save --merge

```

```bash
civo k8s create --nodes=3 --size=g4s.kube.medium --wait --save --merge

```

Create namespace

```bash
kubectl create namespace plausible
```

Verify

```bash
kubectl get nodes
```

Cert Manager:

```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set installCRDs=true
```

```bash
kubectl get pods --namespace cert-manager
```

Issuers for staging and prod:

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
    # Change this to your email address
    email: your-email@example.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      # Secret used to store the account's private key
      name: letsencrypt-production-key
    solvers:
    - http01:
        ingress:
          class: traefik
          
```

```bash
kubectl apply -f issuer.yaml
```

## Deployment

### Namespace 

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: plausible
```

### Secret

As you can see below, some examples are given, you will need to replace these with your own values. Ensure these are base64 encoded.

```yaml

apiVersion: v1
kind: Secret
metadata:
  name: plausible-config
  namespace: plausible
type: Opaque
data:
  POSTGRES_USER: cGxhdXNpYmxl # base64 of "plausible"
  POSTGRES_PASSWORD: cGxhdXNpYmxlX3Bhc3N3b3Jk # base64 of "plausible_password"
  POSTGRES_DB: cGxhdXNpYmxlX2Ri # base64 of "plausible_db"
  DATABASE_URL: cG9zdGdyZXM6Ly9wbGF1c2libGU6cGxhdXNpYmxlX3Bhc3N3b3JkQHBsYXVzaWJsZS1wb3N0Z3Jlc3FsOjU0MzIvcGxhdXNpYmxlX2Ri # base64 of "postgres://plausible:plausible_password@plausible-postgres:5432/plausible_db"
```

### ConfigMap

Next we create the config map with the relevant values, again replace with your domain etc. You will need to generate a random SECRET_KEY_BASE, which you can do by:

```bash
openssl rand -base64 48
```
Use the generated value and your domain value below:
```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: plausible-configmap
  namespace: plausible
data:
  BASE_URL: "https://YOUR_DOMAIN"
  SECRET_KEY_BASE: "GENERATED KEY"
  CLICKHOUSE_DATABASE_URL: "http://plausible-clickhouse:8123/plausible_events_db"

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

```

### Postgres

You can of course use an external postgres database, for example a Civo managed db, but for this example we will deploy a database pod.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible-postgresql
  namespace: plausible
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plausible-postgresql
  template:
    metadata:
      labels:
        app: plausible-postgresql
    spec:
      containers:
      - name: postgresql
        image: postgres:14
        ports:
        - containerPort: 5432
        env:
        - name: POSTGRES_USER
          valueFrom:
            secretKeyRef:
              name: plausible-config
              key: POSTGRES_USER
        - name: POSTGRES_PASSWORD
          valueFrom:
            secretKeyRef:
              name: plausible-config
              key: POSTGRES_PASSWORD
        - name: POSTGRES_DB
          valueFrom:
            secretKeyRef:
              name: plausible-config
              key: POSTGRES_DB
        - name: PGDATA
          value: /var/lib/postgresql/data/pgdata # Use a subdirectory for PostgreSQL data
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-data
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc

```
### Clickhouse deployment

Plausible Analytics uses ClickHouse as its primary database for storing and querying analytics data. ClickHouse is a columnar database management system known for its high performance and efficiency in handling large volumes of data. It is optimized for read-heavy operations, making it ideal for real-time analytics and reporting. By leveraging ClickHouse, Plausible can provide fast and accurate insights into website traffic and user behavior, ensuring a smooth and responsive experience for its users.


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible-clickhouse
  namespace: plausible
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plausible-clickhouse
  template:
    metadata:
      labels:
        app: plausible-clickhouse
    spec:
      containers:
      - name: clickhouse
        image: clickhouse/clickhouse-server:24.3.3.102-alpine
        ports:
        - containerPort: 8123
        volumeMounts:
        - mountPath: /var/lib/clickhouse
          name: clickhouse-data
      volumes:
      - name: clickhouse-data
        persistentVolumeClaim:
          claimName: clickhouse-pvc

```
### Simple Mail Server

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
### The Plausible Application Deployment


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: plausible
  namespace: plausible
spec:
  replicas: 1
  selector:
    matchLabels:
      app: plausible
  template:
    metadata:
      labels:
        app: plausible
    spec:
      initContainers:
      - name: wait-for-postgresql
        image: busybox
        command: ['sh', '-c', 'until nc -z plausible-postgresql 5432; do echo "Waiting for PostgreSQL..."; sleep 2; done']
      containers:
      - name: plausible
        image: ghcr.io/plausible/community-edition:v2.1.4
        command: ["/bin/sh", "-c", "/entrypoint.sh db createdb && /entrypoint.sh db migrate && /entrypoint.sh run"]
        ports:
        - containerPort: 8000
        envFrom:
        - secretRef:
            name: plausible-config
        - configMapRef:
            name: plausible-configmap

```
### Services for Postgres, Clickhouse, Plausible and Mail

```yaml
apiVersion: v1
kind: Service
metadata:
  name: plausible-postgresql
  namespace: plausible
spec:
  ports:
  - port: 5432
    targetPort: 5432
  selector:
    app: plausible-postgresql

---
apiVersion: v1
kind: Service
metadata:
  name: plausible-clickhouse
  namespace: plausible
spec:
  ports:
  - port: 8123
    targetPort: 8123
  selector:
    app: plausible-clickhouse

---
apiVersion: v1
kind: Service
metadata:
  name: plausible-mail
  namespace: plausible
spec:
  ports:
  - port: 25
    targetPort: 25
  selector:
    app: plausible-mail

---
apiVersion: v1
kind: Service
metadata:
  name: plausible
  namespace: plausible
spec:
  ports:
  - port: 80
    targetPort: 8000
  selector:
    app: plausible

```
### Ingress

Lastly we can deploy the ingress. As you will see we have commented out the production issuer until the domain is ready.

```yaml

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: plausible-ingress
  namespace: plausible
  annotations:
    # cert-manager.io/cluster-issuer: letsencrypt-production
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













## Additional Recommendations and Insights

### Choosing Civo Node Sizes
The node size `g4s.kube.medium` specified in the cluster creation is suitable for most small to medium workloads. However, if you anticipate higher traffic or need more processing power, you can explore other Civo node sizes. Ensure your cluster has sufficient resources to handle database workloads and analytics processing.

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
