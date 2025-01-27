


In this blog post, we will explore how to deploy Plausible Analytics to a Civo Kubernetes cluster. Plausible is a lightweight and open-source web analytics tool that prioritizes privacy and simplicity. It provides website owners with essential insights without compromising user privacy.

Civo is a cloud service provider that offers a developer-friendly Kubernetes platform. With its fast and efficient Kubernetes clusters, Civo makes it easy to deploy and manage containerized applications.

By the end of this guide, you will have a fully functional Plausible Analytics instance running on a Civo Kubernetes cluster. Let's get started!

## Prerequisites

Before you begin, ensure you have the following prerequisites:

1. **Civo Account**: You need a Civo account. If you don't have one, you can sign up at [Civo's website](https://www.civo.com/).
2. **Civo CLI**: Install the Civo CLI tool on your local machine. You can find the installation instructions [here](https://www.civo.com/docs/cli).
3. **kubectl**: Ensure you have `kubectl` installed and configured to interact with your Kubernetes cluster. You can download it from [Kubernetes' official site](https://kubernetes.io/docs/tasks/tools/install-kubectl/).
4. **Helm**: Install Helm, the package manager for Kubernetes. Follow the installation guide [here](https://helm.sh/docs/intro/install/).
5. **Docker**: Make sure Docker is installed and running on your local machine. You can download it from [Docker's official site](https://www.docker.com/products/docker-desktop).
6. **Domain Name**: A domain name for your Plausible instance. You can purchase one from any domain registrar.
7. **Basic Knowledge**: Familiarity with Kubernetes concepts and command-line tools.

Once you have these prerequisites in place, you are ready to proceed with setting up your Civo Kubernetes cluster.

## Setting Up Civo Kubernetes Cluster

To set up a Civo Kubernetes cluster using the CLI, follow these steps:

1. **Log in to Civo CLI**: Open your terminal and log in to your Civo account using the CLI.
    ```sh
    civo apikey save
    Enter a nice name for this account/API Key: civo-demo
    Enter the API key:
    Saved the API Key civo-demo
    ```

2. **Set the Default Region**: Set the default region for your Civo account. Replace `LON1` with your desired region.
    ```sh
    civo region ls
    civo region use LON1
    ```

3. **Create a Kubernetes Cluster**: Use the following command to create a new Kubernetes cluster. Replace `my-cluster` with your desired cluster name. The `--wait` flag ensures the command waits until the cluster is ready, and the `--save` and `--merge` flags automatically configure `kubectl` to use the new cluster.
    ```sh
    civo k8s create my-cluster --nodes=3 --size=g4s.kube.medium --wait --save --merge
    ```

4. **Verify the Configuration**: Verify that `kubectl` is configured correctly by listing the nodes in your cluster.
    ```sh
    kubectl get nodes
    ```

Your Civo Kubernetes cluster is now set up and ready for deploying applications.

## Deploying Cert-Manager for HTTPS

To serve Plausible over HTTPS, we need to deploy Cert-Manager to our Kubernetes cluster. Cert-Manager automates the management and issuance of TLS certificates.

1. **Add the Jetstack Helm Repository**: First, add the Jetstack Helm repository, which contains the Cert-Manager Helm chart.
    ```sh
    helm repo add jetstack https://charts.jetstack.io
    helm repo update
    ```

2. **Install Cert-Manager**: Use Helm to install Cert-Manager in your cluster.
    ```sh
    helm install cert-manager jetstack/cert-manager --namespace cert-manager --create-namespace --version v1.6.1 --set installCRDs=true
    ```

3. **Verify the Installation**: Ensure that Cert-Manager is installed and running.
    ```sh
    kubectl get pods --namespace cert-manager
    ```

4. **Create a ClusterIssuer**: Create a ClusterIssuer resource to define how certificates will be issued. Save the following YAML to a file named `cluster-issuer.yaml`:

    For production certificates:
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-prod
    spec:
      acme:
        server: https://acme-v02.api.letsencrypt.org/directory
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-prod
        solvers:
        - http01:
            ingress:
              class: nginx
    ```

    For staging certificates (useful for testing to avoid hitting rate limits):
    ```yaml
    apiVersion: cert-manager.io/v1
    kind: ClusterIssuer
    metadata:
      name: letsencrypt-staging
    spec:
      acme:
        server: https://acme-staging-v02.api.letsencrypt.org/directory
        email: your-email@example.com
        privateKeySecretRef:
          name: letsencrypt-staging
        solvers:
        - http01:
            ingress:
              class: nginx
    ```

    Apply the ClusterIssuer to your cluster:
    ```sh
    kubectl apply -f cluster-issuer.yaml
    ```

    **Note**: It is recommended to use the staging environment first to ensure your configuration is correct without hitting Let's Encrypt's rate limits. Once verified, you can switch to the production environment.

With Cert-Manager deployed and configured, you can now proceed to install Plausible Analytics and configure it to use HTTPS.

## Installing Plausible Analytics

To install Plausible Analytics on your Civo Kubernetes cluster, follow these steps:

1. **Create a Namespace for Plausible**: Create a new namespace in your Kubernetes cluster for Plausible.
    ```sh
    kubectl create namespace plausible
    ```

2. **Create a PostgreSQL Database**: Use the Civo CLI to create a new PostgreSQL database. Replace `my-database` with your desired database name.
    ```sh
    civo db create my-database --size=g3.db.small --software psql --version 14
    ```

3. **Retrieve Database Credentials**: Once the database is created, retrieve the credentials and connection details.
    ```sh
    civo db show my-database
    civo db credential my-database
    ```

4. **Create a Secret for Plausible Configuration**: Create a Kubernetes secret to store your Plausible configuration. Replace the placeholder values with your actual configuration.
    ```sh
    kubectl create secret generic plausible-config --namespace plausible \
      --from-literal=SECRET_KEY_BASE="$(openssl rand -base64 64)" \
      --from-literal=BASE_URL="https://URL" \
      --from-literal=POSTGRES_PASSWORD="password" \
      --from-literal=POSTGRES_DB="plausible_db" \
      --from-literal=POSTGRES_USER="DB USER" \
      --from-literal=POSTGRES_HOST="DB URL" \
      --from-literal=POSTGRES_PORT="5432"
    ```


5. **Create a Deployment for Plausible**: Create a file named `plausible-deployment.yaml` with the following content:
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
          containers:
          - name: plausible
            image: plausible/analytics:latest
            env:
            - name: SECRET_KEY_BASE
              valueFrom:
                secretKeyRef:
                  name: plausible-config
                  key: SECRET_KEY_BASE
            - name: BASE_URL
              valueFrom:
                secretKeyRef:
                  name: plausible-config
                  key: BASE_URL
            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: plausible-config
                  key: DATABASE_URL
            ports:
            - containerPort: 8000
    ```

6. **Create a Service for Plausible**: Create a file named `plausible-service.yaml` with the following content:
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: plausible
      namespace: plausible
    spec:
      selector:
        app: plausible
      ports:
      - protocol: TCP
        port: 80
        targetPort: 8000
    ```

7. **Deploy the Manifests**: Apply the manifests to your Kubernetes cluster.
    ```sh
    kubectl apply -f plausible-deployment.yaml --namespace plausible
    kubectl apply -f plausible-service.yaml --namespace plausible
    ```

8. **Deploy Ingress**: Create an Ingress resource to expose Plausible via HTTPS. Make sure to update the host to your domain.

For production certificates:
    
```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: plausible-ingress
      namespace: plausible
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-prod
    spec:
      rules:
      - host: your-domain.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: plausible
                port:
                  number: 8000
      tls:
      - hosts:
        - your-domain.com
        secretName: plausible-tls
```

For staging certificates:

```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: plausible-ingress
      namespace: plausible
      annotations:
        cert-manager.io/cluster-issuer: letsencrypt-staging
    spec:
      rules:
      - host: your-domain.com
        http:
          paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: plausible
                port:
                  number: 8000
      tls:
      - hosts:
        - your-domain.com
        secretName: plausible-tls
    ```

    Apply the Ingress resource:
    ```sh
    kubectl apply -f ingress.yaml --namespace plausible
    ```

With these steps, Plausible Analytics should be up and running on your Civo Kubernetes cluster. You can now proceed to configure and verify your deployment.

## Configuring Plausible Analytics

## Deploying Plausible to Civo Kubernetes

## Verifying the Deployment

## Conclusion