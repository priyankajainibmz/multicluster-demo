# Multicluster Voting App Demo

End-to-end Tekton demo for the blog:
**"Unlocking Horizontal Scalability: Multicluster Tekton Pipelines on IBM Z s390x"**

## Architecture

```
┌─ HUB CLUSTER (cluster421 · s390x) ─────────────────────┐
│                                                         │
│  ┌──────────────────────────────────────────────────┐   │
│  │  Tekton Pipelines (runs entirely on hub)         │   │
│  │                                                  │   │
│  │  PipelineRun: deploy-voting-app-<uid>            │   │
│  │  └── Task: deploy-voting-app                     │   │
│  │       ├── step: login-to-spoke                   │   │
│  │       ├── step: create-namespace                 │   │
│  │       ├── step: cleanup-previous-deploy          │   │
│  │       ├── step: deploy-redis                     │   │
│  │       ├── step: deploy-vote                      │   │
│  │       ├── step: deploy-result                    │   │
│  │       ├── step: deploy-worker                    │   │
│  │       ├── step: wait-for-pods                    │   │
│  │       └── step: verify-and-report                │   │
│  └──────────────────────────────────────────────────┘   │
└─────────────────────────┬───────────────────────────────┘
                          │  oc login --token=...
                          │  oc apply -f <manifests>
                          ▼
┌─ SPOKE CLUSTER (cluster420 · s390x) ────────────────────┐
│                                                         │
│  Kubernetes target — no Tekton required                 │
│                                                         │
│  namespace: voting-app                                  │
│  ├── deployment/redis    (redis:7-alpine)               │
│  ├── deployment/vote     (python:3.9-alpine)            │
│  ├── deployment/result   (python:3.9-alpine)            │
│  └── deployment/worker   (python:3.9-alpine)            │
│                                                         │
│  Services:                                              │
│  ├── redis:6379  (ClusterIP)                            │
│  ├── vote:5000   (NodePort → 31000)                     │
│  └── result:5001 (NodePort → 31001)                     │
└─────────────────────────────────────────────────────────┘
```

**Key point:** Tekton task pods run on the hub. The spoke is purely a Kubernetes
API target — workloads land there via `oc apply`, not Tekton task execution.

---

## Files

| File | Purpose |
|------|---------|
| `01-spoke-credentials-secret.yaml` | Secret holding spoke API URL + token |
| `02-deploy-voting-app-task.yaml` | Tekton Task with 9 named steps |
| `03-deploy-voting-app-pipeline.yaml` | Pipeline wrapping the task |
| `04-pipelinerun.yaml` | PipelineRun (trigger with `oc create`) |

---

## Prerequisites

### On the hub cluster

- OpenShift Pipelines Operator installed (version 1.14+)
- `pipeline` service account in the `multicluster-demo` namespace
- The `oc` CLI image available at `image-registry.openshift-image-registry.svc:5000/openshift/cli:latest`

### On the spoke cluster

- A service account with permissions to create namespaces and deploy workloads
- A long-lived token for that service account

---

## Step-by-Step Setup

### 1. Log in to the hub cluster

```bash
oc login --token=<HUB_TOKEN> --server=<HUB_API_URL>
```

### 2. Create the demo namespace

```bash
oc new-project multicluster-demo
```

### 3. Create the spoke credentials Secret

Edit `01-spoke-credentials-secret.yaml` and replace the placeholder values:

```yaml
spoke-server: "https://api.YOUR-SPOKE-CLUSTER.example.com:6443"
spoke-token:  "sha256~REPLACE_WITH_REAL_TOKEN"
```

Get the token from the spoke cluster:

```bash
# On the spoke cluster:
oc login --server=<SPOKE_API_URL>
oc create serviceaccount deployer -n voting-app --dry-run=client -o yaml | oc apply -f -
oc adm policy add-role-to-user admin -z deployer -n voting-app
oc create token deployer -n voting-app --duration=8760h
```

Then apply the secret on the hub:

```bash
oc apply -f 01-spoke-credentials-secret.yaml -n multicluster-demo
```

### 4. Apply the Task and Pipeline definitions

```bash
oc apply -f 02-deploy-voting-app-task.yaml     -n multicluster-demo
oc apply -f 03-deploy-voting-app-pipeline.yaml -n multicluster-demo
```

### 5. Trigger a PipelineRun

```bash
# oc create (not apply) because generateName is used
oc create -f 04-pipelinerun.yaml -n multicluster-demo
```

### 6. Watch it run

```bash
# Follow logs in real time
tkn pipelinerun logs --last -f -n multicluster-demo

# Or watch step-by-step in the OpenShift Console:
# Developer → Pipelines → multicluster-demo → PipelineRuns
```

### 7. Verify the deployment on the spoke cluster

```bash
# Switch to the spoke cluster
oc login --token=<SPOKE_TOKEN> --server=<SPOKE_API_URL>

oc get deployments -n voting-app
oc get pods       -n voting-app
oc get services   -n voting-app

# Get the node IP
NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
echo "Vote:   http://${NODE_IP}:31000"
echo "Result: http://${NODE_IP}:31001"
```

---

## Expected Output

After a successful PipelineRun, `oc get deployments -n voting-app` on the spoke shows:

```
NAME     READY   UP-TO-DATE   AVAILABLE   AGE
redis    1/1     1            1           ~5m
vote     1/1     1            1           ~4m
result   1/1     1            1           ~4m
worker   1/1     1            1           ~3m
```

The PipelineRun emits three results:
- `deployment-status` — e.g. `SUCCESS: 4/4 pods running`
- `vote-url` — e.g. `http://10.x.x.x:31000`
- `result-url` — e.g. `http://10.x.x.x:31001`

---

## Customising Images (s390x Compatibility)

All default images are verified multi-arch (including s390x):

| Component | Default Image | Notes |
|-----------|--------------|-------|
| Redis | `redis:7-alpine` | Official multi-arch |
| Vote | `python:3.9-alpine` | Official multi-arch |
| Result | `python:3.9-alpine` | Official multi-arch |
| Worker | `python:3.9-alpine` | Official multi-arch |

To use your own images, override the params in the PipelineRun:

```yaml
params:
  - name: vote-image
    value: "quay.io/your-org/voting-app-vote:s390x"
```

---

## Re-running the Demo

```bash
# Each run creates a new PipelineRun with a unique name
oc create -f 04-pipelinerun.yaml -n multicluster-demo

# Clean up all runs
tkn pipelinerun delete --all -n multicluster-demo
```

---

## Security Notes

- **Never commit real tokens to git.** The `01-spoke-credentials-secret.yaml` file
  contains placeholder values — populate it and apply it, but do not commit it.
- For production, use short-lived tokens (1h) and rotate them via a CronJob or
  External Secrets Operator.
- The `pipeline` service account on the hub only needs permission to read the
  `spoke-cluster-credentials` Secret, not cluster-admin.
