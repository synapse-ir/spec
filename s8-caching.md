# S8 — Caching Architecture

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

SYNAPSE uses five independent cache layers (C1–C5) arranged from fastest to slowest, and from most specific to most general. Each layer serves a distinct purpose and is keyed differently. A request may hit multiple layers across a single call.

```
Caller
  │
  ▼
C1 — Routing Decision Cache       (in-process, per router instance)
  │
  ▼
C2 — Prompt Normalization Cache   (distributed, shared across routers)
  │
  ▼
C3 — Result Cache                 (distributed, keyed by idempotency_key)
  │
  ▼
C4 — Semantic Similarity Cache    (distributed, vector-indexed)
  │
  ▼
C5 — Model Provider Prompt Cache  (provider-managed, transparent)
  │
  ▼
Model API
```

Caches are consulted in order C1 → C5. A hit at any layer returns the result without consulting later layers or the model API.

---

## 2. C1 — Routing Decision Cache

### Purpose

Avoid repeated registry lookups for identical routing requests. Routing decisions are deterministic for a given set of task constraints and the current model pool — caching them avoids the registry round-trip on the critical path.

### Key Design

```
SHA-256(
  task_type +
  domain +
  quality_floor (normalized) +
  cost_ceiling_usd (normalized) +
  timeout_ms (bucketed) +
  sorted(allowed_regions) +
  sorted(compliance_tags) +
  pii_present
)
```

Timeout is bucketed in 5-second increments to improve cache hit rate across requests with slightly different timeouts.

### TTL

Default: **60 seconds.** Configurable via `SYNAPSE_C1_TTL_SECONDS`.

The TTL is intentionally short because routing decisions depend on model health (heartbeats) and calibration scores, which change continuously.

### Invalidation Rules

- **Heartbeat status change:** C1 entries are immediately invalidated for any model whose heartbeat transitions from `healthy` to `degraded` or `unavailable`. The registry broadcasts an invalidation event to all router instances.
- **Manifest update:** C1 entries are invalidated when a model's manifest is updated (via `PUT /v1/models/{model_id}`).
- **TTL expiry:** Natural expiry is the baseline invalidation mechanism.

### Failure Behavior

On C1 cache miss or failure (e.g., cache process crash), the request falls through to the registry. C1 is entirely in-process with no network dependency; failure is equivalent to a cold start.

---

## 3. C2 — Prompt Normalization Cache

### Purpose

Many callers send semantically identical requests with minor textual variations — trailing whitespace, different line endings, slightly different phrasing of a fixed system prompt. C2 normalizes the prompt before caching to collapse these variations into a single cache entry.

### Key Design

```
SHA-256(
  model_id +
  task_type +
  normalize(system_prompt) +
  normalize(user_content) +
  max_tokens +
  modality
)
```

Normalization steps: Unicode NFC, collapse whitespace runs, trim leading/trailing whitespace, lowercase for purely structural fields.

### TTL

Default: **5 minutes (300 seconds).** Configurable via `SYNAPSE_C2_TTL_SECONDS`.

### Invalidation Rules

- **TTL expiry** is the primary mechanism.
- **Explicit invalidation:** Callers may pass `Cache-Control: no-store` in the request header to bypass and not populate C2.
- C2 entries are **not** invalidated by model manifest updates, because they are keyed to a specific `model_id` and the model's output for a fixed prompt does not change.

### Failure Behavior

C2 is a distributed cache (Redis or Memcached). On failure (network partition, Redis unavailability), the request falls through to C3. C2 failure must not cause request failure. Log at `WARN` level and increment the `synapse_cache_miss_total{layer="C2",reason="error"}` metric.

---

## 4. C3 — Result Cache

### Purpose

Serve fully deduplicated results for requests carrying an `idempotency_key`. C3 is the only layer that guarantees exactly-once processing semantics — once a result is stored, all future requests with the same key return the cached result regardless of payload content.

### Key Design

```
idempotency_key (verbatim, as supplied by the caller)
```

