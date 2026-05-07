# S3 — Capability Manifest

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

Every model registered in SYNAPSE must publish a **Capability Manifest** — a structured document that describes what the model can do, how well it performs, where it may operate, and how to reach it. The router uses manifests to make routing decisions; the registry uses them for discovery and health monitoring.

Manifests are static at a given model version. When a model's capabilities or performance profile change materially, a new version must be registered.

---

## 2. Capability Manifest Schema

### 2.1 Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `manifest_version` | string | **Yes** | Semantic version of the manifest schema (e.g. `"1.0.0"`). |
| `model_id` | string | **Yes** | Globally unique model identifier within the SYNAPSE registry (e.g. `"anthropic/claude-sonnet-4-6"`). |
| `model_version` | string | **Yes** | Model provider's version string. |
| `display_name` | string | **Yes** | Human-readable name for UI display. |
| `provider` | string | **Yes** | Name of the model provider (e.g. `"Anthropic"`, `"OpenAI"`, `"Google"`). |
| `endpoint_url` | string | **Yes** | Base URL for the model's API endpoint. |
| `supported_task_types` | array\<string\> | **Yes** | List of `task_type` values this model supports. Must be a non-empty subset of the values defined in S1 §3.1. |
| `supported_domains` | array\<string\> | **Yes** | List of `domain` values this model is suitable for. Must be a non-empty subset of the values defined in S1 §3.2. |
| `supported_modalities` | array\<string\> | **Yes** | Input modalities accepted: any subset of `["text", "embedding", "structured", "binary"]`. |
| `supported_languages` | array\<string\> | No | BCP-47 language tags. Omitting this field means the model makes no language guarantee. |
| `context_window_tokens` | integer | **Yes** | Maximum number of tokens (input + output) the model can process in a single call. |
| `max_output_tokens` | integer | **Yes** | Maximum number of output tokens the model will generate. |
| `performance_profile` | object | **Yes** | Measured quality and latency characteristics. See §3. |
| `compliance` | object | **Yes** | Compliance posture and data residency controls. See §4. |
| `adapter_refs` | object | **Yes** | References to the ingress/egress adapters for this model. See §5. |
| `pricing` | object | No | Cost parameters for routing budget enforcement. See §2.2. |
| `deprecation` | object | No | Present if this model version is deprecated. See §2.3. |

### 2.2 Pricing Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `input_cost_per_1k_tokens` | number | No | USD cost per 1,000 input tokens. |
| `output_cost_per_1k_tokens` | number | No | USD cost per 1,000 output tokens. |
| `flat_call_cost` | number | No | Fixed USD cost per API call, independent of token count. |
| `currency` | string | No | ISO 4217 currency code. Default: `"USD"`. |

### 2.3 Deprecation Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `deprecated_at` | string | **Yes** | ISO 8601 UTC timestamp when this version was deprecated. |
| `sunset_at` | string | No | ISO 8601 UTC timestamp when this version will stop serving requests. |
| `replacement_model_id` | string | No | `model_id` of the recommended replacement. |
| `migration_notes` | string | No | Human-readable guidance for migration. |

---

## 3. Performance Profile

The `performance_profile` object contains empirically measured quality and latency data. Values are populated by the calibration pipeline (see S4 §4) and updated continuously. Initial registration may provide estimated values, which the calibration system will refine.

| Field | Type | Required | Description |
|---|---|---|---|
| `quality_scores` | object | **Yes** | Map of `task_type` → quality score [0.0, 1.0]. Must include a score for every `task_type` listed in `supported_task_types`. |
| `domain_quality_scores` | object | No | Map of `"task_type/domain"` → quality score [0.0, 1.0] for domain-specific performance. Overrides `quality_scores` for matching combinations. |
| `p50_latency_ms` | integer | **Yes** | Median end-to-end latency in milliseconds under normal load. |
| `p95_latency_ms` | integer | **Yes** | 95th-percentile end-to-end latency in milliseconds. |
| `p99_latency_ms` | integer | No | 99th-percentile end-to-end latency in milliseconds. |
| `throughput_rpm` | integer | No | Measured requests per minute the endpoint sustains without degradation. |
| `calibration_sample_size` | integer | No | Number of samples used to compute the above values. |
| `calibration_timestamp_utc` | string | No | ISO 8601 UTC timestamp of the last calibration run. |
| `availability_slo` | number | No | Provider's stated availability SLO as a fraction (e.g. `0.999` for 99.9%). |

### 3.1 Example `quality_scores`

```json
{
  "quality_scores": {
    "extract": 0.94,
    "summarize": 0.91,
    "generate": 0.88,
    "classify": 0.93,
    "translate": 0.87
  },
  "domain_quality_scores": {
    "extract/legal": 0.97,
    "extract/medical": 0.92,
    "summarize/legal": 0.95
  }
}
```

---

## 4. Compliance Fields

The `compliance` object declares the model's compliance posture. The router uses this to enforce `compliance_envelope` constraints from the IR.

| Field | Type | Required | Description |
|---|---|---|---|
| `certifications` | array\<string\> | **Yes** | Compliance certifications the model/infrastructure holds (e.g. `"SOC2"`, `"ISO27001"`, `"HIPAA"`, `"FedRAMP"`, `"GDPR"`). Empty list is valid. |
| `data_residency` | object | **Yes** | Data residency capabilities. See §4.1. |
| `pii_capable` | boolean | **Yes** | Whether this model may process PII data. |
| `audit_logging` | boolean | **Yes** | Whether the provider produces durable audit logs for each request. |
| `data_retention_policy` | string | No | Provider's data retention policy identifier or description. |
| `zero_data_retention` | boolean | No | Whether the provider guarantees zero retention of request/response data. Default: `false`. |
| `encryption_at_rest` | boolean | No | Whether stored data is encrypted at rest. |
| `encryption_in_transit` | boolean | No | Whether data is encrypted in transit (TLS 1.2+). Default: `true`. |

