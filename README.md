# OpenTelemetry & New Relic Integration with Flux CD

This project demonstrates a complete observability pipeline using OpenTelemetry (OTEL) and New Relic, orchestrated by Flux CD in a Kubernetes environment (Minikube). It simulates a real-world "office environment" where application code is hosted on an internal Git server (Gitea) and deployed via GitOps.

## Project Overview

*   **Observability**: OpenTelemetry Collector (Agent & Gateway) exporting metrics and logs to New Relic.
*   **GitOps**: Flux CD syncing application manifests from an internal git repository.
*   **Internal Git**: Gitea running inside the cluster to simulate a private corporate git server.
*   **Applications**: 3 Dummy Java Microservices (Spring Boot) exposing Prometheus metrics.
*   **Istio Service Mesh**: Sidecar injection enabled for `services` namespace, with metrics scraped by OTEL.
*   **Dashboards**: Production-grade New Relic dashboards for Kubernetes, Host, Flux CD, and Istio Mesh.

## Directory Structure

```
.
├── dashboards/          # New Relic Dashboard JSONs
├── flux/                # Flux CD configurations
│   ├── flux-sync.yaml   # GitRepository and Kustomization resources
│   └── gitea/           # Gitea deployment manifests (if not using Helm)
├── manifests/           # Kubernetes Manifests
│   ├── apps/            # Application manifests (Java Services)
│   └── otel/            # OpenTelemetry Collector & RBAC
└── README.md            # This file
```

## Prerequisites

*   [Minikube](https://minikube.sigs.k8s.io/docs/start/)
*   [Helm](https://helm.sh/docs/intro/install/)
*   [kubectl](https://kubernetes.io/docs/tasks/tools/)
*   New Relic License Key

## Setup Instructions

### 1. Start Minikube

```bash
minikube start --cpus 4 --memory 8192
```

### 2. Install Flux CD

```bash
helm repo add fluxcd-community https://fluxcd-community.github.io/helm-charts
helm repo update
helm install flux fluxcd-community/flux2 -n flux-system --create-namespace
```

### 3. Deploy Internal Git Server (Gitea)

To simulate an internal git server, we deploy Gitea in the `flux-system` namespace.

```bash
helm repo add gitea-charts https://dl.gitea.io/charts/
helm repo update
helm install gitea gitea-charts/gitea -n flux-system \
  --set gitea.admin.username=admin \
  --set gitea.admin.password=admin123 \
  --set gitea.config.database.DB_TYPE=sqlite3 \
  --set postgresql.enabled=false \
  --set mysql.enabled=false \
  --set redis.enabled=false \
  --set valkey.enabled=false \
  --set gitea.config.cache.ADAPTER=memory \
  --set gitea.config.server.DISABLE_SSH=true \
  --set gitea.config.repository.DEFAULT_BRANCH=main \
  --set persistence.enabled=false
```

Wait for Gitea to be ready:
```bash
kubectl rollout status deployment/gitea -n flux-system
```

### 4. Push Application Code to Gitea

Since Gitea is running inside the cluster, we need to push the application manifests (`manifests/apps/java-apps.yaml`) to it.

1.  **Port-forward Gitea**:
    ```bash
    kubectl port-forward svc/gitea-http -n flux-system 3000:3000 &
    ```

2.  **Create Repository & Push**:
    ```bash
    # Create a temporary local git repo
    mkdir -p /tmp/flux-repo && cd /tmp/flux-repo
    git init
    cp /path/to/project/manifests/apps/java-apps.yaml .
    git add .
    git commit -m "Initial commit"
    
    # Push to Gitea (using admin/admin123)
    git remote add origin http://admin:admin123@localhost:3000/admin/repo.git
    git push -u origin main
    ```

### 5. Configure Flux Sync

Apply the Flux configuration to sync from Gitea to the `services` namespace.

```bash
kubectl apply -f flux/flux-sync.yaml
```

Flux will now detect the changes in Gitea and deploy the 3 Java microservices to the `services` namespace.

### 6. Deploy OpenTelemetry Collector

1.  **Create Secret**:
    ```bash
    kubectl create secret generic newrelic-license-key \
      --from-literal=license-key=<YOUR_NR_LICENSE_KEY> \
      -n opentelemetry-operator-system
    ```

2.  **Deploy Collector**:
    ```bash
    kubectl apply -f manifests/otel/otel-rbac.yaml
    kubectl apply -f manifests/otel/otel-collector.yaml
    ```

### 7. Deploy Istio Service Mesh

1.  **Install Istio**:
    ```bash
    helm repo add istio https://istio-release.storage.googleapis.com/charts
    helm repo update
    helm install istio-base istio/base -n istio-system --create-namespace
    helm install istiod istio/istiod -n istio-system --wait
    ```

2.  **Enable Sidecar Injection**:
    ```bash
    kubectl label namespace services istio-injection=enabled
    ```
    *Note: The `java-apps.yaml` manifest has been updated to include this label and resource limits (`10m` CPU) to run on Minikube.*

3.  **Restart Apps**:
    ```bash
    kubectl rollout restart deployment -n services
    ```

## Verification

*   **Check Pods**: `kubectl get pods -n services`
*   **Check Flux**: `kubectl get gitrepositories -n flux-system`
*   **Check Logs**: `kubectl logs -l app=otel-agent -n opentelemetry-operator-system`
*   **New Relic**: Check "Kubernetes" and "Logs" in New Relic UI.

## Dashboards

Import the JSON files in `dashboards/` into New Relic to visualize:

1.  **Cluster Health** (`k8s_dashboard_adapted.json`)
2.  **Compute Resources** (`compute_resources_dashboard.json`)
3.  **Host Metrics** (`host_dashboard.json`)
4.  **Flux CD Overview** (`flux_dashboard.json`): Monitor reconciliation rates and controller health.
5.  **Istio Mesh Overview** (`istio_dashboard.json`): Monitor Mesh Traffic (RPS, Latency, Success Rate) and Sidecar Health.

## Latest Activity & Architecture Notes

For detailed architectural decisions regarding OpenTelemetry configuration (e.g., Port Discovery vs Service Discovery), please refer to [need_to_know.md](need_to_know.md).
