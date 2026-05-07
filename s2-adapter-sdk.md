# S2 — Adapter SDK

**SYNAPSE Specification · Version 1.0.0**

---

## 1. What Is an Adapter?

An adapter is a pair of **pure, stateless functions** that translate between the Canonical IR and a specific model's native input/output format. Adapters have no side effects, hold no mutable state, and make no network calls.

```
Canonical IR  ──[ingress]──►  ModelInput
ModelOutput   ──[egress]───►  Canonical IR  (updated)
```

Because adapters are stateless, they are safe to execute concurrently, are trivially testable, and can be hot-swapped without draining in-flight requests.

**What adapters must not do:**

- Maintain instance-level state between calls.
- Call the model directly (the router handles dispatch).
- Modify `compliance_envelope` fields (this is the router's responsibility).
- Remove or reorder entries in `provenance`.

---

## 2. The Adapter Contract

### 2.1 Ingress — `IR → ModelInput`

The ingress function receives a fully-validated Canonical IR document and returns the model's native input structure.

**Signature (abstract):**

```
ingress(ir: CanonicalIR) -> ModelInput
```

**Responsibilities:**

1. Read `task_header`, `payload`, and `compliance_envelope` to build the model-specific prompt or input structure.
2. Apply any model-specific token limits, formatting, or templating.
3. Must be a pure function — given the same IR, always return the same ModelInput.
4. Must not mutate the IR.
5. Must raise `AdapterValidationError` (see §4) if the IR cannot be mapped to a valid ModelInput.

### 2.2 Egress — `(ModelOutput, IR) → IR`

The egress function receives the model's raw output and the original Canonical IR, and returns a new Canonical IR with the result embedded in `payload` and a new entry appended to `provenance`.

**Signature (abstract):**

```
egress(output: ModelOutput, original_ir: CanonicalIR) -> CanonicalIR
```

**Responsibilities:**

1. Parse and validate the model's raw output.
2. Write the result into the appropriate `payload` modality field.
3. Append a `ProvenanceEntry` for the `model_call` step (see S1 §5).
4. Preserve all other fields from `original_ir` unchanged.
5. Must not alter `message_id`, `task_header`, or `compliance_envelope`.
6. Must raise `AdapterValidationError` if the model output is unparseable or violates the output schema.

---

## 3. Python Adapter SDK

### 3.1 `AdapterBase` Abstract Class

```python
from __future__ import annotations

import abc
import datetime
from typing import Any

from synapse.ir import CanonicalIR, ProvenanceEntry


class AdapterBase(abc.ABC):
    """
    Base class for all Python SYNAPSE adapters.

    Subclasses must implement `ingress` and `egress`.
    Both methods must be pure and stateless.
    """

    #: Unique identifier for this adapter, used in provenance entries.
    adapter_id: str
    #: Semantic version of this adapter implementation.
    adapter_version: str

    @abc.abstractmethod
    def ingress(self, ir: CanonicalIR) -> dict[str, Any]:
        """
        Translate a Canonical IR into the model's native input format.

        Args:
            ir: A fully-validated Canonical IR document.

        Returns:
            A dict representing the model's native API request payload.

        Raises:
            AdapterValidationError: If the IR cannot be translated.
        """

    @abc.abstractmethod
    def egress(self, output: dict[str, Any], original_ir: CanonicalIR) -> CanonicalIR:
        """
        Translate model output back into an updated Canonical IR.

        Args:
            output: Raw response from the model's API.
            original_ir: The Canonical IR that was used to generate `output`.

        Returns:
            A new CanonicalIR with the result in `payload` and a new
            ProvenanceEntry appended.

        Raises:
            AdapterValidationError: If `output` cannot be parsed or is invalid.
        """

    def _make_provenance_entry(
        self,
        model_id: str,
        latency_ms: int,
        input_tokens: int | None = None,
        output_tokens: int | None = None,
        cost_usd: float | None = None,
        cache_hit: bool = False,
    ) -> ProvenanceEntry:
        return ProvenanceEntry(
            step="model_call",
            component_id=self.adapter_id,
            component_version=self.adapter_version,
            timestamp_utc=datetime.datetime.utcnow().isoformat() + "Z",
            latency_ms=latency_ms,
            model_id=model_id,
            input_tokens=input_tokens,
            output_tokens=output_tokens,
            cost_usd=cost_usd,
            cache_hit=cache_hit,
        )
```

### 3.2 Complete Example — NER Adapter (OpenAI)

This adapter handles `task_type: "extract"` for named-entity recognition using the OpenAI chat completions API.

```python
from __future__ import annotations

import json
import time
from typing import Any

from synapse.ir import CanonicalIR, ProvenanceEntry
from synapse.adapter import AdapterBase, AdapterValidationError


NER_SYSTEM_PROMPT = (
    "You are a named-entity recognition engine. "
    "Extract all entities from the user's text and return a JSON object "
    "with a single key 'entities', whose value is a list of objects each "
    "containing 'text', 'label', and 'start_char', 'end_char' fields."
)

SUPPORTED_TASK_TYPES = {"extract", "classify"}


class OpenAINERAdapter(AdapterBase):
    adapter_id = "openai-ner-adapter"
    adapter_version = "1.0.0"

    def ingress(self, ir: CanonicalIR) -> dict[str, Any]:
        if ir.task_header.task_type not in SUPPORTED_TASK_TYPES:
            raise AdapterValidationError(
                f"OpenAINERAdapter does not support task_type "
                f"'{ir.task_header.task_type}'. "
                f"Supported: {SUPPORTED_TASK_TYPES}"
            )

        if ir.payload.modality != "text":
            raise AdapterValidationError(
                f"OpenAINERAdapter requires modality 'text', "
                f"got '{ir.payload.modality}'."
            )

        text_payload = ir.payload.text
        system_prompt = text_payload.system_prompt or NER_SYSTEM_PROMPT

        messages = [
            {"role": "system", "content": system_prompt},
            {"role": "user", "content": text_payload.content},
        ]

        request: dict[str, Any] = {
            "model": "gpt-4o",
            "messages": messages,
            "response_format": {"type": "json_object"},
        }

        if text_payload.max_tokens:
            request["max_tokens"] = text_payload.max_tokens

        return request

    def egress(self, output: dict[str, Any], original_ir: CanonicalIR) -> CanonicalIR:
        try:
            choice = output["choices"][0]
            raw_content = choice["message"]["content"]
            parsed = json.loads(raw_content)
        except (KeyError, IndexError, json.JSONDecodeError) as exc:
            raise AdapterValidationError(
                f"Failed to parse model output: {exc}"
            ) from exc

        if "entities" not in parsed:
            raise AdapterValidationError(
                "Model output missing required 'entities' key."
            )

        usage = output.get("usage", {})
        input_tokens = usage.get("prompt_tokens")
        output_tokens = usage.get("completion_tokens")

        provenance_entry = self._make_provenance_entry(
            model_id=output.get("model", "gpt-4o"),
            latency_ms=output.get("_synapse_latency_ms", 0),
            input_tokens=input_tokens,
            output_tokens=output_tokens,
        )

        updated_ir = original_ir.model_copy(deep=True)
        updated_ir.payload.modality = "structured"
        updated_ir.payload.structured = {
            "schema_uri": "https://schemas.synapse.internal/ner/v1/output.json",
            "data": parsed,
        }
        updated_ir.provenance.append(provenance_entry)

        return updated_ir
```

---

## 4. TypeScript Adapter SDK

### 4.1 `SynapseAdapter` Interface

```typescript
import type { CanonicalIR, ProvenanceEntry } from "@synapse/ir";

/**
 * All SYNAPSE TypeScript adapters must implement this interface.
 * Both methods must be pure and stateless.
 */
export interface SynapseAdapter<TInput = unknown, TOutput = unknown> {
  /** Unique identifier for this adapter, written to provenance entries. */
  readonly adapterId: string;
  /** Semantic version of this adapter implementation. */
  readonly adapterVersion: string;

  /**
   * Translate a Canonical IR into the model's native input format.
   * Must be a pure function.
   *
   * @throws {AdapterValidationError} if the IR cannot be translated.
   */
  ingress(ir: CanonicalIR): TInput;

  /**
   * Translate model output back into an updated Canonical IR.
   * Must append exactly one ProvenanceEntry and must not mutate originalIr.
   *
   * @throws {AdapterValidationError} if output cannot be parsed.
   */
  egress(output: TOutput, originalIr: CanonicalIR): CanonicalIR;
}

export class AdapterValidationError extends Error {
  constructor(
    message: string,
    public readonly code: string = "ADAPTER_VALIDATION_ERROR"
  ) {
    super(message);
    this.name = "AdapterValidationError";
  }
}
```

### 4.2 Complete Example — Summarization Adapter (Anthropic)

```typescript
import Anthropic from "@anthropic-ai/sdk";
import type { MessageParam } from "@anthropic-ai/sdk/resources/messages";
import type { CanonicalIR } from "@synapse/ir";
import {
  AdapterValidationError,
  type SynapseAdapter,
} from "@synapse/adapter-sdk";

type AnthropicRequest = {
  model: string;
  max_tokens: number;
  system?: string;
  messages: MessageParam[];
};

type AnthropicResponse = Awaited<ReturnType<Anthropic["messages"]["create"]>> & {
  _synapseLatencyMs?: number;
};

const SUPPORTED_TASK_TYPES = new Set(["summarize", "generate"]);
const DEFAULT_MAX_TOKENS = 512;

export class AnthropicSummarizeAdapter
  implements SynapseAdapter<AnthropicRequest, AnthropicResponse>
{
  readonly adapterId = "anthropic-summarize-adapter";
  readonly adapterVersion = "1.0.0";

  ingress(ir: CanonicalIR): AnthropicRequest {
    if (!SUPPORTED_TASK_TYPES.has(ir.taskHeader.taskType)) {
      throw new AdapterValidationError(
        `AnthropicSummarizeAdapter does not support task_type ` +
          `'${ir.taskHeader.taskType}'. ` +
          `Supported: ${[...SUPPORTED_TASK_TYPES].join(", ")}`
      );
    }

    if (ir.payload.modality !== "text") {
      throw new AdapterValidationError(
        `AnthropicSummarizeAdapter requires modality 'text', ` +
          `got '${ir.payload.modality}'.`
      );
    }

    const { content, systemPrompt, maxTokens } = ir.payload.text!;

    const request: AnthropicRequest = {
      model: "claude-sonnet-4-6",
      max_tokens: maxTokens ?? DEFAULT_MAX_TOKENS,
      messages: [{ role: "user", content }],
    };

    if (systemPrompt) {
      request.system = systemPrompt;
    }

    return request;
  }

  egress(output: AnthropicResponse, originalIr: CanonicalIR): CanonicalIR {
    const textBlock = output.content.find((b) => b.type === "text");
    if (!textBlock || textBlock.type !== "text") {
      throw new AdapterValidationError(
        "Model output contained no text block."
      );
    }

    const provenanceEntry = {
      step: "model_call",
      componentId: this.adapterId,
      componentVersion: this.adapterVersion,
      timestampUtc: new Date().toISOString(),
      latencyMs: output._synapseLatencyMs,
      modelId: output.model,
      inputTokens: output.usage.input_tokens,
      outputTokens: output.usage.output_tokens,
      cacheHit: false,
    };

    return {
      ...originalIr,
      payload: {
        modality: "text",
        text: {
          content: textBlock.text,
          contentType: "text/plain",
        },
      },
      provenance: [...originalIr.provenance, provenanceEntry],
    };
  }
}
```

---

## 5. Adapter Validation Rules

All adapter implementations must be validated against these rules. Rules marked **MUST** are mandatory; violations result in rejection. Rules marked **SHOULD** are strong recommendations; violations trigger a warning in the registry.

| # | Severity | Rule |
|---|---|---|
| AV-01 | **MUST** | `ingress` must be a pure function: identical inputs must produce identical outputs. |
| AV-02 | **MUST** | `egress` must not mutate `originalIr`; it must return a new object. |
| AV-03 | **MUST** | `egress` must append exactly one new `ProvenanceEntry` with `step = "model_call"`. |
| AV-04 | **MUST** | `egress` must preserve `message_id`, `task_header`, and `compliance_envelope` unchanged. |
| AV-05 | **MUST** | Both functions must complete without network I/O. Network calls belong in the model call layer. |
| AV-06 | **MUST** | Both functions must not access global mutable state. |
| AV-07 | **MUST** | `AdapterValidationError` (or equivalent) must be raised for untranslatable inputs or unparseable outputs. Unhandled exceptions are not acceptable. |
| AV-08 | **MUST** | `adapter_id` must be unique within the registry. Attempting to register a duplicate ID is an error. |
| AV-09 | **MUST** | `adapter_version` must follow semantic versioning (`MAJOR.MINOR.PATCH`). |
| AV-10 | **SHOULD** | `ingress` should validate the `task_type` against the set of task types the adapter was designed to handle and raise `AdapterValidationError` for unsupported types. |
| AV-11 | **SHOULD** | `ingress` should validate the `payload.modality` and raise `AdapterValidationError` for unsupported modalities. |
| AV-12 | **SHOULD** | `egress` should populate `input_tokens`, `output_tokens`, and `cost_usd` in the provenance entry when the model API provides usage data. |
| AV-13 | **SHOULD** | Both functions should complete in under 50 ms on a reference machine (exclude network time). Adapters that consistently exceed this threshold will be flagged during registry review. |
