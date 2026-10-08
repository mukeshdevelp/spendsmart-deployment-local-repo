# SpendSmart Local Kubernetes Deployment Guide

Exact steps to deploy SpendSmart (Backend, Frontend, Engine) with ClickHouse on Minikube using Helm.

> **Security:** Never commit Harbor credentials, ClickHouse passwords, encryption keys, API keys or Keycloak secrets to Git. Replace every `<PLACEHOLDER>` locally.

---

## 1. Prerequisites

```bash
docker --version
kubectl version --client
helm version
minikube version
```

Start Minikube and verify the node is `Ready`:

```bash
minikube status || minikube start
kubectl get nodes
```

---

## 2. Login to Harbor

Registry: `harbor.ldc.opstree.dev`

```bash
docker login harbor.ldc.opstree.dev
```

Enter the Harbor username and password / robot secret when prompted.

---

## 3. Pull the Application Images

```bash
docker pull harbor.ldc.opstree.dev/coe/uniteconpro-backend-prod:latest
docker pull harbor.ldc.opstree.dev/coe/uniteconpro-frontend-prod:latest
docker pull harbor.ldc.opstree.dev/coe/uniteconpro-engine-dev:latest
docker images | grep uniteconpro
```

> The local deployment uses the **dev** Engine image because `coe/uniteconpro-engine-prod:latest` is not available in Harbor.

---

## 4. Go to the Helm Repository

```bash
cd ~/clickhouse/helm-charts
```

Expected structure:

```text
helm-charts/
├── charts/
│   └── spendsmart/
└── client-a-file.yml
```

---

## 5. Create the Namespace

```bash
kubectl create namespace spendsmart
kubectl get namespace spendsmart
```

("already exists" is fine.)

---

## 6. Create the Harbor Image Pull Secret

The chart expects the secret name `harbor-ldc-coe-regcred`.

```bash
kubectl create secret docker-registry harbor-ldc-coe-regcred \
  --docker-server=harbor.ldc.opstree.dev \
  --docker-username='<HARBOR_USERNAME>' \
  --docker-password='<HARBOR_PASSWORD_OR_ROBOT_SECRET>' \
  -n spendsmart

kubectl get secret harbor-ldc-coe-regcred -n spendsmart
```

---

## 7. Deploy ClickHouse

**`clickhouse-deployment.yaml`**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: clickhouse
  namespace: spendsmart
spec:
  replicas: 1
  selector:
    matchLabels:
      app: clickhouse
  template:
    metadata:
      labels:
        app: clickhouse
    spec:
      containers:
        - name: clickhouse
          image: clickhouse/clickhouse-server:latest
          ports:
            - containerPort: 8123
            - containerPort: 9000
```

**`clickhouse-service.yaml`**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: clickhouse
  namespace: spendsmart
spec:
  selector:
    app: clickhouse
  ports:
    - name: http
      port: 8123
      targetPort: 8123
    - name: native
      port: 9000
      targetPort: 9000
```

Apply and check:

```bash
kubectl apply -f clickhouse-deployment.yaml
kubectl apply -f clickhouse-service.yaml

kubectl get pods -n spendsmart
kubectl get svc -n spendsmart
kubectl get endpoints clickhouse -n spendsmart
```

Wait until the ClickHouse pod is `Running` before continuing.

---

## 8. Verify ClickHouse

```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SELECT currentUser()"
# Expected: default

kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW DATABASES"
```

---

## 9. Create the `cost` Database

```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "CREATE DATABASE IF NOT EXISTS cost"

kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW DATABASES"
```

`cost` must appear in the output.

---

## 10. Create the `dba` User

```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query \
  "CREATE USER IF NOT EXISTS dba IDENTIFIED WITH plaintext_password BY '<YOUR_CLICKHOUSE_PASSWORD>'"

kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "GRANT ALL ON cost.* TO dba"

kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW USERS"
```

Expected users: `dba`, `default`.

---

## 11. Check Tables

```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW TABLES FROM cost"
```

The application expects `cost.service_costs`. The Engine creates `vm_utilization_summary` when that functionality runs.

> Do **not** invent the `service_costs` schema. It must match the application's ingestion/data model.

---

## 12. Create the Backend ClickHouse Secret (`clickhouse`)

```bash
kubectl create secret generic clickhouse \
  --from-literal=CLICKHOUSE_HOST=clickhouse \
  --from-literal=CLICKHOUSE_PORT=8123 \
  --from-literal=CLICKHOUSE_USER=dba \
  --from-literal=CLICKHOUSE_PASSWORD='<YOUR_CLICKHOUSE_PASSWORD>' \
  --from-literal=CLICKHOUSE_DATABASE=cost \
  --from-literal=CLICKHOUSE_TABLE=service_costs \
  --from-literal=CLICKHOUSE_TLS=false \
  -n spendsmart

kubectl get secret clickhouse -n spendsmart   # DATA = 7
```

