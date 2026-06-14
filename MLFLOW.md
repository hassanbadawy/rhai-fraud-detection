# MLflow on RHOAI 3.4 — Setup Reference

This document captures lessons learned deploying MLflow via the native RHOAI 3.4 operator. Use it as a reference for future clusters.

---

## TL;DR

| What | Value |
|---|---|
| Public URL | `https://rh-ai.apps.<cluster>/mlflow` |
| Internal URL | `https://mlflow.redhat-ods-applications.svc:8443` |
| CRD to use | `mlflow.opendatahub.io/v1 MLflow` (cluster-scoped) |
| Pod namespace | `redhat-ods-applications` (always, not your project namespace) |
| S3 secret namespace | `redhat-ods-applications` (copy your project secret there) |

---

## What NOT to do (traps)

### Wrong CRD: MLflowConfig
`mlflow.kubeflow.org/v1 MLflowConfig` exists on the cluster but **has no controller**. Creating it does nothing. The MLflow operator does not watch this resource. Do not use it.

### Missing `MLFLOW_S3_ENDPOINT_URL` in the notebook
The MLflow Python client uploads artifacts **directly to S3** using boto3. It does NOT inherit `AWS_S3_ENDPOINT` — it looks for its own variable `MLFLOW_S3_ENDPOINT_URL`. Without it, boto3 constructs virtual-hosted-style URLs like `models.s3.<region>.amazonaws.com` which can never reach a private cluster endpoint.

**Symptom:** `EndpointConnectionError: Could not connect to the endpoint URL: "https://models.s3.local.amazonaws.com/mlflow/..."`

**Fix** — add this to your MLflow setup cell before calling `mlflow.tensorflow.autolog()`:
```python
s3_endpoint = os.environ.get("AWS_S3_ENDPOINT")
if s3_endpoint:
    os.environ["MLFLOW_S3_ENDPOINT_URL"] = s3_endpoint
```

Or set `MLFLOW_S3_ENDPOINT_URL` as a workbench environment variable with the same value as `AWS_S3_ENDPOINT`.

---

### Wrong CRD: MLflow in a project namespace
`MLflow` is **cluster-scoped** — it has no namespace. Trying to create it in a project namespace fails.

### Stale operator image
If the operator pod hasn't been restarted since RHOAI was installed it may not respond to new CRs. Check `oc logs` to confirm it is reconciling.

---

## Prerequisites

1. MLflow operator enabled in the DataScienceCluster:

```yaml
spec:
  components:
    mlflowoperator:
      managementState: Managed
```

2. An S3-compatible data connection secret in your project namespace (e.g. `models`) with keys:
   - `AWS_ACCESS_KEY_ID`
   - `AWS_SECRET_ACCESS_KEY`
   - `AWS_S3_ENDPOINT`
   - `AWS_S3_BUCKET`
   - `AWS_DEFAULT_REGION`

---

## Installation Steps

### Step 1 — Copy S3 credentials to the operator namespace

The MLflow pod always runs in `redhat-ods-applications`. It cannot read secrets from your project namespace directly. Copy the secret:

```bash
oc get secret models -n fraud-detection -o json | \
  python3 -c "
import json, sys
s = json.load(sys.stdin)
s['metadata'] = {'name': 'mlflow-s3-credentials', 'namespace': 'redhat-ods-applications'}
s.pop('status', None)
print(json.dumps(s))
" | oc apply -f -
```

Repeat this step whenever the source secret is rotated.

### Step 2 — Create the MLflow CR

```yaml
apiVersion: mlflow.opendatahub.io/v1
kind: MLflow
metadata:
  name: mlflow          # must be exactly "mlflow"
spec:
  workspaceLabelSelector:
    matchLabels:
      kubernetes.io/metadata.name: fraud-detection   # namespace(s) that can use this server
  defaultArtifactRoot: s3://models/mlflow/            # bucket from your S3 secret
  serveArtifacts: true
  backendStoreUri: sqlite:////mlflow/mlflow.db        # SQLite on PVC (simple); swap for postgres URI if needed
  storage:
    accessModes:
      - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
  envFrom:
    - secretRef:
        name: mlflow-s3-credentials                  # S3 creds copied in Step 1
```

```bash
oc apply -f mlflow-cr.yaml
```

### Step 3 — Verify

```bash
oc get mlflow mlflow -o jsonpath='{.status.url}'
# → https://rh-ai.apps.<cluster>/mlflow

oc get pod -n redhat-ods-applications -l app=mlflow
# → mlflow-<hash>   2/2   Running
```

---

## Notebook Usage

Add `MLFLOW_TRACKING_URI` as a workbench environment variable:

```
MLFLOW_TRACKING_URI = https://rh-ai.apps.<cluster>/mlflow
```

Then in the notebook:

```python
import mlflow
import os

# Required: copy AWS_S3_ENDPOINT to MLFLOW_S3_ENDPOINT_URL so the MLflow
# artifact client reaches your cluster S3, not amazonaws.com
s3_endpoint = os.environ.get("AWS_S3_ENDPOINT")
if s3_endpoint:
    os.environ["MLFLOW_S3_ENDPOINT_URL"] = s3_endpoint

mlflow.set_tracking_uri(os.environ["MLFLOW_TRACKING_URI"])
mlflow.set_experiment("fraud-detection")

# Keras/TF autologging — logs params, metrics, and registers the model
mlflow.tensorflow.autolog(registered_model_name="fraud-detection")

with mlflow.start_run():
    history = model.fit(...)

    # Log custom metrics manually if needed
    mlflow.log_metric("precision", precision)
    mlflow.log_metric("recall", recall)

    # Log ONNX model explicitly
    mlflow.onnx.log_model(model_proto, "onnx-model")
```

### Authentication

The RHOAI-native MLflow server sits behind the RHOAI dashboard route. From inside the cluster (workbench notebooks), no extra auth is needed — the internal service URL works without a token:

```python
mlflow.set_tracking_uri("https://mlflow.redhat-ods-applications.svc:8443")
```

If calling from outside the cluster (CI, local machine), use the public route and pass your OCP token:

```python
import os
os.environ["MLFLOW_TRACKING_TOKEN"] = "<oc whoami -t output>"
mlflow.set_tracking_uri("https://rh-ai.apps.<cluster>/mlflow")
```

---

## Architecture Notes

- One `MLflow` CR → one MLflow server → always deployed in `redhat-ods-applications`
- `workspaceLabelSelector` controls which namespace labels see the server in the RHOAI dashboard (cosmetic), not actual network access
- SQLite backend is fine for a single-team demo; use PostgreSQL (Crunchy Postgres operator) for production multi-user deployments
- Artifacts land in `s3://<AWS_S3_BUCKET>/mlflow/` — separate prefix from model serving artifacts
- The `MLflowConfig` (`mlflow.kubeflow.org/v1`) and the per-namespace `mlflow-artifact-connection` secret are unused by the RHOAI 3.4 operator but may be required in future versions — leave them in place

---

## Teardown

```bash
oc delete mlflow mlflow
oc delete secret mlflow-s3-credentials -n redhat-ods-applications
```

The PVC for the SQLite backend is NOT deleted automatically — delete it manually if you want a clean slate:

```bash
oc delete pvc -n redhat-ods-applications -l app=mlflow
```
