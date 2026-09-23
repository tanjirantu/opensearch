# OpenSearch Setup

End-to-end runbook for the hybrid product search cluster: start the cluster, deploy an
embedding model, provision the **ingest pipeline + search pipeline + index** from the app
(`yarn create:products-index`), backfill from the app, and cut the `products` alias over.

Steps 2 onward run in **OpenSearch Dashboards → Dev Tools** (`http://localhost:5601/app/dev_tools`)
unless a step says otherwise.

> **Alias vs. index.** The alias is always `products`. Concrete indices are versioned
> (`products_v1`, `products_v2`, …) and bumped on every mapping change — analysis settings are
> static on a live index, so a new analyzer means a new index plus a reindex. Every step below
> provisions `<target_index>` and cuts the alias over from `<active_index>`; nothing in this
> runbook is specific to a particular version number.

### Placeholders

Substitute these throughout — they are the only values that change between runs. Nothing else in
this document should be edited to fit your cluster.

| Placeholder | What it is | Where it comes from |
| --- | --- | --- |
| `<embedding_model>` | Remote embedding model, e.g. `text-embedding-3-small` | You choose it — step 3 |
| `<active_index>` | Concrete index currently behind the `products` alias | `GET /_alias/products` (step 10) |
| `<target_index>` | New index you are provisioning this cycle | You choose it — next version number |
| `<connector_id>` | Remote model connector | Response to step 3, or step 10 |
| `<model_id>` | Deployed embedding model | Completed task in step 4, or step 10 |
| `<task_id>` | Async model-deployment task | Response to step 4 |
| `<OPENAI_API_KEY>` | Embedding-provider API key | Your provider account — never commit it |
| `<GATEWAY_SHARED_TOKEN>` | `x-gateway-token` for the app's internal reindex endpoint | `GATEWAY_SHARED_TOKEN` in the app env |

The pipeline ids (`products-embedding-ingest`, `products-hybrid-pipeline`) are conventions, not
placeholders — keep them stable, except per-environment suffixes (see Notes).

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

The connector is the cluster's handle on a remote embedding API. The example below is OpenAI; any
provider whose endpoint matches `trusted_connector_endpoints_regex` from step 2 works the same way
— only `url`, `request_body` and the pre/post-process functions change.

> **Dimension contract — decide this before anything else.** Whatever model you pick, its output
> dimension must equal the app's `EMBEDDING_DIMENSION` (default **1536**), which
> `productsIndexBody.ts` uses for `embedding_vector.dimension`. It cannot change in place once
> indexing has begun: a different dimension means a new index and a reindex.

Common OpenAI choices:

| Model | Native dimension | Note |
| --- | --- | --- |
| `text-embedding-3-small` | 1536 | Matches the default `EMBEDDING_DIMENSION` |
| `text-embedding-3-large` | 3072 | Either set `EMBEDDING_DIMENSION=3072`, or truncate — see below |

`text-embedding-3-*` models accept a `dimensions` parameter, so a larger model can be pinned back
to the committed 1536 by adding it to the request body (OpenAI truncates server-side and the
vectors stay usable):

```
"request_body": "{ \"input\": ${parameters.input}, \"model\": \"${parameters.model}\", \"dimensions\": 1536 }"
```

Replace `<OPENAI_API_KEY>` with your key, and `<embedding_model>` with the model you chose:

```
POST /_plugins/_ml/connectors/_create
{
  "name": "OpenAI Embeddings Connector",
  "description": "Connector for OpenAI <embedding_model>",
  "version": 1,
  "protocol": "http",
  "parameters": {
    "model": "<embedding_model>"
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

Replace `<connector_id>` with the value from step 3. The `name` you give here is what step 10
searches on, so make it descriptive and unique — it is the only human-readable handle on the
model once the response scrolls away.

```
POST /_plugins/_ml/models/_register?deploy=true
{
  "name": "OpenAI <embedding_model>",
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

Confirm `"model_state": "DEPLOYED"`, a `shape` matching your `EMBEDDING_DIMENSION`, and a
non-empty `data` array.
**If `_predict` fails, stop here.** Step 5 checks that the model is `DEPLOYED`, but not that
inference actually works.

---

## 5. Provision from the app

The app owns every cluster-side resource except the model: the **ingest pipeline**
(`embedding_text → embedding_vector` at index time), the **hybrid search pipeline** (score fusion
at query time) and the **index** itself. Their definitions live in `xpress-service`
(`core/provisioning/` for the definitions, `service/createProductsIngestPipeline.ts` and
`service/createHybridSearchPipeline.ts` for the provisioning steps), so a new environment is
provisioned from source.

### 5.1 Set the app env

Set these in `xpress-service`. The script in 5.2 reads them, and so does the app at runtime:

```env
OPENSEARCH_NODE=https://localhost:9200
OPENSEARCH_USER=...
OPENSEARCH_PASSWORD=...
OPENSEARCH_SSL_VERIFY=false            # true once real certs are in place

OPENSEARCH_PRODUCTS_ALIAS=products
OPENSEARCH_INGEST_PIPELINE=products-embedding-ingest

OPENSEARCH_HYBRID_ENABLED=true
OPENSEARCH_HYBRID_SEARCH_PIPELINE=products-hybrid-pipeline
OPENSEARCH_EMBEDDING_MODEL_ID=<model_id from step 4>
OPENSEARCH_HYBRID_K=100
HYBRID_LEXICAL_WEIGHT=0.6     # written into the search pipeline by 5.2
HYBRID_SEMANTIC_WEIGHT=0.4    # written into the search pipeline by 5.2
EMBEDDING_DIMENSION=1536      # must match the model (step 3)

SEARCH_INDEXED_SELLER_UIDS=<comma-separated seller uids>
```

`SEARCH_INDEXED_SELLER_UIDS` is **fail-closed**: unset means nothing is indexed at all.

### 5.2 Run the provisioning script

From `xpress-service`:

```bash
yarn create:products-index <target_index>
```

It stops at the first failure, and every refusal comes before its first write:

1. **refuses a `<target_index>` that already exists**, because its mapping and settings were fixed
   when it was created;
2. checks that `OPENSEARCH_EMBEDDING_MODEL_ID` exists on this cluster and is `DEPLOYED`;
3. creates or replaces the ingest pipeline (**refusing** to switch an existing one to a different
   model), then the search pipeline if `OPENSEARCH_HYBRID_SEARCH_PIPELINE` is set and it does not
   exist yet (an existing one is left unchanged), then the index;
4. reads the live index back and checks the vector dimension, `hnsw` / `lucene` / `cosinesimil`,
   and `default_pipeline`;
5. on a fresh cluster with no `products` alias, points the alias at the new, empty index, so
   search returns no hits instead of `index_not_found` and the upsert consumer can start filling
   it. An existing alias is left for step 9 to switch.

`yarn create:ingest-pipeline` recreates only the ingest pipeline, e.g. after a processor change.
The pipeline is shared by every products index, so **changing the model** changes the live
index's vectors from the next write on, while stored ones keep the old model: search then scores
across two embedding spaces until a full reindex. It is refused unless you pass
`--allow-model-change`, and should be followed by a reindex.

> `workflow-template.json` still provisions the two pipelines through Flow Framework, with the
> same definitions, but not the index. It is not part of this runbook, and 5.2 overwrites both
> pipelines with the app's definitions either way.

---

## 6. Verify and smoke-test

```
GET /_ingest/pipeline/products-embedding-ingest
GET /_search/pipeline/products-hybrid-pipeline
GET /<target_index>/_mapping
GET /<target_index>/_settings
```

Confirm:

- the ingest pipeline has a `text_embedding` processor with `field_map: { embedding_text: embedding_vector }`,
  `skip_existing: true`, and an `on_failure` that sets `embedding_failed: true`;
- the search pipeline has a `normalization-processor` (`min_max` / `arithmetic_mean`, weights `[0.6, 0.4]`);
- the index has `index.knn: true`, `default_pipeline: products-embedding-ingest`, and the
  `sku_trigram` / `english_stem` analyzers under `analysis`;
- `embedding_vector` is a `knn_vector` of the dimension fixed in step 3 (`hnsw` / `lucene` / `cosinesimil`);
- `skus` is `text` with `analyzer: sku_trigram` and a `skus.keyword` subfield;
- the mapping is `"dynamic": "strict"`, and `artworks`, `artworkLayouts`, `variants`, `images`
  and `prices` are `{"type": "object", "enabled": false}` with no sub-properties.

**Ingest smoke test** — proves OpenSearch generates the vector, not the app:

```
POST /<target_index>/_doc/test-smoke
{ "name": "Smoke Test Product", "sellerUid": "test", "isActive": true, "embedding_text": "Classic cotton polo shirt" }

GET /<target_index>/_doc/test-smoke
```

`embedding_vector` must be a non-empty array of exactly `EMBEDDING_DIMENSION` elements. If the
document has `embedding_failed: true` and no vector instead, the model call failed; recheck
step 4's `_predict`. Then clean up:

```
DELETE /<target_index>/_doc/test-smoke
```

**Hybrid smoke test:**

```
GET /<target_index>/_search?search_pipeline=products-hybrid-pipeline
{
  "query": { "hybrid": { "queries": [
    { "match": { "name": "polo shirt" } },
    { "neural": { "embedding_vector": { "query_text": "polo shirt", "model_id": "<model_id>", "k": 5 } } }
  ] } }
}
```

Re-index the smoke doc first if you already deleted it; confirm `hits.hits` is non-empty.

---

## 7. Restart the app

Restart or redeploy `xpress-service` so search and the `sellable.product.upserted` consumer pick up
the env from step 5.1. The consumer writes through the alias with `require_alias`, so it never
creates an index. On a cluster with no alias its writes fail and are reported to Sentry.

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
  -d '{ "targetIndex": "<target_index>", "dryRun": true }'
```

Confirm the estimated document count looks right, then run it:

```bash
curl -X POST "http://localhost:3015/api/v2/internal/products/reindex" \
  -H "Content-Type: application/json" \
  -H "x-gateway-token: <GATEWAY_SHARED_TOKEN>" \
  -d '{
    "targetIndex": "<target_index>",
    "alias": "products",
    "batchSize": 200,
    "validateAfterIndex": true,
    "switchAliasAfterValidation": false
  }'