---

## 13. Create the Engine ClickHouse Secret (`clickhouse-db`)

```bash
kubectl create secret generic clickhouse-db \
  --from-literal=CLICKHOUSE_HOST=clickhouse \
  --from-literal=CLICKHOUSE_PORT=8123 \
  --from-literal=CLICKHOUSE_USER=dba \
  --from-literal=CLICKHOUSE_PASSWORD='<YOUR_CLICKHOUSE_PASSWORD>' \
  --from-literal=CLICKHOUSE_DATABASE=cost \
  --from-literal=CLICKHOUSE_TABLE=service_costs \
  -n spendsmart

kubectl get secret clickhouse-db -n spendsmart   # DATA = 6
```

---

## 14. Create the Encryption Secret (`encryption-key`)

Use the approved key name/value provided by the application team:

```bash
kubectl create secret generic encryption-key \
  --from-literal=<KEY_NAME>=<KEY_VALUE> \
  -n spendsmart

kubectl get secret encryption-key -n spendsmart
```

---

## 15. Verify All Required Secrets

```bash
kubectl get secrets -n spendsmart
```

Must include: `harbor-ldc-coe-regcred`, `encryption-key`, `clickhouse`, `clickhouse-db`.

---

## 16. Configure `client-a-file.yml`

File: `~/clickhouse/helm-charts/client-a-file.yml`

```yaml
backend:
  image:
    repository: harbor.ldc.opstree.dev/coe/uniteconpro-backend-prod
    tag: "latest"

frontend:
  image:
    repository: harbor.ldc.opstree.dev/coe/uniteconpro-frontend-prod
    tag: "latest"

  configMap:
    data:
      VITE_API_BASE_URL: ""
      VITE_CLOUD_SENTRY_API_BASE_URL: ""
      VITE_KEYCLOAK_CLIENT_ID: ""
      VITE_KEYCLOAK_REALM: ""
      VITE_KEYCLOAK_URL: ""

cloudsentry:
  enabled: false

engine:
  enabled: true
  image:
    repository: harbor.ldc.opstree.dev/coe/uniteconpro-engine-dev
    tag: "latest"
```

The image block must use `repository:` and `tag:` keys (not `image: name`).

---

## 17. Lint the Chart

```bash
cd ~/clickhouse/helm-charts
helm lint ./charts/spendsmart -f ./client-a-file.yml
```

Expected: `1 chart(s) linted, 0 chart(s) failed`. Fix any errors before continuing.

---

## 18. Render the Manifests (Dry Run)

```bash
helm template spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f ./client-a-file.yml \
  > /tmp/spendsmart-rendered.yaml

grep -nE '^kind:|^  name:|image:' /tmp/spendsmart-rendered.yaml
grep -nE "encryption-key|clickhouse-db|clickhouse|harbor-ldc-coe-regcred" /tmp/spendsmart-rendered.yaml
```

Expect: Backend Deployment, Frontend Deployment, Engine CronJob, Services, ServiceAccounts, ConfigMaps. Cloud Sentry must be absent.

---

## 19. Install the Chart

```bash
helm upgrade --install spendsmart ./charts/spendsmart \
  -n spendsmart \
  -f ./client-a-file.yml

helm list -n spendsmart
```

---

## 20. Verify the Deployment

```bash
kubectl get pods -n spendsmart
kubectl get deploy -n spendsmart
kubectl get svc -n spendsmart
kubectl get cronjob -n spendsmart
```

Expect `spendsmart-backend-*` and `spendsmart-frontend-*` pods `Running`. The Engine is a CronJob, so it has no continuously running pod.

Logs:

```bash
kubectl logs -n spendsmart deploy/spendsmart-backend
kubectl logs -n spendsmart deploy/spendsmart-frontend
```

If a deployment name differs, use `kubectl get pods -n spendsmart` and `kubectl logs -n spendsmart <POD_NAME>`.

---

## 21. Test the Engine Manually

```bash
kubectl create job \
  --from=cronjob/spendsmart-engine \
  spendsmart-engine-test \
  -n spendsmart

kubectl get jobs -n spendsmart
kubectl logs -n spendsmart job/spendsmart-engine-test
```

---

## 22. Verify ClickHouse After Startup

```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW DATABASES"

kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW TABLES FROM cost"
```

---