### 4.1 Data Residency Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `processing_regions` | array\<string\> | **Yes** | ISO 3166-1 alpha-2 country codes where data may be processed. |
| `storage_regions` | array\<string\> | No | ISO 3166-1 alpha-2 country codes where data may be stored. Defaults to `processing_regions`. |
| `cross_region_transfer` | boolean | No | Whether data may transit regions outside `processing_regions`. Default: `false`. |

---

## 5. Adapter References

The `adapter_refs` object tells the runtime which adapter implementations to use for this model.

| Field | Type | Required | Description |
|---|---|---|---|
| `python` | PythonAdapterRef | No | Reference to the Python adapter. At least one of `python` or `typescript` must be present. |
| `typescript` | TypeScriptAdapterRef | No | Reference to the TypeScript adapter. At least one of `python` or `typescript` must be present. |

### 5.1 PythonAdapterRef

| Field | Type | Required | Description |
|---|---|---|---|
| `package` | string | **Yes** | PyPI package name (e.g. `"synapse-adapter-openai"`). |
| `package_version` | string | **Yes** | Exact version constraint (e.g. `">=2.1.0,<3.0.0"`). |
| `class_path` | string | **Yes** | Fully-qualified Python class path (e.g. `"synapse_adapter_openai.NERAdapter"`). |

### 5.2 TypeScriptAdapterRef

| Field | Type | Required | Description |
|---|---|---|---|
| `package` | string | **Yes** | npm package name (e.g. `"@synapse/adapter-anthropic"`). |
| `package_version` | string | **Yes** | npm version range (e.g. `"^1.4.0"`). |
| `export_name` | string | **Yes** | Named export from the package (e.g. `"AnthropicSummarizeAdapter"`). |

---

## 6. HeartbeatResponse

Models registered in SYNAPSE must expose a `/health` endpoint that returns a `HeartbeatResponse`. The registry polls this endpoint at the interval specified in the manifest or every 30 seconds by default.

| Field | Type | Required | Description |
|---|---|---|---|
| `model_id` | string | **Yes** | Must match the `model_id` in the registered manifest. |
| `status` | string | **Yes** | Current health status: `"healthy"`, `"degraded"`, `"unavailable"`. |
| `timestamp_utc` | string | **Yes** | ISO 8601 UTC timestamp of when this heartbeat was generated. |
| `latency_ms` | integer | No | Self-reported round-trip latency for a no-op probe call, in milliseconds. |
| `error_rate_1m` | number | No | Fraction of requests that errored in the past 60 seconds [0.0, 1.0]. |
| `queue_depth` | integer | No | Number of requests currently queued, if the model exposes this. |
| `message` | string | No | Human-readable status description. Required when `status` is `"degraded"` or `"unavailable"`. |

### 6.1 Status Semantics

| Status | Router Behavior |
|---|---|
| `healthy` | Eligible for routing. |
| `degraded` | Eligible for routing; router increases weight of alternatives. |
| `unavailable` | Excluded from routing. Retry heartbeat after back-off interval. |

---

## 7. Complete Manifest Example

```json
{
  "manifest_version": "1.0.0",
  "model_id": "anthropic/claude-sonnet-4-6",
  "model_version": "claude-sonnet-4-6",
  "display_name": "Claude Sonnet 4.6",
  "provider": "Anthropic",
  "endpoint_url": "https://api.anthropic.com/v1",
  "supported_task_types": ["extract", "summarize", "generate", "classify", "translate", "validate"],
  "supported_domains": ["general", "legal", "medical", "finance", "scientific", "code", "conversational"],
  "supported_modalities": ["text", "structured"],
  "supported_languages": ["en", "fr", "de", "es", "pt", "ja", "zh"],
  "context_window_tokens": 200000,
  "max_output_tokens": 8192,
  "performance_profile": {
    "quality_scores": {
      "extract": 0.94,
      "summarize": 0.92,
      "generate": 0.90,
      "classify": 0.93,
      "translate": 0.88,
      "validate": 0.91
    },
    "domain_quality_scores": {
      "extract/legal": 0.97,
      "summarize/legal": 0.95,
      "extract/medical": 0.92
    },
    "p50_latency_ms": 820,
    "p95_latency_ms": 3400,
    "p99_latency_ms": 6100,
    "throughput_rpm": 2000,
    "calibration_sample_size": 50000,
    "calibration_timestamp_utc": "2026-05-01T00:00:00Z",
    "availability_slo": 0.9995
  },
  "compliance": {
    "certifications": ["SOC2", "ISO27001", "GDPR", "HIPAA"],
    "data_residency": {
      "processing_regions": ["US", "EU"],
      "storage_regions": ["US"],
      "cross_region_transfer": false
    },
    "pii_capable": true,
    "audit_logging": true,
    "zero_data_retention": true,
    "encryption_at_rest": true,
    "encryption_in_transit": true
  },
  "adapter_refs": {
    "python": {
      "package": "synapse-adapter-anthropic",
      "package_version": ">=1.2.0,<2.0.0",
      "class_path": "synapse_adapter_anthropic.ClaudeSonnetAdapter"
    },
    "typescript": {
      "package": "@synapse/adapter-anthropic",
      "package_version": "^1.4.0",
      "export_name": "ClaudeSonnetAdapter"
    }
  },
  "pricing": {
    "input_cost_per_1k_tokens": 0.003,
    "output_cost_per_1k_tokens": 0.015,
    "currency": "USD"
  }
}
```