```

Confirm `errors: 0` and `validation.passed: true`. Validation requires `embeddingVectorCount` to
equal `embeddingTextCount`; when it fails, `embeddingFailedCount` counts the documents the
pipeline indexed without a vector. A run with any failed item never switches the alias. The reindex
also refuses, with a 400, a target without `default_pipeline` or with a missing ingest pipeline.
Spot-check a document: both `embedding_text` and `embedding_vector` must be present:

```
GET /<target_index>/_doc/<known_product_uid>
```

---

## 9. Cut the alias over

Either pass `switchAliasAfterValidation: true` on the reindex call, or switch by hand. Confirm
the current target first (`GET /_alias/products`, see [step 10](#10-inspecting-whats-already-in-the-cluster))
and name *that* index in the `remove` action — a `remove` naming an index the alias is not on
fails the whole atomic request, and the alias never moves:

```
POST /_aliases
{ "actions": [
  { "remove": { "index": "<active_index>", "alias": "products" } },
  { "add":    { "index": "<target_index>", "alias": "products" } }
] }

GET /_cat/aliases/products?v
```

Validate search through the app — this one is a `curl` against the service, not Dev Tools —
confirming `debug.mode === "hybrid"` in non-prod:

```bash
curl "http://localhost:3015/api/v2/products/search?q=polo+shirt&sellerUid=<seller_uid>"
```

After ~24h stable, drop the old index:

```
DELETE /<active_index>
```

---

## 10. Inspecting what's already in the cluster

Everything created in steps 3–5 is stored cluster-side and survives restarts, so on a cluster you
did not provision yourself — or after losing the ids you were told to save — look them up rather
than recreating them. Recreating a connector or model produces a *second* copy with a new id.

**Connectors** — list every one, to recover a `connector_id`:

```
POST /_plugins/_ml/connectors/_search
{
  "query": {
    "match_all": {}
  }
}
```

**Models** — find one by name, to recover its `model_id` and confirm `model_state: "DEPLOYED"`:

```
POST /_plugins/_ml/models/_search
{
  "query": {
    "match": {
      "name": "OpenAI <embedding_model>"
    }
  }
}
```

Match on the `name` you registered in step 4. Swap in `match_all` to list everything, including models
left over from earlier attempts — more than one `DEPLOYED` embedding model is a smell: the index
was built with exactly one of them, and querying with the other returns nonsense rankings.

**Workflows** — only on clusters provisioned through Flow Framework before the app took over.
List them, to recover a `workflow_id`:

```
GET /_plugins/_flow_framework/workflow/_search
{
  "query": {
    "match_all": {}
  }
}
```

Then check what a given workflow actually created. Its `resources_created` array is the
authoritative answer to "which index, ingest pipeline and search pipeline does this workflow own":

```
GET /_plugins/_flow_framework/workflow/<workflow_id>/_status
```

**Index settings** — confirm `index.knn`, `default_pipeline`, `number_of_replicas` and the
analyzers on a concrete index:

```
GET /<target_index>/_settings
```

**Alias** — which concrete index is `products` currently pointing at:

```
GET /_alias/products
```

`GET /_cat/aliases/products?v` gives the same answer in table form. Check this before every
cutover: the "old" index in the step-9 swap must be whatever this returns, not whatever the
runbook example says.

---

## Notes

- **Single source of truth.** The index body is
  `xpress-service/src/services/product/search/core/provisioning/productsIndexBody.ts`, and the
  ingest pipeline is `getIngestPipelineBody.ts` next to it. Nothing here defines the index any more. The mapping
  is `strict` and pins the vector method (`hnsw` / `lucene` / `cosinesimil`).
- **`number_of_replicas: 0`** is set in `productsIndexBody.ts` and is fine for local and
  single-node environments. Before an alias cutover makes an index the production search target,
  raise it (`PUT /<target_index>/_settings {"index.number_of_replicas": 1}`) — with no replica, one
  data-node restart takes `products` red and search starts returning 503s.
- **Response-only objects are not mapped.** `artworks`, `artworkLayouts`, `variants`, `images`
  and `prices` are `enabled: false`: kept in `_source`, never parsed. Mapped dynamically they added
  140 guessed fields and rejected real products on `long`/`float` conflicts inside
  `artworkLayouts.fulfillEngine`.
- **Search pipeline weights.** `yarn create:search-pipeline` is the one way to change the search
  pipeline: it writes `HYBRID_LEXICAL_WEIGHT` / `HYBRID_SEMANTIC_WEIGHT` into it, replacing
  whatever is there. `yarn create:products-index` creates the pipeline only when it is missing,
  and the reindex never writes it. With hybrid search on, the reindex refuses (400) to run while
  the pipeline is missing. Each weight must be in [0, 1] and the two must add up to 1, with
  OpenSearch's 0.01 of slack. A blank value means the default (0.6 / 0.4), and anything
  malformed is refused before a script writes anything.
- **Client-side embedding vars are legacy.** `EMBEDDING_ENABLED`, `EMBEDDING_MODEL`,
  `EMBEDDING_SERVICE_URL` predate cluster-side inference. Embeddings are generated in OpenSearch;
  leave `EMBEDDING_ENABLED` unset.
- **Per-environment.** Use distinct pipeline names per cluster (e.g. `products-hybrid-pipeline-dev`)
  and set `OPENSEARCH_HYBRID_SEARCH_PIPELINE` accordingly. Every other step is identical.

### Updating the search pipeline alone

To retune weights without re-provisioning, PUT the search pipeline directly — it is
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
