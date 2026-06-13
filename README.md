# fraud-detection

URL for Workshop Instructions: <https://rh-aiservices-bu.github.io/fraud-detection/>

## Notebook sequence

| Notebook | Purpose |
|---|---|
| `1_experiment_train.ipynb` | Train the Keras model, export to ONNX, evaluate |
| `2_save_model.ipynb` | Upload `models/fraud/1/model.onnx` to S3 |
| `2.1_register_experiment.ipynb` | Register experiment metadata in RHOAI Model Registry |
| `3_rest_requests.ipynb` | Test the deployed inference endpoint |

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
