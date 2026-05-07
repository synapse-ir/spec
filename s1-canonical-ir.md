# S1 — Canonical Intermediate Representation (IR)

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

The Canonical IR is the single shared data structure that flows between every component in the SYNAPSE pipeline — routers, adapters, models, and compliance infrastructure. All messages entering or leaving the system are expressed as a Canonical IR document. Adapters translate model-specific formats to and from this structure; nothing else in the platform needs to know about vendor formats.

A Canonical IR document is a JSON object. It must conform to this specification at every hop.

---

## 2. Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `ir_version` | string | **Yes** | Semantic version of the IR schema in use (e.g. `"1.0.0"`). |
| `message_id` | string | **Yes** | Globally unique identifier for this IR document. Use UUIDv4. |
| `task_header` | object | **Yes** | Routing and classification metadata. See §3. |
| `payload` | object | **Yes** | The actual data being processed. See §4. |
| `provenance` | array\<ProvenanceEntry\> | **Yes** | Ordered log of all transforms applied to this IR. See §5. |
| `compliance_envelope` | object | **Yes** | Privacy, residency, and audit controls. See §6. |

All six fields must be present. An IR document missing any top-level field is invalid and must be rejected with error `IR_MISSING_REQUIRED_FIELD`.

### 2.1 Example top-level skeleton

```json
{
  "ir_version": "1.0.0",
  "message_id": "018e8b2a-7c1d-7f3a-a9e1-2d4f6b8c0e22",
  "task_header": { ... },
  "payload": { ... },
  "provenance": [ ... ],
  "compliance_envelope": { ... }
}
```

---

## 3. TaskHeader

The `task_header` object describes what the system is being asked to do and constrains how the router selects a model.

| Field | Type | Required | Description |
|---|---|---|---|
| `task_type` | string (enum) | **Yes** | The class of operation requested. See §3.1. |
| `domain` | string (enum) | **Yes** | Subject-matter domain. See §3.2. |
| `priority` | integer | No | Scheduling priority, 1 (lowest) – 10 (highest). Default: `5`. |
| `timeout_ms` | integer | No | Maximum wall-clock time in milliseconds the caller will wait. |
| `language` | string | No | BCP-47 language tag of the primary input content (e.g. `"en"`, `"fr-CA"`). |
| `model_hints` | array\<string\> | No | Ordered list of preferred model identifiers. Router may ignore if no compliant candidate is available. |
| `quality_floor` | number | No | Minimum acceptable quality score [0.0, 1.0]. Router must not route to a model whose calibrated score for this task falls below this value. |
| `cost_ceiling_usd` | number | No | Maximum acceptable per-call cost in USD. Router must not route to a model whose estimated cost exceeds this value. |
| `idempotency_key` | string | No | Caller-supplied key used for deduplication. If a response exists for this key in the result cache (C3), it is returned immediately. |

### 3.1 Valid `task_type` Values

| Value | Description |
|---|---|
| `classify` | Assign one or more labels to input content. |
| `extract` | Pull structured information from unstructured content. |
| `generate` | Produce new content given a prompt or context. |
| `summarize` | Produce a condensed representation of input content. |
| `embed` | Produce a dense vector representation of input content. |
| `rank` | Order a set of candidates by relevance to a query. |
| `validate` | Check content against a schema, rule set, or factual standard. |
| `translate` | Convert content from one language to another. |
| `score` | Assign a numeric score to content along one or more dimensions. |

New values may only be added through the process described in [CONTRIBUTING.md](../CONTRIBUTING.md).

### 3.2 Valid `domain` Values

| Value | Description |
|---|---|
| `general` | No domain specialization required. |
| `legal` | Legal documents, contracts, statutes, case law. |
| `medical` | Clinical notes, patient records, biomedical literature. |
| `finance` | Financial reports, market data, accounting documents. |
| `code` | Source code, configuration files, technical documentation. |
| `scientific` | Research papers, experimental data, technical datasets. |
| `multilingual` | Content spanning multiple languages or requiring cross-lingual reasoning. |
| `conversational` | Dialogue, chat, customer support interactions. |

