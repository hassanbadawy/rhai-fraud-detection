# fraud-detection

URL for Workshop Instructions: <https://rh-aiservices-bu.github.io/fraud-detection/>

## Notebook sequence

### Keras path (single-node, quick experiment)

| Notebook | Purpose |
|---|---|
| `1_tf_experiment_train.ipynb` | Train the Keras/TF model locally, export to ONNX, evaluate |
| `2_save_model.ipynb` | Upload `models/fraud/1/model.onnx` to S3 |
| `2.1_register_experiment.ipynb` | Register experiment metadata in RHOAI Model Registry |
| `3_rest_requests.ipynb` | Test the deployed inference endpoint |

### PyTorch distributed path (multi-node, production scale)

| Notebook | Purpose |
|---|---|
| `6_torch_distributed_train.ipynb` | Upload CSVs to S3, then run PyTorch DDP training across 2 nodes via KFTO — exports ONNX and uploads to S3 directly |
| `2.1_register_experiment.ipynb` | Register experiment metadata in RHOAI Model Registry |
| `3_rest_requests.ipynb` | Test the deployed inference endpoint |

> Both paths produce the same artifact (`s3://models/models/fraud/1/model.onnx`) and converge at `2.1` onward.

> **Note:** Ray/Codeflare distributed training was removed. The Codeflare SDK is deprecated as of RHOAI 3.4. Use `6_torch_distributed_train.ipynb` (KFTO) for distributed training instead.

### Supporting scripts & pipelines

| File | Purpose |
|---|---|
| `4-train-save.pipeline` | Elyra pipeline: `1_tf` → `2_save` → `2.1_register` (runs on Kubeflow Pipelines) |
| `5_kfp_pipeline.py` | KFP pipeline source — compile with `./5_kfp_pipeline.sh` |
| `5_kfp_pipeline.yaml` | Compiled KFP YAML: download data → train → upload to S3 |
| `6_torch_distributed_train.ipynb` | KFTO/PyTorch DDP distributed training across 2 nodes |
| `kfto-scripts/train_pytorch_cpu.py` | PyTorch DDP training script used by `6_torch_distributed_train.ipynb` |
| `5_kfp_pipeline.sh` | Installs KFP dependencies and compiles `5_kfp_pipeline.py` to YAML |

## Running `2.1_register_experiment.ipynb` — Model Registry setup

The RHOAI Model Registry REST API authenticates via an **OCP user token**, not the workbench service account token. You must supply it as an environment variable before running the notebook.

### 1. Get your OCP token

Run this from a terminal where you are logged into the cluster (e.g. your local machine, not the workbench terminal):

```bash
oc whoami -t
```

Copy the output — it looks like `sha256~...`.

### 2. Set the token in the workbench

In the RHOAI dashboard:

1. Go to **Workbenches** → find `fraud-detection-wb` → click **Edit**
2. Scroll to **Environment variables** → **Add variable**
3. Set `MODEL_REGISTRY_TOKEN` = `<paste the token from step 1>`
4. Save and restart the workbench

### 3. Run the notebook

With `MODEL_REGISTRY_TOKEN` set, run all cells in `2.1_register_experiment.ipynb`. The notebook connects to:

```
https://model-registry-rest.apps.cluster-7hb2t.7hb2t.sandbox670.opentlc.com
```

Override this with the `MODEL_REGISTRY_URL` environment variable if your cluster URL differs.

## Note: inference endpoint port mismatch (8080 vs 8888)

The RHOAI dashboard and the InferenceService `status.address.url` display the internal endpoint as:

```
http://fraud-predictor.fraud-detection.svc.cluster.local:8080
```

**This port is wrong.** The OVMS container is configured with `--rest_port=8888` and only listens on port **8888**. KServe falls back to `8080` in its status when the ServingRuntime does not explicitly declare `httpDataEndpoint`, causing the dashboard to show an incorrect URL.

Additionally, the `fraud-predictor` Kubernetes service is **headless** (`ClusterIP: None`), meaning there is no NAT or port translation — DNS resolves directly to the pod IP. The port in the URL must always be the actual container port.

**Use port `8888` in all notebooks and client code:**

```
http://fraud-predictor.fraud-detection.svc.cluster.local:8888
```

This is already set correctly in `3_rest_requests.ipynb`.
