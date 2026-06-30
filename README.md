# ArgoCD GitOps Engine for Multi-Cloud Control Plane

This directory constitutes the complete GitOps application manifest repository for our hybrid multi-cloud infrastructure. It orchestrates the declarative deployment of all database, remote display (VDI), load balancing, and monitoring components onto our cluster.

It uses the **App of Apps** pattern, where a single master root application manages and auto-syncs the individual sub-applications.

---

## 📂 Repository Directory Layout

*   **`apps/`**: Contains the ArgoCD Application definitions for all components.
    *   `root-app.yaml`: The master application tracking the `apps/` folder itself.
    *   `sealed-secrets.yaml`: Bitnami Sealed Secrets controller.
    *   `cnpg-operator.yaml`: CloudNativePG operator.
    *   `postgres-ha.yaml`: Core HA database cluster (primary, replicas, pooler).
    *   `haproxy-ingress.yaml`: Ingress controller Helm application (multi-source values).
    *   `kasmweb.yaml`: Remote browser container grid.
    *   `monitoring.yaml`: Prometheus/Grafana stack (multi-source values).
    *   `monitoring-resources.yaml`: Scrapers and alerts in the database namespace.
    *   `haproxy-monitor-resources.yaml`: ServiceMonitor for Ingress.
*   **`database/`**: Raw PostgreSQL HA and PgBouncer manifests, plus the `db-creds` `SealedSecret`.
*   **`monitoring/`**: Prometheus scrapers (PodMonitors), Alertmanager config, alerting rules, and the SMTP `SealedSecret`.
*   **`kasmweb/`**: Kasmweb browser grid statefulset, headless service, and ingress with session stickiness.
*   **`haproxy/`**: ServiceMonitor configuration for HAProxy Ingress controller metrics.
*   **`values/`**: Production Helm chart overrides for HAProxy Ingress and Prometheus.

---

## 🔒 Bitnami Sealed Secrets: Sealing Credentials

To prevent sensitive credentials (e.g. database password, SMTP authentication details) from being stored in Git in plain text, we use **Bitnami Sealed Secrets**. A controller running in the cluster uses asymmetric encryption to decrypt `SealedSecret` resources into standard `Secret` resources in the cluster.

### 1. Download the `kubeseal` CLI Tool
If `kubeseal` is not installed on your client machine, download it without root privileges:
```bash
# Create a local bin directory
mkdir -p $HOME/.local/bin

# Set the desired version
KUBESEAL_VERSION="0.27.1"

# Download the tarball
curl -L -o kubeseal.tar.gz "https://github.com/bitnami-labs/sealed-secrets/releases/download/v${KUBESEAL_VERSION}/kubeseal-${KUBESEAL_VERSION}-linux-amd64.tar.gz"

# Extract and install to local bin
tar -xvzf kubeseal.tar.gz kubeseal
mv kubeseal $HOME/.local/bin/kubeseal
chmod +x $HOME/.local/bin/kubeseal
export PATH="$HOME/.local/bin:$PATH"

# Cleanup
rm kubeseal.tar.gz
```

### 2. Encrypt Your Database Secret
Assuming the Sealed Secrets controller is running in your cluster under `kube-system` namespace:
```bash
# Create a temporary raw secret yaml file (DO NOT COMMIT THIS FILE TO GIT)
cat <<EOF > temp-db-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-creds
  namespace: database
type: Opaque
stringData:
  username: app_user
  password: "YOUR_PRODUCTION_DB_PASSWORD"
EOF

# Seal the secret using kubeseal
kubeseal --controller-name=sealed-secrets \
         --controller-namespace=kube-system \
         --format=yaml < temp-db-secret.yaml > database/sealed-secret.yaml

# Safely delete the temporary file
rm temp-db-secret.yaml
```

### 3. Encrypt Your SMTP Alerts Secret
```bash
# Create a temporary raw secret yaml file (DO NOT COMMIT THIS FILE TO GIT)
cat <<EOF > temp-smtp-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: alertmanager-smtp-secret
  namespace: database
type: Opaque
stringData:
  password: "YOUR_SMTP_APPLICATION_PASSWORD"
EOF

# Seal the secret using kubeseal
kubeseal --controller-name=sealed-secrets \
         --controller-namespace=kube-system \
         --format=yaml < temp-smtp-secret.yaml > monitoring/sealed-smtp-secret.yaml

# Safely delete the temporary file
rm temp-smtp-secret.yaml
```

---

## 🚀 Bootstrap & Deployment Steps

Once you are ready to deploy this stack onto your production hybrid cluster (AWS control plane + DO workers):

### Step 1: Push Changes to the Git Repository
Use `kubectl port-forward` to establish access to the internal Git server:
```bash
# Forward traffic from your workstation to the in-cluster Git server
kubectl port-forward -n gitops svc/git-server 9418:9418 &
PORT_FORWARD_PID=$!

# Add the remote and push
git remote add origin git://127.0.0.1/repo.git
git branch -M main
git push -u origin main

# Kill the port forward process
kill $PORT_FORWARD_PID
```

### Step 2: Bootstrap the ArgoCD Root Application
Apply the `root-app.yaml` manifest. ArgoCD will inspect the `apps/` directory and spin up the remaining components automatically:
```bash
kubectl apply -f apps/root-app.yaml
```

### Step 3: Monitor Sync Status
Check the status of all applications:
```bash
kubectl get applications -n argocd
```
All child applications (`sealed-secrets`, `cnpg-operator`, `postgres-ha`, `haproxy-ingress`, `kasmweb`, `prometheus-monitoring`, `monitoring-resources`, and `haproxy-monitor-resources`) will synchronize sequentially:
1. The namespaces and CRDs will be provisioned.
2. The Helm charts will deploy the controller workloads.
3. The raw manifests will mount database storage, start the replicas, connect PgBouncer, and load the metrics collectors.
