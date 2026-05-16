# OpenSearch Setup

Run all queries below in **OpenSearch Dashboards → Dev Tools** (`http://localhost:5601/app/dev_tools`).

---

## 1. Start the cluster

```bash
docker compose -f docker-compose.yml up -d
```

Wait ~30s for the cluster to be ready, then verify:

```
GET /
```

---

## 2. Apply cluster settings

Cluster settings for ML Commons (trusted endpoints, remote inference, etc.). Run once after the cluster starts.

```
PUT /_cluster/settings
{
  "persistent": {
    "plugins.ml_commons.only_run_on_ml_node": true,
    "plugins.ml_commons.task_dispatch_policy": "least_load",
    "plugins.ml_commons.max_ml_task_per_node": 10,
    "plugins.ml_commons.max_model_on_node": 10,
    "plugins.ml_commons.connector_access_control_enabled": true,
    "plugins.ml_commons.remote_inference.enabled": true,
    "plugins.ml_commons.agent_framework_enabled": true,
    "plugins.ml_commons.trusted_connector_endpoints_regex": [
      "^https://api\\.openai\\.com/.*$",
      "^https://api\\.cohere\\.ai/.*$",
      "^https://bedrock-runtime\\..*[a-z0-9-]\\.amazonaws\\.com/.*$",
      "^https://runtime\\.sagemaker\\..*[a-z0-9-]\\.amazonaws\\.com/.*$"
    ]
  }
}
```

---

## 3. Create remote model connector

Replace `<OPENAI_API_KEY>` with your key.

```
POST /_plugins/_ml/connectors/_create
{
  "name": "OpenAI Embeddings Connector",
  "description": "Connector for OpenAI text-embedding-3-small",
  "version": 1,
  "protocol": "http",
  "parameters": {
    "model": "text-embedding-3-small"
  },
  "credential": {
    "openAI_key": "<OPENAI_API_KEY>"
  },
  "actions": [
    {
      "action_type": "predict",
      "method": "POST",
      "url": "https://api.openai.com/v1/embeddings",
      "headers": {
        "Authorization": "Bearer ${credential.openAI_key}"
      },
      "request_body": "{ \"input\": ${parameters.input}, \"model\": \"${parameters.model}\" }",
      "pre_process_function": "connector.pre_process.openai.embedding",
      "post_process_function": "connector.post_process.openai.embedding"
    }
  ]
}
```

Save the `connector_id` from the response.

---

## 4. Register and deploy the model

Replace `<connector_id>` with the value from step 3.

```
POST /_plugins/_ml/models/_register?deploy=true
{
  "name": "OpenAI text-embedding-3-small",
  "function_name": "remote",
  "description": "Remote OpenAI text embedding model",
  "connector_id": "<connector_id>"
}
```

This returns a `task_id`. Poll until status is `COMPLETED`:

```
GET /_plugins/_ml/tasks/<task_id>
```

Save the `model_id` from the completed task response.

Verify the model is deployed:

```
GET /_plugins/_ml/models/<model_id>
```

---

## 5. Create the search pipeline

> **This replaces running `src/scripts/createSearchPipeline.ts`.**
>
> The pipeline name must match the `OPENSEARCH_HYBRID_SEARCH_PIPELINE` env var in the app (default: `products-hybrid-pipeline`).
> Weights must match `HYBRID_LEXICAL_WEIGHT` (0.6) and `HYBRID_SEMANTIC_WEIGHT` (0.4).

```
PUT /_search/pipeline/products-hybrid-pipeline
{
  "description": "Hybrid score fusion for products search (lexical + semantic)",
  "phase_results_processors": [
    {
      "normalization-processor": {
        "normalization": {
          "technique": "min_max"
        },
        "combination": {
          "technique": "arithmetic_mean",
          "parameters": {
            "weights": [0.6, 0.4]
          }
        }
      }
    }
  ]
}
```

Verify:

```
GET /_search/pipeline/products-hybrid-pipeline
```

---

## Notes

- **Idempotent** — re-running the `PUT /_search/pipeline/...` safely updates the pipeline in place.
- **Survives restarts** — the pipeline is stored in the cluster, not in the app. No need to recreate it on deploy.
- **Per-environment** — use a different pipeline name per cluster (e.g. `products-hybrid-pipeline-dev`) and set `OPENSEARCH_HYBRID_SEARCH_PIPELINE` accordingly. All other steps are identical.
- **Weights changed?** — re-run step 5 only.
