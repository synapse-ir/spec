# S5 — Versioning Policy

**SYNAPSE Specification · Version 1.0.0**

---

## 1. Overview

SYNAPSE uses semantic versioning (`MAJOR.MINOR.PATCH`) for all versioned artifacts: the Canonical IR schema, the adapter SDK, and capability manifests. This document defines what constitutes each type of change and how compatibility is maintained across versions.

---

## 2. IR Version Policy

The `ir_version` field in every Canonical IR document identifies the schema version in use. All components must declare which IR versions they support.

### 2.1 PATCH Release (`x.y.Z`)

A PATCH release corrects errors in the specification without changing the schema.

**Triggers for a PATCH release:**

- Typo or editorial correction in field descriptions.
- Clarification of ambiguous normative language that does not change the intent.
- Addition of a non-normative example.
- Correction of an incorrect constraint that was never implemented (e.g., a documented minimum that was always enforced differently).

**Compatibility:** Fully backward and forward compatible. No consumer or producer changes required.

### 2.2 MINOR Release (`x.Y.z`)

A MINOR release adds new optional capabilities without breaking existing consumers.

**Triggers for a MINOR release:**

- Addition of a new **optional** top-level field.
- Addition of a new **optional** field within an existing object.
- Addition of a new valid `task_type` value (see §5).
- Addition of a new valid `domain` value (see §5).
- Addition of a new valid `modality` under `payload`.
- Addition of a new optional field to `ProvenanceEntry` or `ComplianceEnvelope`.
- Relaxation of a constraint (e.g., changing a field from required to optional).

**Compatibility:** Backward compatible. Producers on the new MINOR version may produce documents that older consumers cannot fully interpret, but older consumers must not reject documents with unrecognized optional fields (see §4 — Forward Compatibility).

### 2.3 MAJOR Release (`X.y.z`)

A MAJOR release introduces breaking changes.

**Triggers for a MAJOR release:**

- Removal of any field.
- Renaming of any field.
- Changing the type of any field (e.g., string to object).
- Making an optional field required.
- Changing the semantics of an existing field in a way that breaks existing consumers.
- Removing a valid enum value from `task_type` or `domain`.
- Adding a new **required** top-level field.

**Compatibility:** Not backward compatible. A migration plan must accompany all MAJOR releases. The registry must support both the previous and new MAJOR version concurrently for a minimum of 90 days during the transition window.

---

## 3. Adapter Version Policy

Adapter versions are independent of the IR version but must declare IR compatibility.

### 3.1 PATCH Release

- Bug fix that does not change observable behavior for any valid input.
- Improvement to error messages.
- Performance optimization with no behavioral change.

### 3.2 MINOR Release

- Support for a new optional IR field.
- Support for a new `task_type` or `domain` value.
- Addition of a new optional configuration parameter.
- New handling for an edge case that was previously undefined behavior.

### 3.3 MAJOR Release

- Change to the ingress output format in a way that breaks the model's API contract (e.g., restructuring the prompt template).
- Change to the egress output format that changes what fields are populated in the returned IR.
- Dropping support for a `task_type` or `domain` that was previously supported.
- Changing the `adapter_id` (treated as a new adapter, not a new version of the existing one).

### 3.4 IR Compatibility Declaration

Adapter manifests must declare IR compatibility as a semver range:

```json
{
  "adapter_id": "openai-ner-adapter",
  "adapter_version": "2.1.0",
  "ir_compatibility": ">=1.0.0 <2.0.0"
}
```

The runtime must reject an adapter whose `ir_compatibility` range does not include the `ir_version` of the IR being processed.

---

## 4. Forward Compatibility Rules

Forward compatibility — the ability of an older consumer to process a document produced under a newer MINOR version — is mandatory for all SYNAPSE components.

### Rule FC-01 — Ignore Unknown Optional Fields

Any consumer receiving a Canonical IR document must silently ignore fields it does not recognize, provided those fields are at the MINOR-version boundary (i.e., they were added in a MINOR release and are optional). Consumers must not raise an error for unrecognized optional fields.

### Rule FC-02 — Pass Through Unknown Fields

Adapters and routers acting as pass-through components must propagate unrecognized optional fields unchanged. They must not strip fields they do not understand.

### Rule FC-03 — Unknown Enum Values

A consumer encountering an unrecognized `task_type` or `domain` value (added in a MINOR release) must not silently process the request as if the field were absent. It must either:

1. Reject the request with error `UNSUPPORTED_TASK_TYPE` or `UNSUPPORTED_DOMAIN` (preferred for adapters that are not designed to handle the value), or
2. Route around the component to one that does support the value.

### Rule FC-04 — Schema Version Comparison

Components must compare `ir_version` using semantic version ordering. A component that supports `1.2.0` must accept documents with `ir_version: "1.0.0"` through `"1.2.x"`. It must not accept documents with `ir_version: "2.0.0"`.

### Rule FC-05 — Provenance Array Ordering

The `provenance` array is append-only and must be treated as an immutable ordered log. Forward-compatible consumers must be prepared for the array to contain entries with `step` values or fields they do not recognize.

---

## 5. Registering New `task_type` or `domain` Values

New enum values must go through a formal process to ensure ecosystem-wide consistency. Adding a value to `task_type` or `domain` is a MINOR IR change and follows the process below.

### 5.1 Process

1. **Open a proposal.** Create a pull request against the SYNAPSE specification repository. The PR must include:
   - The proposed value (all lowercase, underscore-separated if multi-word).
   - A clear definition of the value's semantics.
   - At least two concrete examples of use cases that the new value covers.
   - Evidence that the use case is not adequately served by an existing value or a combination of existing values.
   - A list of known adapters or models that would implement support for the new value.

2. **Review period.** The proposal enters a 14-day review period. See [CONTRIBUTING.md](../CONTRIBUTING.md) for reviewer assignment and approval requirements.

3. **Specification amendment.** Upon approval, the specification is updated in a MINOR release. The value is added to the appropriate table in S1 §3.1 or §3.2 and the JSON Schema is updated.

4. **Registry update.** The registry's enum validator is updated to accept the new value. A grace period of 7 days is allowed for the rollout.

5. **Adapter and model updates.** Model manifests may begin declaring support for the new value after the registry update. Adapters may release MINOR updates to handle the new value.

### 5.2 Naming Conventions for New Values

- All lowercase.
- Use underscores to separate words (e.g., `multi_label_classify`, not `multiLabelClassify`).
- Use the most generic name that accurately describes the capability.
- Avoid provider-specific or technology-specific terminology.
- New `domain` values should represent subject-matter domains, not industries or verticals (prefer `legal` over `law_firm`).

### 5.3 Deprecating Values

Values are never removed from the specification in a MINOR release. Deprecation is announced in the specification and in a registry warning. Removal requires a MAJOR release with a minimum 6-month notice period.