The `idempotency_key` is caller-supplied and globally scoped within a SYNAPSE deployment. Callers are responsible for key uniqueness. Recommended format: `{service}-{resource-type}-{resource-id}-{operation}`.

### TTL

Default: **24 hours (86400 seconds).** Configurable via `SYNAPSE_C3_TTL_SECONDS`.

### Invalidation Rules

- **TTL expiry** is the only automatic invalidation mechanism.
- There is intentionally no API for explicit C3 invalidation. If a caller needs to reprocess a request (e.g., because the model was updated), they must use a different `idempotency_key`.
- C3 entries are stored after the result is returned to the caller, not before. A result is only committed to C3 if the model call completed successfully (`success: true` in the calibration signal).

### Failure Behavior

On C3 failure, the request continues to C4/model. C3 failure must not cause request failure. The `idempotency_key` is not consumed on failure — the next request with the same key will attempt to populate C3 again. Log at `ERROR` level and page on-call if the C3 error rate exceeds `SYNAPSE_C3_ERROR_THRESHOLD` (default: 1%).

---

## 5. C4 — Semantic Similarity Cache

### Purpose

Serve cached results for requests that are semantically equivalent to a previous request, even if not textually identical. C4 uses vector embeddings to find approximate matches.

### Key Design

C4 is indexed by an embedding vector, not a hash. The lookup process:

1. Embed the normalized user content using a dedicated lightweight embedding model (configured via `SYNAPSE_C4_EMBEDDING_MODEL_ID`).
2. Query the vector index for the nearest neighbor within cosine distance threshold `SYNAPSE_C4_SIMILARITY_THRESHOLD` (default: `0.05`, meaning cosine similarity ≥ 0.95).
3. If a match is found, verify that `task_type`, `domain`, and `model_id` match the cached entry (similarity alone is insufficient).
4. Return the cached result.

### TTL

Default: **1 hour (3600 seconds).** Configurable via `SYNAPSE_C4_TTL_SECONDS`.

Entries with `pii_present: true` in the `compliance_envelope` must not be stored in C4. PII content must not be embedded and stored in a shared vector index.

### Invalidation Rules

- **TTL expiry** is the primary mechanism.
- **PII guard:** C4 is bypassed entirely (read and write) when `compliance_envelope.pii_present` is `true`.
- **Classification guard:** C4 is bypassed when `compliance_envelope.data_classification` is `restricted`.
- **Model update:** When a model's manifest version changes, all C4 entries for that `model_id` are invalidated, because the model's outputs may differ under the new version.

### Failure Behavior

On C4 failure (vector database unavailable, embedding call failure), the request falls through to the model API. C4 failure must not cause request failure. Log at `WARN` level. C4 is an optimization layer; its unavailability has cost/latency implications but no correctness impact.

---

## 6. C5 — Model Provider Prompt Cache

### Purpose

Leverage provider-side prefix caching (e.g., Anthropic's prompt cache, OpenAI's prompt caching) to reduce token processing costs for requests that share a long system prompt prefix. C5 is managed entirely by the model provider — SYNAPSE does not control it directly.

### Key Design

Provider-dependent. Anthropic's implementation caches based on the first N tokens of the prompt that are marked with a `cache_control` block. OpenAI's implementation caches based on prefix hashing.

SYNAPSE adapters are responsible for emitting the provider-specific cache hint markers in the ingress output when the IR's `system_prompt` exceeds the provider's minimum cacheable length (typically 1,024 tokens for Anthropic).

### TTL

Provider-managed. Anthropic: 5 minutes. OpenAI: varies by plan.

### Invalidation Rules

Entirely provider-managed. SYNAPSE cannot invalidate C5 entries. This is acceptable because C5 only caches static context (system prompts), not dynamic request content.

### Failure Behavior