## 23. Helm Release Inspection

```bash
helm status spendsmart -n spendsmart
helm get values spendsmart -n spendsmart
helm get manifest spendsmart -n spendsmart
```

---

## 24. Troubleshooting

### ImagePullBackOff
```bash
kubectl describe pod -n spendsmart <POD_NAME>
kubectl get secret harbor-ldc-coe-regcred -n spendsmart
kubectl get pod -n spendsmart <POD_NAME> -o jsonpath='{.spec.imagePullSecrets}'
```
Look for `pull access denied`, `unauthorized`, `i/o timeout`.

### CreateContainerConfigError
Usually a missing Secret or ConfigMap.
```bash
kubectl describe pod -n spendsmart <POD_NAME>
kubectl get secrets -n spendsmart
kubectl get configmaps -n spendsmart
```

### Backend cannot connect to ClickHouse
```bash
kubectl get svc clickhouse -n spendsmart
kubectl get endpoints clickhouse -n spendsmart
kubectl get pods -n spendsmart -l app=clickhouse
kubectl exec -n spendsmart <BACKEND_POD_NAME> -- getent hosts clickhouse
```
Host must be `clickhouse`, port `8123`.

### Database `cost` missing
```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "CREATE DATABASE IF NOT EXISTS cost"
```

### User `dba` missing
Re-run Section 10.

### `Table cost.service_costs doesn't exist`
```bash
kubectl exec -n spendsmart deploy/clickhouse -- \
  clickhouse-client --query "SHOW TABLES FROM cost"
```
Do not create an arbitrary schema. Inspect the application's migration/schema source or the component that loads billing data.

### Engine image not found
Keep the dev image for local testing:
```yaml
engine:
  image:
    repository: harbor.ldc.opstree.dev/coe/uniteconpro-engine-dev
    tag: "latest"
```

---

## 25. Useful Daily Commands

```bash
kubectl get all -n spendsmart
kubectl get pods -n spendsmart -o wide
kubectl get svc,secrets,deploy,cronjob,jobs -n spendsmart
helm status spendsmart -n spendsmart
kubectl logs -n spendsmart deploy/spendsmart-backend -f
kubectl logs -n spendsmart deploy/spendsmart-frontend -f
```

---

## 26. Clean Redeployment

Remove only the Helm release:

```bash
helm uninstall spendsmart -n spendsmart
kubectl get all -n spendsmart
```

The manually created ClickHouse Deployment/Service and Secrets remain.

Delete everything (**irreversible**: removes all resources and Secrets in the namespace):

```bash
kubectl delete namespace spendsmart
```

---

## 27. Deployment Order Summary

1. Start Minikube
2. Login to Harbor
3. Pull/verify images
4. Create `spendsmart` namespace
5. Create Harbor pull secret
6. Deploy ClickHouse Deployment + Service
7. Create `cost` database
8. Create `dba` user and grant `cost.*`
9. Create `clickhouse`, `clickhouse-db`, `encryption-key` Secrets
10. Configure `client-a-file.yml`
11. `helm lint` → `helm template` → `helm upgrade --install`
12. Verify Backend and Frontend
13. Run Engine test Job
14. Check logs and ClickHouse tables

---

## 28. Architecture

```text
                    Minikube
                       │
               ┌───────┴────────┐
               │  spendsmart ns │
               └───────┬────────┘
        ┌──────────────┼──────────────┐
        ▼              ▼              ▼
    Backend        Frontend        Engine (CronJob)
        │                              │
        └──────────────┬───────────────┘
                       ▼
              ClickHouse Service → ClickHouse Pod
                   db: cost / table: service_costs
```

| Secret | Used by |
|---|---|
| `harbor-ldc-coe-regcred` | Backend, Frontend, Engine (image pull) |
| `clickhouse` | Backend |
| `clickhouse-db` | Engine |
| `encryption-key` | Backend, Engine |

---

## 29. Quick Start Checklist

- [ ] Minikube running
- [ ] Docker logged into Harbor
- [ ] Backend, Frontend, Engine images accessible
- [ ] `spendsmart` namespace exists
- [ ] Harbor pull secret exists
- [ ] ClickHouse pod running and Service exists
- [ ] `cost` database and `dba` user exist, `dba` has `cost.*`
- [ ] `clickhouse`, `clickhouse-db`, `encryption-key` Secrets exist
- [ ] `client-a-file.yml` configured
- [ ] `helm lint` and `helm template` succeed
- [ ] Helm release deployed
- [ ] Backend and Frontend `Running`
- [ ] Engine CronJob exists and test Job succeeds
- [ ] Required ClickHouse tables exist
