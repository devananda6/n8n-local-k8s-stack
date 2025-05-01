## 專案重點

此專案示範在本機 K8s（含 `nfs-client` 動態 RWX PVC）部署 **n8n 1.x + PostgreSQL 17 + Valkey**，並解決常見三大痛點：

1. **Service-link 衝突** — 透過 `fullnameOverride` 避開 K8s 注入 `N8N_PORT=tcp://…` 造成主節點啟動失敗 citeturn0search1  
2. **PostgreSQL／Valkey 密碼同步** — 一次寫好 `auth.database` 與 `QUEUE_BULL_REDIS_PASSWORD`，避免 *password authentication failed* 與 *NOAUTH* 迴圈 citeturn0search2turn0search5  
3. **Worker 探針重啟** — Worker 無 HTTP 端點，關閉其 liveness／readinessProbe 或改用 tcpSocket 偵測即可 citeturn0search5turn0search7  

---

## 0　前置需求

| 元件 | 建議版本 |
|------|----------|
| Kubernetes | 1.25 以上（示例 1.29.10） |
| Helm | 3.8 以上 |
| StorageClass | `nfs-client`（RWX，可搭 nfs-subdir-external-provisioner ）citeturn0search7 |
| 網路出口 | 可存取 `8gears.container-registry.com`、`registry-1.docker.io` |

---

## 1　建立命名空間

```bash
kubectl create namespace n8n
```

---

## 2　一鍵產生機密（僅用一次、請勿 commit）

```bash
export POSTGRES_PASSWORD=$(openssl rand -base64 20)
export N8N_ENCRYPTION_KEY=$(openssl rand -base64 32 | tr -d '\n')
export VALKEY_PASSWORD=$(openssl rand -base64 10 | tr -d '=+/')
```

---

## 3　產出 `pg-values.yaml`

```bash
cat <<EOF > pg-values.yaml
fullnameOverride: postgresql
auth:
  postgresPassword: "$POSTGRES_PASSWORD"
  database: n8n
primary:
  persistence:
    storageClass: nfs-client
    size: 5Gi
EOF
```

---

## 4　產出 `n8n-values.yaml`

```bash
cat <<EOF > n8n-values.yaml
fullnameOverride: n8n-main

main:
  extraEnv:
    - {name: N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS, value: "true"}   # citeturn0search3
    - {name: N8N_RUNNERS_ENABLED, value: "true"}                     # citeturn0search6
    - {name: OFFLOAD_MANUAL_EXECUTIONS_TO_WORKERS, value: "true"}    # citeturn0search6
    - {name: QUEUE_BULL_REDIS_HOST, value: n8n-valkey-primary}       # citeturn0search4
    - {name: QUEUE_BULL_REDIS_PORT, value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "$VALKEY_PASSWORD"}
    - {name: N8N_SECURE_COOKIE, value: "false"}                      # citeturn0search8
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
        password: $POSTGRES_PASSWORD
    n8n:
      encryption_key: $N8N_ENCRYPTION_KEY
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
    - {name: QUEUE_BULL_REDIS_PORT,  value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "$VALKEY_PASSWORD"}
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
    - {name: QUEUE_BULL_REDIS_PORT,  value: "6379"}
    - {name: QUEUE_BULL_REDIS_PASSWORD, value: "$VALKEY_PASSWORD"}
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
    password: "$VALKEY_PASSWORD"
EOF
```

---

## 5　安裝

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

## 6　存取 UI

```bash
NODEPORT=$(kubectl -n n8n get svc n8n-main -o jsonpath='{.spec.ports[0].nodePort}')
NODEIP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "➡  http://$NODEIP:$NODEPORT"
```

---

## 7　升級

```bash
helm upgrade n8n oci://8gears.container-registry.com/library/n8n \
  --version 1.0.7 -n n8n -f n8n-values.yaml
helm upgrade postgresql bitnami/postgresql -n n8n -f pg-values.yaml
```

---

## 8　常見問題

| 現象 | 解法 |
|------|------|
| `Invalid number value for N8N_PORT` | `fullnameOverride: n8n-main` 或 `enableServiceLinks: false` citeturn0search9 |
| `password authentication failed` | Postgres 舊密碼仍在 PVC → 刪 PVC 或啟用 `passwordUpdateJob` citeturn0search2 |
| `NOAUTH Authentication required` | 三處 `QUEUE_BULL_REDIS_PASSWORD` 與 `valkey.auth.password` 同步 citeturn0search5 |
| Worker 重啟 | 關閉 Probe 或改 tcpSocket；本檔已關閉 citeturn0search5 |
| Secure-cookie 警告 | 內網設 `N8N_SECURE_COOKIE=false` 或上 HTTPS citeturn0search8 |

---

## 9　一鍵清除

```bash
helm uninstall n8n postgresql -n n8n
kubectl -n n8n delete pvc -l app.kubernetes.io/instance in (n8n,postgresql)
kubectl delete ns n8n
```

---

## 10　延伸

* **Ingress + TLS**  
  ```yaml
  ingress:
    enabled: true
    className: nginx
    hosts: [n8n.local]
    tls:
      - secretName: n8n-tls
        hosts: [n8n.local]
  ```
* **單 Pod 測試**：刪除 `executions_mode`、`worker.enabled`、`valkey.enabled`。  
* **SQLite 模式**：`config.db.type: sqlite`，PVC 仍可用 `nfs-client`。  

---

### 使用方式

1. 執行 **章 2** 指令產生兩份 values 檔。  
2. 跑 **章 5** 安裝步驟，即可獲得可水平擴充的 n8n 叢集。  
3. 如需開源分享，直接貼此 README，或用 Sealed-Secrets 取代純文字密碼。  

祝自動化愉快！🚀