---

## 4. Payload

The `payload` object carries the data to be processed. Exactly one modality key must be present.

| Field | Type | Required | Description |
|---|---|---|---|
| `modality` | string (enum) | **Yes** | Must be one of: `text`, `embedding`, `structured`, `binary`. |
| `text` | TextPayload | Conditional | Present when `modality` is `"text"`. |
| `embedding` | EmbeddingPayload | Conditional | Present when `modality` is `"embedding"`. |
| `structured` | StructuredPayload | Conditional | Present when `modality` is `"structured"`. |
| `binary` | BinaryPayload | Conditional | Present when `modality` is `"binary"`. |
| `metadata` | object | No | Free-form key-value pairs for caller-defined payload metadata. |

### 4.1 TextPayload

| Field | Type | Required | Description |
|---|---|---|---|
| `content` | string | **Yes** | The raw text content. |
| `content_type` | string | No | MIME type of the content (e.g. `"text/plain"`, `"text/html"`, `"text/markdown"`). Default: `"text/plain"`. |
| `truncation_strategy` | string | No | How to handle content exceeding model context: `"head"`, `"tail"`, `"middle"`. Default: `"tail"`. |
| `max_tokens` | integer | No | Maximum number of tokens to generate in the response. |
| `system_prompt` | string | No | System-level instruction to prepend before the user content. |
| `few_shot_examples` | array\<object\> | No | List of `{input, output}` pairs for few-shot prompting. |

### 4.2 EmbeddingPayload

| Field | Type | Required | Description |
|---|---|---|---|
| `inputs` | array\<string\> | **Yes** | Ordered list of strings to embed. At least one element required. |
| `dimensions` | integer | No | Requested embedding dimensionality. Model may ignore if unsupported. |
| `normalize` | boolean | No | Whether to L2-normalize output vectors. Default: `false`. |

### 4.3 StructuredPayload

| Field | Type | Required | Description |
|---|---|---|---|
| `schema_uri` | string | **Yes** | URI identifying the schema that `data` conforms to. |
| `data` | object | **Yes** | The structured data object. |
| `output_schema_uri` | string | No | URI of the schema the model's output must conform to. |

### 4.4 BinaryPayload

| Field | Type | Required | Description |
|---|---|---|---|
| `content_type` | string | **Yes** | MIME type of the binary content (e.g. `"application/pdf"`, `"image/png"`). |
| `encoding` | string | **Yes** | Encoding of the `data` field. Must be `"base64"`. |
| `data` | string | **Yes** | Base64-encoded binary content. |
| `size_bytes` | integer | No | Unencoded size in bytes. Used for routing decisions. |
| `filename` | string | No | Original filename hint for the model. |

---

## 5. ProvenanceEntry

The `provenance` field is an ordered array of `ProvenanceEntry` objects. Each transform applied to the IR — ingress, model call, egress, enrichment — must append an entry. The array grows monotonically; entries must never be removed or reordered.

| Field | Type | Required | Description |
|---|---|---|---|
| `step` | string | **Yes** | Identifier for this processing step (e.g. `"ingress"`, `"router"`, `"model_call"`, `"egress"`). |
| `component_id` | string | **Yes** | Unique identifier of the component that performed this step (adapter ID, model ID, router instance ID). |
| `component_version` | string | **Yes** | Semantic version of the component at time of processing. |
| `timestamp_utc` | string | **Yes** | ISO 8601 UTC timestamp of when this step completed (e.g. `"2026-05-06T14:23:01.042Z"`). |
| `latency_ms` | integer | No | Wall-clock duration of this step in milliseconds. |
| `model_id` | string | No | Fully-qualified model identifier, if this step involved a model call. |
| `input_tokens` | integer | No | Number of input tokens consumed at this step. |
| `output_tokens` | integer | No | Number of output tokens produced at this step. |
| `cost_usd` | number | No | Computed cost of this step in USD. |
| `cache_hit` | boolean | No | Whether this step was served from a cache. |
| `cache_layer` | string | No | Which cache layer was hit (e.g. `"C3"`, `"C4"`). See S8. |
| `transform_digest` | string | No | SHA-256 of the component's deterministic transform function, for audit purposes. |