C5 hit/miss is transparent to SYNAPSE. A C5 miss results in higher token costs for that request; it does not cause functional failure. C5 hit status is reported in the `cache_hit` and `cache_layer` fields of the `ProvenanceEntry` when the provider includes cache hit signals in the response (e.g., `cache_read_input_tokens` in Anthropic's usage response).

---

## 7. Environment Variable Configuration Reference

All cache configuration is controlled via environment variables. These may be set in the SYNAPSE deployment environment or in a `.env` file loaded at startup.

### 7.1 C1 — Routing Decision Cache

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_C1_ENABLED` | bool | `true` | Enable or disable C1. |
| `SYNAPSE_C1_TTL_SECONDS` | integer | `60` | Cache entry TTL in seconds. |
| `SYNAPSE_C1_MAX_ENTRIES` | integer | `10000` | Maximum number of entries in the in-process LRU cache. |

### 7.2 C2 — Prompt Normalization Cache

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_C2_ENABLED` | bool | `true` | Enable or disable C2. |
| `SYNAPSE_C2_TTL_SECONDS` | integer | `300` | Cache entry TTL in seconds. |
| `SYNAPSE_C2_BACKEND_URL` | string | — | Redis or Memcached connection URL. Required when C2 is enabled. |
| `SYNAPSE_C2_KEY_PREFIX` | string | `"synapse:c2:"` | Key namespace prefix. |
| `SYNAPSE_C2_MAX_CONTENT_BYTES` | integer | `65536` | Maximum content size (bytes) to cache. Larger payloads bypass C2. |

### 7.3 C3 — Result Cache

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_C3_ENABLED` | bool | `true` | Enable or disable C3. |
| `SYNAPSE_C3_TTL_SECONDS` | integer | `86400` | Cache entry TTL in seconds. |
| `SYNAPSE_C3_BACKEND_URL` | string | — | Redis connection URL. Required when C3 is enabled. |
| `SYNAPSE_C3_KEY_PREFIX` | string | `"synapse:c3:"` | Key namespace prefix. |
| `SYNAPSE_C3_ERROR_THRESHOLD` | float | `0.01` | Error rate fraction above which on-call is paged. |

### 7.4 C4 — Semantic Similarity Cache

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_C4_ENABLED` | bool | `true` | Enable or disable C4. |
| `SYNAPSE_C4_TTL_SECONDS` | integer | `3600` | Cache entry TTL in seconds. |
| `SYNAPSE_C4_BACKEND_URL` | string | — | Vector database connection URL (e.g., Qdrant, Weaviate, Pinecone). Required when C4 is enabled. |
| `SYNAPSE_C4_EMBEDDING_MODEL_ID` | string | — | `model_id` of the embedding model used to generate cache keys. Required when C4 is enabled. |
| `SYNAPSE_C4_SIMILARITY_THRESHOLD` | float | `0.05` | Maximum cosine distance for a cache hit (0.0 = exact, 1.0 = anything). |
| `SYNAPSE_C4_COLLECTION_NAME` | string | `"synapse_c4"` | Vector collection/index name. |
| `SYNAPSE_C4_SKIP_PII` | bool | `true` | Whether to bypass C4 for requests with `pii_present: true`. Must not be set to `false` in production. |
| `SYNAPSE_C4_SKIP_RESTRICTED` | bool | `true` | Whether to bypass C4 for requests with `data_classification: restricted`. |

### 7.5 C5 — Model Provider Prompt Cache

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_C5_ENABLED` | bool | `true` | Enable or disable SYNAPSE-side C5 hint emission. Does not control provider-side behavior. |
| `SYNAPSE_C5_MIN_PROMPT_TOKENS` | integer | `1024` | Minimum system prompt token length before a cache hint is emitted. |

### 7.6 Global Cache Settings

| Variable | Type | Default | Description |
|---|---|---|---|
| `SYNAPSE_CACHE_LOG_LEVEL` | string | `"INFO"` | Log level for cache hit/miss events: `DEBUG`, `INFO`, `WARN`, `ERROR`. |
| `SYNAPSE_CACHE_METRICS_ENABLED` | bool | `true` | Whether to emit Prometheus metrics for cache operations. |
| `SYNAPSE_CACHE_METRICS_PREFIX` | string | `"synapse_cache"` | Prefix for Prometheus metric names. |
