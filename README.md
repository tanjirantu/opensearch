# OpenSearch Setup

End-to-end runbook for the hybrid product search cluster: start the cluster, deploy an
embedding model, provision the **index + ingest pipeline + search pipeline**, backfill from
the app, and cut the `products` alias over.

Steps 2 onward run in **OpenSearch Dashboards → Dev Tools** (`http://localhost:5601/app/dev_tools`)
unless a step says otherwise.

> **Alias vs. index.** The alias is always `products`. Concrete indices are versioned
> (`products_v3`, `products_v4`, …) and bumped on every mapping change — analysis settings are
> static on a live index, so a new analyzer means a new index plus a reindex. The examples below
> provision `products_v4` and cut the alias over from `products_v3`.

---

## 0. Create `.env`

`docker-compose.yml` reads these; compose fails to start without them.

```bash
cp .env.example .env
```

`.env.example` ships every value blank — **fill all four in before starting the cluster.**

| Var | Purpose |
| --- | --- |
| `OS_VER` | OpenSearch image tag (`3.6.0`) |
| `OPENSEARCH_INITIAL_ADMIN_PASSWORD` | Bootstraps the `admin` user on first start |
| `DASHBOARDS_USERNAME` / `DASHBOARDS_PASSWORD` | Credentials Dashboards uses to reach the cluster |

`OPENSEARCH_INITIAL_ADMIN_PASSWORD` must satisfy the 3.x demo-config password policy (min 8
chars, with upper, lower, digit and special). An empty or weak value makes the node exit during
bootstrap, and empty Dashboards credentials fail authentication.

`.env` is gitignored — never commit it.

---

## 1. Start the cluster

```bash
docker compose -f docker-compose.yml up -d
```

Three nodes come up: `opensearch-node1` / `opensearch-node2` (cluster_manager, data, ingest) and
`opensearch-ml1` (ML node — `plugins.ml_commons.only_run_on_ml_node=true`, so all model work
lands there). Wait ~30s for the healthchecks to pass:

```bash
docker compose ps
```

The cluster is TLS + security enabled, so it answers on **https** with a self-signed cert:

```bash
set -a; . ./.env; set +a          # compose reads .env; your shell does not
curl -k -u "admin:$OPENSEARCH_INITIAL_ADMIN_PASSWORD" https://localhost:9200
```

A `401` here usually means the variable was empty, not that the cluster is broken.

Dashboards itself is plain http: `http://localhost:5601`. Log in with the credentials from
step 0, then open Dev Tools. Confirm ML Commons is loaded:

```
GET /_cat/plugins?v
GET /_plugins/_ml/stats
```

---

## 2. Apply cluster settings

Cluster settings for ML Commons (trusted endpoints, remote inference, etc.). Run once after the
cluster starts.

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

`connector_access_control_enabled: true` means steps 3–4 must run as an identity carrying
`ml_full_access`. Do **not** grant that to the application runtime user — see
[Phase 1 §3](../docs/xpress-service/2026-06-09-opensearch-phase-1-cluster-and-model-setup.md)
for the two-identity split.

---

## 3. Create remote model connector

> **Dimension contract.** The model's output dimension must be **1536** — it must match
> `EMBEDDING_DIMENSION` in the app and the `embedding_vector` dimension in
> `workflow-template.json`. `text-embedding-3-small` is 1536. This cannot change in place once
> indexing has begun.

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

Save the `model_id` from the completed task response. Verify it is deployed and that inference
actually works:

```
GET /_plugins/_ml/models/<model_id>
POST /_plugins/_ml/models/<model_id>/_predict
{ "parameters": { "input": ["test embedding verification"] } }
```

Confirm `"model_state": "DEPLOYED"`, `shape: [1536]`, and a non-empty `data` array.
**If `_predict` fails, stop here — step 5 will provision an index that can never be populated.**

---

## 5. Provision the index pipeline

This is the main event: one Flow Framework template creates all three cluster-side resources
together — the **ingest pipeline** (`embedding_text → embedding_vector` at index time), the
**KNN index**, and the **hybrid search pipeline** (score fusion at query time).

### 5.1 Create the workflow

```
POST /_plugins/_flow_framework/workflow
```

Body: the entire contents of [`workflow-template.json`](./workflow-template.json). Save the
returned `workflow_id`.

### 5.2 Provision it

```
POST /_plugins/_flow_framework/workflow/<workflow_id>/_provision
{
  "model_id":           "<model_id from step 4>",
  "index_name":         "products_v4",
  "ingest_pipeline_id": "products-embedding-ingest",
  "search_pipeline_id": "products-hybrid-pipeline"
}
```

The names must match the app env vars in step 7 — `OPENSEARCH_PRODUCTS_TARGET_INDEX`,
`OPENSEARCH_INGEST_PIPELINE`, `OPENSEARCH_HYBRID_SEARCH_PIPELINE`.

### 5.3 Poll until complete

