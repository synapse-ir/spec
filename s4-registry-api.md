# S4 — Registry API

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

The SYNAPSE Registry API is a REST/JSON service that manages model manifests, exposes the routing decision endpoint, and collects calibration signals. All endpoints are versioned under `/v1`.

**Base URL:** `https://registry.synapse.internal`

**Protocol:** HTTPS only. TLS 1.2 minimum.

---

## 2. Authentication

All endpoints require authentication unless marked **None**. The registry supports two authentication methods:

- **Bearer token** — Pass a SYNAPSE service token in the `Authorization` header:
  ```
  Authorization: Bearer <token>
  ```
- **mTLS** — Mutual TLS with a client certificate issued by the SYNAPSE PKI. Appropriate for service-to-service calls in secure environments.

Tokens are scoped. Registry write operations (register, deregister) require the `registry:write` scope. Read and routing operations require `registry:read`. Calibration submission requires `calibration:write`.

---

## 3. Endpoints

| # | Method | Path | Auth | Description |
|---|---|---|---|---|
| 1 | `GET` | `/v1/route` | Required | Select the best-fit model for a given IR. Primary routing endpoint. |
| 2 | `GET` | `/v1/models` | Required | List all registered models with optional filters. |
| 3 | `GET` | `/v1/models/{model_id}` | Required | Retrieve the full manifest for a specific model. |
| 4 | `POST` | `/v1/models` | Required (`registry:write`) | Register a new model manifest. |
| 5 | `PUT` | `/v1/models/{model_id}` | Required (`registry:write`) | Replace the manifest for an existing model. |
| 6 | `DELETE` | `/v1/models/{model_id}` | Required (`registry:write`) | Deregister a model. |
| 7 | `GET` | `/v1/models/{model_id}/health` | Required | Retrieve the latest heartbeat for a model. |
| 8 | `POST` | `/v1/calibration/signal` | Required (`calibration:write`) | Submit a calibration signal after a completed model call. |
| 9 | `GET` | `/v1/health` | None | Registry service health check. |

---

## 4. GET /v1/route

Select the optimal model for a given Canonical IR. This is the critical path endpoint; it must respond in under 20 ms at p99.

### 4.1 Request

The request body is a subset of the Canonical IR — only the fields needed for routing decisions are required. A full IR is also accepted.

| Field | Type | Required | Description |
|---|---|---|---|
| `task_header` | object | **Yes** | The `task_header` from the Canonical IR. |
| `compliance_envelope` | object | **Yes** | The `compliance_envelope` from the Canonical IR. |
| `payload_summary` | object | No | Lightweight payload descriptor (avoids sending full content to the registry). |
| `payload_summary.modality` | string | No | The `payload.modality` value. |
| `payload_summary.estimated_tokens` | integer | No | Estimated input token count for routing budget calculations. |

**Example request:**

```json
{
  "task_header": {
    "task_type": "extract",
    "domain": "legal",
    "priority": 7,
    "timeout_ms": 30000,
    "quality_floor": 0.85,
    "cost_ceiling_usd": 0.05
  },
  "compliance_envelope": {
    "data_classification": "confidential",
    "pii_present": true,
    "allowed_regions": ["US", "GB"],
    "compliance_tags": ["GDPR", "SOC2"]
  },
  "payload_summary": {
    "modality": "text",
    "estimated_tokens": 800
  }
}
```

### 4.2 Response — Successful Route

HTTP `200 OK`

