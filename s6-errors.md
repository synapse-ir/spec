# S6 — Error Codes

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

All SYNAPSE components report errors using structured error objects. Every error has a machine-readable `code`, a human-readable `message`, and an optional `details` field for structured context.

**Standard error envelope:**

```json
{
  "error": "IR_MISSING_REQUIRED_FIELD",
  "message": "The 'task_header' field is required but was not present.",
  "details": {
    "missing_field": "task_header"
  },
  "request_id": "req-018e9b1a-4c3d-7e2f-a1b2-3c4d5e6f7a8b",
  "timestamp_utc": "2026-05-06T14:23:01.042Z"
}
```

---

## 2. IR Validation Error Codes

These errors are raised when a Canonical IR document fails validation. They are surfaced at ingress (adapter input validation) and at egress (adapter output validation).

| Code | HTTP Equivalent | Description |
|---|---|---|
| `IR_MISSING_REQUIRED_FIELD` | 400 Bad Request | A required top-level field (`ir_version`, `message_id`, `task_header`, `payload`, `provenance`, `compliance_envelope`) is absent. |
| `IR_INVALID_VERSION` | 400 Bad Request | `ir_version` is present but is not a valid semantic version string, or the version is not supported by this component. |
| `IR_INVALID_MESSAGE_ID` | 400 Bad Request | `message_id` is present but is not a valid UUIDv4. |
| `IR_INVALID_TASK_TYPE` | 400 Bad Request | `task_header.task_type` is not one of the valid enum values defined in S1 §3.1. |
| `IR_UNSUPPORTED_TASK_TYPE` | 422 Unprocessable Entity | `task_header.task_type` is a valid enum value, but this component does not support it. |
| `IR_INVALID_DOMAIN` | 400 Bad Request | `task_header.domain` is not one of the valid enum values defined in S1 §3.2. |
| `IR_UNSUPPORTED_DOMAIN` | 422 Unprocessable Entity | `task_header.domain` is a valid enum value, but this component does not support it. |
| `IR_INVALID_PRIORITY` | 400 Bad Request | `task_header.priority` is present but is outside the valid range [1, 10]. |
| `IR_INVALID_QUALITY_FLOOR` | 400 Bad Request | `task_header.quality_floor` is present but is outside the range [0.0, 1.0]. |
| `IR_INVALID_MODALITY` | 400 Bad Request | `payload.modality` is not one of: `text`, `embedding`, `structured`, `binary`. |
| `IR_MISSING_MODALITY_BODY` | 400 Bad Request | `payload.modality` is set to a value (e.g. `"text"`) but the corresponding modality field (`payload.text`) is absent. |
| `IR_MULTIPLE_MODALITY_BODIES` | 400 Bad Request | More than one modality body field is present in `payload`. Exactly one must be present. |
| `IR_EMPTY_EMBEDDING_INPUTS` | 400 Bad Request | `payload.embedding.inputs` is present but contains zero elements. |
| `IR_MISSING_SCHEMA_URI` | 400 Bad Request | `payload.structured.schema_uri` is required for structured payloads but is absent. |
| `IR_INVALID_BINARY_ENCODING` | 400 Bad Request | `payload.binary.encoding` is present but is not `"base64"`. |
| `IR_INVALID_BASE64` | 400 Bad Request | `payload.binary.data` is not valid Base64. |
| `IR_INVALID_DATA_CLASSIFICATION` | 400 Bad Request | `compliance_envelope.data_classification` is not one of: `public`, `internal`, `confidential`, `restricted`. |
| `IR_INVALID_REGION_CODE` | 400 Bad Request | A region code in `allowed_regions` or `denied_regions` is not a valid ISO 3166-1 alpha-2 code. |
| `IR_REGION_CONFLICT` | 400 Bad Request | A region code appears in both `allowed_regions` and `denied_regions`. |
| `IR_EMPTY_PROVENANCE` | 400 Bad Request | The `provenance` array is present but contains no entries. At least the ingress entry must be present. |
| `IR_INVALID_PROVENANCE_ENTRY` | 400 Bad Request | A `ProvenanceEntry` is missing a required field (`step`, `component_id`, `component_version`, `timestamp_utc`). |
| `IR_INVALID_TIMESTAMP` | 400 Bad Request | A timestamp field is present but is not a valid ISO 8601 UTC string. |
| `IR_VERSION_INCOMPATIBLE` | 422 Unprocessable Entity | The `ir_version` in the document is outside the range this component supports. |

---

## 3. Registry API Error Codes

These errors are returned by the Registry API (see S4).