```
GET /_plugins/_flow_framework/workflow/<workflow_id>/_status
```

Repeat until `state: "COMPLETED"`. On `FAILED`, the `error` field names the failing node;
de-provision (`POST /_plugins/_flow_framework/workflow/<workflow_id>/_deprovision`) before
retrying, otherwise the half-created resources block the rerun.

---

## 6. Verify and smoke-test

```
GET /_ingest/pipeline/products-embedding-ingest
GET /_search/pipeline/products-hybrid-pipeline
GET /products_v4/_mapping
GET /products_v4/_settings
```

Confirm:

- the ingest pipeline has a `text_embedding` processor with `field_map: { embedding_text: embedding_vector }`;
- the search pipeline has a `normalization-processor` (`min_max` / `arithmetic_mean`, weights `[0.6, 0.4]`);
- the index has `index.knn: true`, `default_pipeline: products-embedding-ingest`, and the
  `sku_trigram` / `english_stem` analyzers under `analysis`;
- `embedding_vector` is a `knn_vector` of **dimension 1536** (`hnsw` / `lucene` / `cosinesimil`);
- `skus` is `text` with `analyzer: sku_trigram` and a `skus.keyword` subfield.

**Ingest smoke test** — proves OpenSearch generates the vector, not the app:

```
POST /products_v4/_doc/test-smoke
{ "name": "Smoke Test Product", "sellerUid": "test", "isActive": true, "embedding_text": "Classic cotton polo shirt" }

GET /products_v4/_doc/test-smoke
```

`embedding_vector` must be a non-empty 1536-element array. Then clean up:

```
DELETE /products_v4/_doc/test-smoke
```

**Hybrid smoke test:**

```
GET /products_v4/_search?search_pipeline=products-hybrid-pipeline
{
  "query": { "hybrid": { "queries": [
    { "match": { "name": "polo shirt" } },
    { "neural": { "embedding_vector": { "query_text": "polo shirt", "model_id": "<model_id>", "k": 5 } } }
  ] } }
}
```

Re-index the smoke doc first if you already deleted it; confirm `hits.hits` is non-empty.

---

## 7. Point the app at it

Set these in `xpress-service` and redeploy/restart:

```env
OPENSEARCH_NODE=https://localhost:9200
OPENSEARCH_USER=...
OPENSEARCH_PASSWORD=...
OPENSEARCH_SSL_VERIFY=false            # true once real certs are in place

OPENSEARCH_PRODUCTS_ALIAS=products
OPENSEARCH_PRODUCTS_ACTIVE_INDEX=products_v3     # currently behind the alias
OPENSEARCH_PRODUCTS_TARGET_INDEX=products_v4     # provisioned in step 5
OPENSEARCH_INGEST_PIPELINE=products-embedding-ingest

OPENSEARCH_HYBRID_ENABLED=true
OPENSEARCH_HYBRID_SEARCH_PIPELINE=products-hybrid-pipeline
OPENSEARCH_EMBEDDING_MODEL_ID=<model_id from step 4>
OPENSEARCH_HYBRID_K=100
HYBRID_LEXICAL_WEIGHT=0.6     # must match the provisioned search pipeline
HYBRID_SEMANTIC_WEIGHT=0.4    # must match the provisioned search pipeline
EMBEDDING_DIMENSION=1536      # must match the model + index knn_vector dimension

SEARCH_INDEXED_SELLER_UIDS=<comma-separated seller uids>
```

`SEARCH_INDEXED_SELLER_UIDS` is **fail-closed** — unset means nothing is indexed at all.

> **Alternative to step 5 — read the caveats first.** `yarn create:products-index products_v4`
> creates the index from the app's own mappings without Flow Framework. It does **not** create the
> ingest or search pipelines, it omits the explicit `method` on `embedding_vector` (see Notes),
> and — most importantly — **it does not set `default_pipeline` on the index.** The reindex sends
> no `pipeline` parameter on its bulk requests, so vectors are generated *only* by the index's
> `default_pipeline`. Creating the ingest pipeline separately is not enough: the entire step-8
> backfill will run, produce no `embedding_vector`, and then fail validation on
> `embeddingVectorPresent`. If you take this path, `PUT /products_v4/_settings` with
> `{"index.default_pipeline": "products-embedding-ingest"}` before backfilling. Prefer step 5.

---

## 8. Backfill

The app builds each document (`buildProductEmbeddingText()` plus brand and category lookups
exist only in the app), so use the app's reindex endpoint — **never** OpenSearch `/_reindex`.
It requires the `X-Gateway-Token` header. Default `batchSize` is 500 (`REINDEX_BATCH_SIZE`).

Dry run first:

```bash
curl -X POST "http://localhost:3015/api/v2/internal/products/reindex" \
  -H "Content-Type: application/json" \
  -H "x-gateway-token: <GATEWAY_SHARED_TOKEN>" \
  -d '{ "targetIndex": "products_v4", "dryRun": true }'
```

