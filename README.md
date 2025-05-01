# n8n on Kubernetes (local) with PostgreSQL & NFS  
[中文版 README](./README_zh-TW.md)

> Self-hosted n8n 1.x + Bitnami PostgreSQL 17 + Valkey (Redis-compatible) on a local K8s cluster  
> StorageClass =`nfs-client` (RWX) · Queue mode · worker + runner scaling

---

### Why use this template?

* **Queue mode** & horizontal scaling out-of-the-box.  
* **RWX** dynamic PVCs via *nfs-subdir-external-provisioner*.  
* **Service-link issue fixed** – `fullnameOverride` removes the `N8N_PORT=tcp://…` env clash.  
* Turn-key Helm values (Postgres, n8n, Valkey) plus troubleshooting notes.

---

## 0 · Prerequisites

| Component | Minimum version |
|-----------|-----------------|
| Kubernetes | v1.25 (guide tested on v1.29) |
| Helm | v3.8 (OCI registry support) |
| StorageClass | `nfs-client` (RWX) |
| CLI tools | `kubectl`, `openssl`, (Optional `jq`) |
| Outbound | Access to `8gears.container-registry.com`, `registry-1.docker.io` |

---

## 1 · Namespace

```bash
kubectl create namespace n8n
```

---

## 2 · One-shot secrets (do **not** commit them)

```bash
export POSTGRES_PASSWORD=$(openssl rand -base64 20)
export N8N_ENCRYPTION_KEY=$(openssl rand -base64 32 | tr -d '\n')
export VALKEY_PASSWORD=$(openssl rand -base64 10 | tr -d '=+/')
```

---

## 3 · `pg-values.yaml`

```bash
cat <<EOF > pg-values.yaml
fullnameOverride: postgresql
auth:
  postgresPassword: "${POSTGRES_PASSWORD}"
  database: n8n
primary:
  persistence:
    storageClass: nfs-client
    size: 5Gi
EOF
```

---

## 4 · `n8n-values.yaml`

```bash
cat <<EOF > n8n-values.yaml
fullnameOverride: n8n-main

main:
  extraEnv:
    - {name: N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS, value: "true"}
    - {name: N8N_RUNNERS_ENABLED, value: "true"}
    - {name: OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS, value: "true"}
    - {name: QUEUE_BULL_REDIS_HOST, value: n8n-valkey-primary}
    - {name: QUEUE_BULL_REDIS_PORT, value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "${VALKEY_PASSWORD}"}
    - {name: N8N_SECURE_COOKIE, value: "false"}
  config:
    db:
      type: postgresdb
      postgresdb:
        host: postgresql-hl
        port: 5432
        user: postgres
        database: n8n
    n8n:
      executions_mode: queue
  secret:
    db:
      postgresdb:
        password: ${POSTGRES_PASSWORD}
    n8n:
      encryption_key: ${N8N_ENCRYPTION_KEY}
  persistence:
    enabled: true
    type: dynamic
    storageClass: nfs-client
    accessModes: [ReadWriteMany]
    size: 5Gi
  service:
    type: NodePort
    port: 5678

worker:
  enabled: true
  livenessProbe: {enabled: false}
  readinessProbe: {enabled: false}
  extraEnv:
    - {name: QUEUE_BULL_REDIS_HOST, value: n8n-valkey-primary}
    - {name: QUEUE_BULL_REDIS_PORT, value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "${VALKEY_PASSWORD}"}
    - {name: N8N_SECURE_COOKIE, value: "false"}
  persistence:
    enabled: true
    type: dynamic
    storageClass: nfs-client
    accessModes: [ReadWriteMany]
    size: 1Gi
  resources:
    requests: {memory: 256Mi}
    limits:   {memory: 1Gi}

runner:
  enabled: true
  replicas: 1
  livenessProbe: {enabled: false}
  readinessProbe: {enabled: false}
  extraEnv:
    - {name: QUEUE_BULL_REDIS_HOST, value: n8n-valkey-primary}
    - {name: QUEUE_BULL_REDIS_PORT, value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "${VALKEY_PASSWORD}"}
    - {name: N8N_SECURE_COOKIE, value: "false"}
  persistence:
    enabled: true
    type: dynamic
    storageClass: nfs-client
    accessModes: [ReadWriteMany]
    size: 1Gi
  resources:
    requests: {memory: 256Mi}
    limits:   {memory: 1Gi}

valkey:
  enabled: true
  auth:
    enabled: true
    password: "${VALKEY_PASSWORD}"
EOF
```

---

## 5 · Install

```bash
# PostgreSQL
helm repo add bitnami https://charts.bitnami.com/bitnami
helm install postgresql bitnami/postgresql -n n8n -f pg-values.yaml
kubectl -n n8n rollout status deploy/postgresql

# n8n 1.0.6
helm registry login 8gears.container-registry.com -u anonymous -p ""
helm install n8n oci://8gears.container-registry.com/library/n8n \
  --version 1.0.6 -n n8n -f n8n-values.yaml
kubectl -n n8n rollout status deploy/n8n-main
```

---

## 6 · Open the UI

```bash
NODEPORT=$(kubectl -n n8n get svc n8n-main -o jsonpath='{.spec.ports[0].nodePort}')
NODEIP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "http://$NODEIP:$NODEPORT"
```

---

## 7 · Upgrades

```bash
helm upgrade n8n oci://8gears.container-registry.com/library/n8n \
  --version 1.0.7 -n n8n -f n8n-values.yaml

helm upgrade postgresql bitnami/postgresql -n n8n -f pg-values.yaml
```

---

## 8 · Quick FAQ

| Symptom | Fix |
|---------|-----|
| `Invalid number value for N8N_PORT` | keep `fullnameOverride: n8n-main` or set `enableServiceLinks: false` |
| `password authentication failed` | PostgreSQL already initialised – delete PVC or enable `passwordUpdateJob` |
| `NOAUTH Authentication required` | Valkey password not in n8n env – ensure all pods share the same `${VALKEY_PASSWORD}` |
| Worker keeps restarting | disable HTTP probes (see values above) or switch to `tcpSocket` |

---

## 9 · Cleanup

```bash
helm uninstall n8n postgresql -n n8n
kubectl -n n8n delete pvc -l app.kubernetes.io/instance in (n8n,postgresql)
kubectl delete ns n8n
```

---

## 10 · Extras

* **Ingress + TLS** – sample snippet in the Chinese README.  
* **Single-Pod demo** – remove `executions_mode`, disable `worker` & `valkey`.  
* **SQLite quick test** – set `config.db.type: sqlite`; PVC may stay on `nfs-client`.

Happy automating 🚀

---

### Quick tips to generate the two values files

```bash
chmod +x ./gen-values.sh
./gen-values.sh
```

Create a small shell script with the exact blocks shown in sections 3 & 4; it will drop `pg-values.yaml` and `n8n-values.yaml` next to your chart.