| Code | HTTP Equivalent | Description |
|---|---|---|
| `MODEL_NOT_FOUND` | 404 Not Found | No model with the specified `model_id` is registered in the registry. |
| `MODEL_ALREADY_EXISTS` | 409 Conflict | A model with the specified `model_id` is already registered. Use `PUT` to update. |
| `MANIFEST_VALIDATION_ERROR` | 400 Bad Request | The submitted manifest does not conform to the manifest schema (see S3). |
| `MANIFEST_VERSION_MISMATCH` | 400 Bad Request | The `model_id` in the manifest body does not match the `{model_id}` path parameter. |
| `UNSUPPORTED_TASK_TYPE` | 400 Bad Request | The manifest declares a `task_type` that is not in the registry's known enum list. |
| `UNSUPPORTED_DOMAIN` | 400 Bad Request | The manifest declares a `domain` that is not in the registry's known enum list. |
| `ROUTE_NO_CANDIDATES` | 422 Unprocessable Entity | No registered model satisfies all routing constraints. See S4 §4.4 for the full response schema including `relaxation_suggestions`. |
| `ROUTE_CONSTRAINT_CONFLICT` | 400 Bad Request | The routing request contains contradictory constraints (e.g. a region appears in both `allowed_regions` and `denied_regions`). |
| `CALIBRATION_UNKNOWN_TRACE` | 404 Not Found | The `routing_trace_id` in the calibration signal does not correspond to a known routing decision. |
| `CALIBRATION_DUPLICATE_SIGNAL` | 409 Conflict | A calibration signal for this `routing_trace_id` has already been received. |
| `ADAPTER_NOT_FOUND` | 404 Not Found | An `adapter_ref` references a package or class that cannot be resolved. |
| `ADAPTER_VALIDATION_ERROR` | 422 Unprocessable Entity | An adapter implementation failed one or more validation rules defined in S2 §5. |
| `UNAUTHORIZED` | 401 Unauthorized | The request did not include a valid `Authorization` header or client certificate. |
| `FORBIDDEN` | 403 Forbidden | The authenticated principal does not have the required scope for this operation. |
| `RATE_LIMITED` | 429 Too Many Requests | The caller has exceeded the rate limit for this endpoint. The `Retry-After` response header contains the number of seconds to wait. |
| `REGISTRY_UNAVAILABLE` | 503 Service Unavailable | The registry is temporarily unable to serve requests. |
| `INTERNAL_ERROR` | 500 Internal Server Error | An unexpected error occurred. The `details.incident_id` field contains an identifier for support escalation. |

---

## 4. Adapter Error Codes

These errors are raised by adapter implementations and surfaced to the routing layer.

| Code | HTTP Equivalent | Description |
|---|---|---|
| `ADAPTER_VALIDATION_ERROR` | 422 Unprocessable Entity | The IR cannot be translated to a valid model input (raised in `ingress`). |
| `ADAPTER_OUTPUT_PARSE_ERROR` | 502 Bad Gateway | The model's output could not be parsed by the adapter (raised in `egress`). |
| `ADAPTER_OUTPUT_SCHEMA_VIOLATION` | 502 Bad Gateway | The model's output was parseable but does not conform to the expected output schema (raised in `egress`). |
| `ADAPTER_TIMEOUT` | 504 Gateway Timeout | The adapter's ingress or egress function exceeded the 50 ms processing budget. |

---

## 5. Recommended Retry Policy

### 5.1 Retryable vs. Non-Retryable Errors

Not all errors are worth retrying. Retrying non-retryable errors wastes resources and increases latency.

| Category | Codes | Retryable? |
|---|---|---|
| Client validation errors | `IR_*`, `MANIFEST_VALIDATION_ERROR`, `ROUTE_CONSTRAINT_CONFLICT`, `ADAPTER_VALIDATION_ERROR` | **No** — the request is malformed. Fix the request. |
| Not found | `MODEL_NOT_FOUND`, `CALIBRATION_UNKNOWN_TRACE`, `ADAPTER_NOT_FOUND` | **No** — the resource does not exist. |
| Conflict | `MODEL_ALREADY_EXISTS`, `CALIBRATION_DUPLICATE_SIGNAL` | **No** — a state conflict. Resolve before retrying. |
| Authorization | `UNAUTHORIZED`, `FORBIDDEN` | **No** — a credential or permission issue. |
| No candidates | `ROUTE_NO_CANDIDATES` | **Conditionally** — only after relaxing a constraint as suggested. |
| Rate limiting | `RATE_LIMITED` | **Yes** — after the `Retry-After` interval. |
| Transient server errors | `REGISTRY_UNAVAILABLE`, `INTERNAL_ERROR` | **Yes** — with exponential backoff. |
| Gateway errors | `ADAPTER_OUTPUT_PARSE_ERROR`, `ADAPTER_OUTPUT_SCHEMA_VIOLATION`, `ADAPTER_TIMEOUT` | **Yes** — route to an alternative model. |

### 5.2 Exponential Backoff Algorithm

For retryable errors, use truncated binary exponential backoff with jitter:

```
delay(attempt) = min(base_ms * 2^attempt, cap_ms) + jitter(jitter_max_ms)
```

**Recommended parameters:**

| Parameter | Value |
|---|---|
| `base_ms` | 100 ms |
| `cap_ms` | 30,000 ms (30 seconds) |
| `jitter_max_ms` | 250 ms |
| `max_attempts` | 5 |

**Example delays (without jitter):**

| Attempt | Delay |
|---|---|
| 1 (first retry) | 200 ms |
| 2 | 400 ms |
| 3 | 800 ms |
| 4 | 1,600 ms |
| 5 | 3,200 ms |

After `max_attempts` retries without success, surface the error to the caller.

### 5.3 Rate-Limited Retry

When the response includes a `Retry-After` header, use that value exactly — do not apply exponential backoff on top of it. The header value is always in seconds.

### 5.4 Alternative Model Retry

For gateway errors (`ADAPTER_OUTPUT_PARSE_ERROR`, `ADAPTER_TIMEOUT`), prefer routing to an alternative model (from the `alternatives` array in the routing response) over retrying the same model. Only fall back to same-model retry if no alternative is available.

### 5.5 Idempotency

Always set `task_header.idempotency_key` on requests you may retry. This ensures that if a request succeeded but the response was lost in transit, a retry returns the cached result rather than processing the request again (cache layer C3).