---

## 6. ComplianceEnvelope

The `compliance_envelope` carries privacy, residency, and audit controls that the router and all downstream components must honor.

| Field | Type | Required | Description |
|---|---|---|---|
| `data_classification` | string | **Yes** | Sensitivity level: `"public"`, `"internal"`, `"confidential"`, `"restricted"`. |
| `pii_present` | boolean | **Yes** | Whether the payload contains personally identifiable information. |
| `retention_policy` | string | No | Named retention policy to apply (e.g. `"30d"`, `"1y"`, `"indefinite"`). Default: `"30d"`. |
| `allowed_regions` | array\<string\> | No | ISO 3166-1 alpha-2 country codes where data may be processed. Empty list means no restriction. |
| `denied_regions` | array\<string\> | No | ISO 3166-1 alpha-2 country codes where data must not be processed. |
| `compliance_tags` | array\<string\> | No | Named compliance frameworks that apply (e.g. `"HIPAA"`, `"GDPR"`, `"SOC2"`, `"FedRAMP"`). |
| `audit_required` | boolean | No | Whether this request must produce a durable audit log entry. Default: `false`. |
| `audit_trail_id` | string | No | Identifier linking this request to an external audit trail. |
| `anonymization_applied` | boolean | No | Whether PII was anonymized before this IR was created. |
| `consent_reference` | string | No | Reference to the data subject consent record, if applicable. |

---

## 7. Complete Example — Legal Extraction Task

The following is a valid Canonical IR for a task that extracts party names and obligations from a legal contract.

```json
{
  "ir_version": "1.0.0",
  "message_id": "018e8b2a-7c1d-7f3a-a9e1-2d4f6b8c0e22",
  "task_header": {
    "task_type": "extract",
    "domain": "legal",
    "priority": 7,
    "timeout_ms": 30000,
    "language": "en",
    "model_hints": ["gpt-4o", "claude-opus-4-7"],
    "quality_floor": 0.85,
    "cost_ceiling_usd": 0.05,
    "idempotency_key": "contract-extract-acme-msft-2026-001"
  },
  "payload": {
    "modality": "text",
    "text": {
      "content": "This Master Services Agreement ('Agreement') is entered into as of January 1, 2026, by and between Acme Corporation, a Delaware corporation ('Client'), and Contoso Ltd., a UK private limited company ('Vendor'). Vendor shall deliver the software services described in Exhibit A within ninety (90) days of the Effective Date. Client shall pay Vendor $250,000 USD within thirty (30) days of invoice receipt.",
      "content_type": "text/plain",
      "system_prompt": "You are a legal extraction engine. Extract all parties, obligations, deadlines, and monetary amounts. Return structured JSON conforming to the provided output schema.",
      "max_tokens": 1024
    },
    "metadata": {
      "source_document": "acme-contoso-msa-2026.pdf",
      "page_range": "1-3"
    }
  },
  "provenance": [
    {
      "step": "ingress",
      "component_id": "synapse-pdf-adapter",
      "component_version": "2.1.0",
      "timestamp_utc": "2026-05-06T14:23:00.100Z",
      "latency_ms": 45,
      "cache_hit": false
    }
  ],
  "compliance_envelope": {
    "data_classification": "confidential",
    "pii_present": true,
    "retention_policy": "1y",
    "allowed_regions": ["US", "GB"],
    "denied_regions": [],
    "compliance_tags": ["GDPR", "SOC2"],
    "audit_required": true,
    "audit_trail_id": "aud-2026-05-acme-001",
    "anonymization_applied": false,
    "consent_reference": "consent-acme-dpa-2025-11"
  }
}
```

---

## 8. Schema Validation

Implementations must validate every IR document against this specification at ingress and at egress. The full JSON Schema is published at:

```
https://schemas.synapse.internal/ir/v1.0.0/canonical-ir.schema.json
```

Validation errors must be surfaced as structured error responses. See [S6 — Error Codes](s6-errors.md) for the complete list.