Confirm the estimated document count looks right, then run it:

```bash
curl -X POST "http://localhost:3015/api/v2/internal/products/reindex" \
  -H "Content-Type: application/json" \
  -H "x-gateway-token: <GATEWAY_SHARED_TOKEN>" \
  -d '{
    "targetIndex": "products_v4",
    "alias": "products",
    "batchSize": 200,
    "validateAfterIndex": true,
    "switchAliasAfterValidation": false
  }'
```

Confirm `errors: 0` and `validation.passed: true`, then spot-check a document — both
`embedding_text` and `embedding_vector` must be present:

```
GET /products_v4/_doc/<known_product_uid>
```

---

## 9. Cut the alias over

Either pass `switchAliasAfterValidation: true` on the reindex call, or switch by hand:

```
POST /_aliases
{ "actions": [
  { "remove": { "index": "products_v3", "alias": "products" } },
  { "add":    { "index": "products_v4", "alias": "products" } }
] }

GET /_cat/aliases/products?v
```

Validate search through the app — this one is a `curl` against the service, not Dev Tools —
confirming `debug.mode === "hybrid"` in non-prod:

```bash
curl "http://localhost:3015/api/v2/products/search?q=polo+shirt&sellerUid=<seller_uid>"
```

Then set `OPENSEARCH_PRODUCTS_ACTIVE_INDEX=products_v4` for the next cycle, and after ~24h
stable, drop the old index:

```
DELETE /products_v3
```

---

## Notes

- **Mapping source of truth** is `xpress-service/src/services/product/search/repository/productMappings.ts`.
  The `create_index` node in `workflow-template.json` mirrors `productsIndexMappingV2` field for
  field; regenerate it whenever that file changes, or a workflow-provisioned index will not serve
  the current query builder.
- **Known divergence.** `productMappings.ts` declares `embedding_vector` with no `method`, so
  `yarn create:products-index` gets OpenSearch's defaults (faiss / l2) while this template pins
  `hnsw` / `lucene` / `cosinesimil`. The two provisioning paths therefore produce different vector
  spaces for the same index name. Prefer the workflow template until `productMappings.ts` pins the
  method too.
- **`number_of_replicas: 0`** is mirrored from `productMappings.ts` and is fine for local and
  single-node environments. Before an alias cutover makes an index the production search target,
  raise it (`PUT /products_v4/_settings {"index.number_of_replicas": 1}`) — with no replica, one
  data-node restart takes `products` red and search starts returning 503s.
- **Object blobs are dynamically mapped.** `artworks`, `artworkLayouts`, `variants`, `images` and
  `prices` are mapped `{"type": "object", "enabled": true}` to mirror `productMappings.ts`, even
  though `ProductIndexDoc` documents them as response-payload only. `enabled: true` means their
  subfields are dynamically mapped and indexed, so a backfill can hit `mapper_parsing_exception`
  when a later document's subfield type conflicts with the first document's, and the artwork trees
  count against `index.mapping.total_fields.limit`. Switching to `enabled: false` is the right fix
  but must land in `productMappings.ts` and here together, or the two provisioning paths diverge.
- **Weight-overwrite warning.** The reindex calls `ensureHybridPipelineExists()`, which PUTs the
  search pipeline using `HYBRID_LEXICAL_WEIGHT` / `HYBRID_SEMANTIC_WEIGHT`. If those don't match
  what step 5 provisioned, the reindex silently overwrites the pipeline. Keep them aligned.
- **Client-side embedding vars are legacy.** `EMBEDDING_ENABLED`, `EMBEDDING_MODEL`,
  `EMBEDDING_SERVICE_URL` predate cluster-side inference. Embeddings are generated in OpenSearch;
  leave `EMBEDDING_ENABLED` unset.
- **Per-environment.** Use distinct pipeline names per cluster (e.g. `products-hybrid-pipeline-dev`)
  and set `OPENSEARCH_HYBRID_SEARCH_PIPELINE` accordingly. Every other step is identical.

### Updating the search pipeline alone

To retune weights without re-running the whole workflow, PUT the search pipeline directly — it is
idempotent and stored in the cluster, so it survives restarts and needs no recreation on deploy.
This is what `yarn create:search-pipeline` (`src/scripts/createSearchPipeline.ts`) does.

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

GET /_search/pipeline/products-hybrid-pipeline
```

Keep `HYBRID_LEXICAL_WEIGHT` / `HYBRID_SEMANTIC_WEIGHT` in the app in sync with whatever you PUT.

---

## Related docs

- [Phase 1 — Cluster & Model Setup](../docs/xpress-service/2026-06-09-opensearch-phase-1-cluster-and-model-setup.md)
- [Phase 2 — Index, Pipelines & Rollout](../docs/xpress-service/2026-06-09-opensearch-phase-2-index-pipelines-and-rollout.md)
- [Operations & Safeguards](../docs/xpress-service/2026-06-09-opensearch-operations-and-safeguards.md)