| Field | Type | Description |
|---|---|---|
| `model_id` | string | The selected model's `model_id`. |
| `model_version` | string | The selected model's version string. |
| `endpoint_url` | string | The model's API endpoint URL. |
| `adapter_ref` | object | The adapter reference to use for this call (matches the runtime's language preference). |
| `score` | number | Composite routing score [0.0, 1.0]. |
| `score_breakdown` | object | Decomposition of the composite score. See §4.3. |
| `alternatives` | array | Up to 3 alternative models in ranked order, each with `model_id` and `score`. |
| `routing_trace_id` | string | Identifier for this routing decision. Include in the provenance entry for audit correlation. |
| `cache_ttl_seconds` | integer | How long this routing decision may be cached by the caller. |

**Example response:**

```json
{
  "model_id": "anthropic/claude-sonnet-4-6",
  "model_version": "claude-sonnet-4-6",
  "endpoint_url": "https://api.anthropic.com/v1",
  "adapter_ref": {
    "package": "@synapse/adapter-anthropic",
    "package_version": "^1.4.0",
    "export_name": "ClaudeSonnetAdapter"
  },
  "score": 0.923,
  "score_breakdown": {
    "quality": 0.970,
    "latency": 0.881,
    "cost": 0.944,
    "compliance": 1.0,
    "availability": 0.9995
  },
  "alternatives": [
    { "model_id": "openai/gpt-4o", "score": 0.871 },
    { "model_id": "google/gemini-1.5-pro", "score": 0.844 }
  ],
  "routing_trace_id": "rt-018e9a3c-5f2b-7e1a-b3c4-8d0f2e4a6c88",
  "cache_ttl_seconds": 60
}
```

### 4.3 `score_breakdown` Fields

The composite score is a weighted combination of the following sub-scores:

| Field | Range | Description |
|---|---|---|
| `quality` | [0.0, 1.0] | Model's calibrated quality score for the requested `task_type`/`domain` combination. |
| `latency` | [0.0, 1.0] | Normalized latency score, derived from the model's p95 latency relative to the `timeout_ms` constraint. |
| `cost` | [0.0, 1.0] | Normalized cost score, derived from estimated call cost relative to `cost_ceiling_usd`. |
| `compliance` | 0.0 or 1.0 | Binary: 1.0 if the model fully satisfies all `compliance_envelope` constraints, 0.0 otherwise. A model with `compliance = 0.0` is never selected. |
| `availability` | [0.0, 1.0] | Model's current availability based on recent heartbeat data and provider SLO. |

Default weights are `quality: 0.40, latency: 0.25, cost: 0.20, compliance: disqualifier, availability: 0.15`. Weights are configurable per deployment via registry settings.

### 4.4 Response — No Candidates Found

HTTP `422 Unprocessable Entity` — error code `ROUTE_NO_CANDIDATES`

Returned when no registered model satisfies all hard constraints (compliance, `quality_floor`, `cost_ceiling_usd`, `timeout_ms`, `allowed_regions`).

| Field | Type | Description |
|---|---|---|
| `error` | string | `"ROUTE_NO_CANDIDATES"` |
| `message` | string | Human-readable explanation. |
| `violated_constraints` | array\<string\> | List of constraint names that eliminated all candidates. |
| `relaxation_suggestions` | array\<RelaxationSuggestion\> | Ordered list of suggestions for relaxing constraints to find a match. |

**RelaxationSuggestion schema:**

| Field | Type | Description |
|---|---|---|
| `constraint` | string | The constraint to relax (e.g. `"quality_floor"`, `"cost_ceiling_usd"`, `"allowed_regions"`). |
| `current_value` | any | The value that was requested. |
| `suggested_value` | any | The suggested relaxed value. |
| `candidate_count` | integer | Number of models that would become eligible if this suggestion were applied. |
| `rationale` | string | Brief explanation of why this relaxation would help. |

**Example response:**

```json
{
  "error": "ROUTE_NO_CANDIDATES",
  "message": "No registered model satisfies all routing constraints.",
  "violated_constraints": ["quality_floor", "allowed_regions"],
  "relaxation_suggestions": [
    {
      "constraint": "quality_floor",
      "current_value": 0.95,
      "suggested_value": 0.88,
      "candidate_count": 3,
      "rationale": "Reducing quality_floor to 0.88 makes 3 models eligible, all with strong extract/legal scores."
    },
    {
      "constraint": "allowed_regions",
      "current_value": ["US"],
      "suggested_value": ["US", "EU"],
      "candidate_count": 5,
      "rationale": "Permitting EU processing regions expands the eligible pool to 5 models."
    }
  ]
}
```

---

## 5. GET /v1/models

List all registered models, with optional filtering.

### 5.1 Query Parameters

| Parameter | Type | Description |
|---|---|---|
| `task_type` | string | Filter to models supporting this `task_type`. |
| `domain` | string | Filter to models supporting this `domain`. |
| `compliance_tag` | string | Filter to models holding this compliance certification. |
| `region` | string | Filter to models that can process data in this ISO 3166-1 alpha-2 region. |
| `status` | string | Filter by heartbeat status: `healthy`, `degraded`, `unavailable`. Default: `healthy,degraded`. |
| `page` | integer | 1-based page number. Default: `1`. |
| `page_size` | integer | Results per page. Default: `20`. Max: `100`. |

### 5.2 Response

HTTP `200 OK`

```json
{
  "models": [
    {
      "model_id": "anthropic/claude-sonnet-4-6",
      "display_name": "Claude Sonnet 4.6",
      "provider": "Anthropic",
      "status": "healthy",
      "supported_task_types": ["extract", "summarize", "generate", "classify", "translate", "validate"],
      "supported_domains": ["general", "legal", "medical", "finance", "scientific", "code", "conversational"]
    }
  ],
  "total": 1,
  "page": 1,
  "page_size": 20
}
```

---

## 6. GET /v1/models/{model_id}

Retrieve the full registered manifest for a model.

**Path parameter:** `model_id` — URL-encoded model identifier.

**Response:** HTTP `200 OK` with the full manifest object (see S3).

**Error:** HTTP `404 Not Found` with error code `MODEL_NOT_FOUND` if the model is not registered.

---

## 7. POST /v1/models

Register a new model manifest. The request body must be a valid manifest (see S3). The registry validates the manifest against the schema and rejects malformed documents.

**Response:** HTTP `201 Created` with the registered manifest and a generated `registered_at` timestamp.

**Errors:**
- `400` — `MANIFEST_VALIDATION_ERROR`: Schema validation failed.
- `409` — `MODEL_ALREADY_EXISTS`: A model with this `model_id` is already registered. Use `PUT` to update.

---

## 8. PUT /v1/models/{model_id}

Replace the manifest for an existing model. The new manifest must have the same `model_id`. If `model_version` changes, the registry records a version history entry.

**Response:** HTTP `200 OK` with the updated manifest.

**Error:** HTTP `404 Not Found` with `MODEL_NOT_FOUND` if the model is not registered.

---

## 9. DELETE /v1/models/{model_id}

Deregister a model. The router immediately stops routing to this model. Historical provenance records that reference the model are preserved.

**Response:** HTTP `204 No Content`.

**Error:** HTTP `404 Not Found` with `MODEL_NOT_FOUND`.

---

## 10. GET /v1/models/{model_id}/health

Retrieve the most recent heartbeat for a model.

**Response:** HTTP `200 OK` with a `HeartbeatResponse` object (see S3 §6).

---

## 11. POST /v1/calibration/signal

Submit a calibration signal after a completed model call. The calibration pipeline aggregates these signals to update `quality_scores` and latency percentiles in the model manifest.

### 11.1 CalibrationSignal Schema

| Field | Type | Required | Description |
|---|---|---|---|
| `model_id` | string | **Yes** | The model that served the request. |
| `routing_trace_id` | string | **Yes** | The `routing_trace_id` returned by `/v1/route` for this call. |
| `task_type` | string | **Yes** | The `task_type` from the IR's `task_header`. |
| `domain` | string | **Yes** | The `domain` from the IR's `task_header`. |
| `latency_ms` | integer | **Yes** | Measured end-to-end latency in milliseconds. |
| `success` | boolean | **Yes** | Whether the model call completed successfully. |
| `quality_score` | number | No | Human or automated quality rating for the output [0.0, 1.0]. Omit if not evaluated. |
| `quality_method` | string | No | How `quality_score` was determined: `"human"`, `"llm_judge"`, `"automated_metric"`. |
| `input_tokens` | integer | No | Actual input tokens consumed. |
| `output_tokens` | integer | No | Actual output tokens generated. |
| `cost_usd` | number | No | Actual cost in USD for this call. |
| `error_code` | string | No | If `success` is `false`, the error code from the model or adapter. |
| `cache_hit` | boolean | No | Whether the result was served from a cache. |
| `client_id` | string | No | Identifier of the calling service, for per-client calibration analysis. |
| `timestamp_utc` | string | No | ISO 8601 UTC timestamp. Defaults to server receipt time if omitted. |

**Example request:**

```json
{
  "model_id": "anthropic/claude-sonnet-4-6",
  "routing_trace_id": "rt-018e9a3c-5f2b-7e1a-b3c4-8d0f2e4a6c88",
  "task_type": "extract",
  "domain": "legal",
  "latency_ms": 1240,
  "success": true,
  "quality_score": 0.97,
  "quality_method": "llm_judge",
  "input_tokens": 812,
  "output_tokens": 234,
  "cost_usd": 0.0059
}
```

**Response:** HTTP `202 Accepted`. Calibration signals are processed asynchronously; the response body is empty.

---

## 12. GET /v1/health

Registry service health check. No authentication required.

**Response:** HTTP `200 OK`

```json
{
  "status": "healthy",
  "version": "1.0.0",
  "timestamp_utc": "2026-05-06T14:23:01Z",
  "registered_models": 47,
  "healthy_models": 45
}
```
