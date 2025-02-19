---
title: "Deploying Plausible to Civo Kubernetes"
date: 2025-01-22
draft: false
tags: ["Civo", "Kubernetes"]
---

In this guide, we will walk you through the process of deploying Plausible Analytics to a Civo Kubernetes cluster. Plausible Analytics is a lightweight and open-source web analytics tool that prioritizes privacy and simplicity. By the end of this tutorial, you will have a fully functional Plausible Analytics instance running on your Civo Kubernetes cluster.

## Prerequisites

Before you begin, ensure you have the following setup:

1. **Civo Account**: A [Civo account](https://www.civo.com/signup) with access to create Kubernetes clusters.
2. **Civo CLI**: The [Civo CLI](https://www.civo.com/docs/cli) installed and configured on your local machine.
3. **kubectl**: The [Kubernetes command-line tool](https://kubernetes.io/docs/tasks/tools/) installed and configured to interact with your cluster.
4. **Helm**: The [Helm package manager](https://helm.sh/docs/intro/install/) installed on your local machine.
5. **Domain Name**: A domain name that you can use for your Plausible Analytics instance.
6. **Email Address**: An email address to use for Let's Encrypt SSL certificates.

With these prerequisites in place, you are ready to proceed with the deployment.

## Create Cluster

We will now use the Civo CLI to create the cluster, waiting for it to provion and automatically saving and merging the KUBECONFIG. You may want to review the Civo documentation for further information.

> You can specify a name for the cluster after the create command:

```bash
civo k8s create my-cluster --nodes=3 --size=g4s.kube.medium --wait --save --merge

```

Or leave this blank and one will be generated for you:

```bash
civo k8s create --nodes=3 --size=g4s.kube.medium --wait --save --merge

```

Verify the cluster is up and running:

```bash
kubectl get nodes
```

## Cert Manager

Cert Manager is a Kubernetes add-on that automates the management and issuance of TLS certificates from various issuing sources. It ensures that certificates are valid and up-to-date, and it will attempt to renew certificates at an appropriate time before expiry. In this guide, we will use Cert Manager to handle the SSL certificates for our Plausible Analytics instance.

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

## Setting Up the Project

To get started, create a new folder for your project and open it in Visual Studio Code (VSCode). This will help you organize your files and make it easier to follow along with the tutorial.

```bash
mkdir civo-plausible-demo
cd civo-plausible-demo
code .
```

This will open the `civo-plausible-demo` folder in VSCode, where you can create and manage your Kubernetes manifests and other configuration files.

## Creating k8s manifests

Creating Kubernetes manifests involves defining the desired state of your Kubernetes resources using YAML files. These manifests describe various aspects of your application, such as deployments, services, config maps, secrets, and persistent volume claims (PVCs). By applying these manifests to your Kubernetes cluster, you instruct Kubernetes to create and manage the resources accordingly. In this section, we will create the necessary Kubernetes manifests to deploy Plausible Analytics and its dependencies on your Civo Kubernetes cluster.

### Certificate Issuers

Cert issuers are responsible for obtaining and managing SSL/TLS certificates for secure communication in your Kubernetes cluster.

Issuers for staging and prod:

```bash
cat <<'EOF' > issuers.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    email: change@me.com
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
    email: change@me.com
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
    - http01:
        ingress:
          class: traefik
EOF
```

You can then edit this newly created file and update the email address used by cert manager to your own.

## Plausible Deployment

### Namespace 

We will next create a namespace so plausible is segregated from other applications on the cluster.

```bash
cat <<'EOF' > namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: plausible
EOF
```

```bash
kubectl apply -f namespace.yaml
```

### Secret

Next we will create the Kubernetes secret which will hold some values used by Plausible. As with any secret held in Kubernetes secrets, these are only base64 encoded and therefore is only really designed for simple deployments. It is worth considering an external secrets manager to provide better management of secrets, one simple and free secrets manager is [Bitwarden Secrets Manager](https://bitwarden.com/help/secrets-manager-quick-start/).

Below is an example of how to create your base64 encoded values:

```bash
echo -n "plausible" | base64
echo -n "plausible_password" | base64
echo -n "plausible_db" | base64
echo -n "postgres://plausible:plausible_password@plausible-postgres:5432/plausible_db" | base64
```

We can then create and update the relevant values with those generated above:

```yaml
cat <<'EOF' > secret.yaml
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
EOF
```
### ConfigMap

Next we create the config map with the relevant values, again replace with your domain etc. You will need to generate a random SECRET_KEY_BASE, which you can do by:

```bash
openssl rand -base64 48
```

We can then create the config map and alter the values including the generated key to use as the SECRET_KEY_BASE.

When deploying a cluster to Civo, you are automatically allocated a DNS record for your cluster. If you don't have a domain you can use this. 

You can find this value by using the civo cli:

List out your clusters:

```bash
civo k3s ls
```

Use the cluster name to find the DNS record:

```bash
civo k3s show crimson-darkness-b519665c | grep "DNS A record"
```

We can then use this, or your actual DNS record, when creating the config for Plausible:

> I will be using my example value in this demo later when we create the ingress so make sure you replace this with your own.

```bash
cat <<'EOF' > configMap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: plausible-configmap
  namespace: plausible
data:
  BASE_URL: "https://6c01f85b-7f17-4fa6-bc59-1a6c16f1576d.k8s.civo.com"
  SECRET_KEY_BASE: "bUxqWWdaSmQ3cjdkQXpqdmpTTE5CMldIZ1pXWlNZc2ZBU3dxRFpnV3o1UWpOUk9MS2hUR2F1U1Q1RUVKRjFScQo="
  CLICKHOUSE_DATABASE_URL: "http://plausible-clickhouse:8123/plausible_events_db"
EOF
```
### Creating PVCs

Plausible and the supporting applications require persistant storage, next we will create this storage. In this demo we are using postgres and clickhouse services inside the cluster, if you are deploying at scale you may wish to look at external services. 

```bash
cat <<'EOF' > pvcs.yaml
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
EOF
```

### Postgres

As mentioned previously you can use an external postgres database, for example a [Civo managed db](https://www.civo.com/docs/database/postgresql). For the purpose of this demo, we will keep things simple and deploy postgres into the cluster.

```bash
cat <<'EOF' > postgres.yaml
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
          value: /var/lib/postgresql/data/pgdata
        volumeMounts:
        - mountPath: /var/lib/postgresql/data
          name: postgres-data
      volumes:
      - name: postgres-data
        persistentVolumeClaim:
          claimName: postgres-pvc
EOF
```
### Clickhouse deployment

Plausible Analytics uses ClickHouse as its primary database for storing and querying analytics data. ClickHouse is a columnar database management system known for its high performance and efficiency in handling large volumes of data. It is optimized for read-heavy operations, making it ideal for real-time analytics and reporting. By leveraging ClickHouse, Plausible can provide fast and accurate insights into website traffic and user behavior, ensuring a smooth and responsive experience for its users.


```bash
cat <<'EOF' > clickhouse.yaml
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
EOF
```
### Simple Mail Server



```bash
cat <<'EOF' > mail.yaml
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
EOF
```
### The Plausible Application Deployment

And finally we get to the actual deployment of Plausible:

```bash
cat <<'EOF' > deployment.yaml
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
EOF
```
### Services for Postgres, Clickhouse, Plausible and Mail

```bash
cat <<'EOF' > services.yaml
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
EOF
```
### Ingress

Lastly we can deploy the ingress. As you will see we have commented out the production issuer until the domain is ready.

> As mentioned earlier, here I am using the example DNS record for the cluster, you will need to update this with your own.

```bash
cat <<'EOF' > ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: plausible-ingress
  namespace: plausible
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-staging
    traefik.ingress.kubernetes.io/redirect-scheme: https
    traefik.ingress.kubernetes.io/redirect-permanent: "true"
spec:
  rules:
  - host: 6c01f85b-7f17-4fa6-bc59-1a6c16f1576d.k8s.civo.com
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
    - 6c01f85b-7f17-4fa6-bc59-1a6c16f1576d.k8s.civo.com
    secretName: plausible-tls
EOF
```

## Deploying

Now we have all the files created, we can now easily deploy the application:

First we need to create the namespace:

```bash
kubectl apply -f namespace.yaml
```

Then with this created all the remaining resources:

```bash
kubectl apply -f .
```

You can review the status of everything in the namespace:

> Give this a few minutes, maybe make a ☕️

```bash
kubectl get all -n plausible
```

Then we should be able to access and URL and setup Plausible!

> Don't forget when you are happy the DNS is working you can update the ingress to use the production cert:

```bash
kubectl annotate ingress plausible-ingress -n plausible \
  cert-manager.io/cluster-issuer=letsencrypt-production --overwrite
```

Hope you enjoyed working through this guide! 














