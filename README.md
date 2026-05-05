# SYNAPSE Canonical IR Specification

The SYNAPSE Canonical Intermediate Representation (IR) is an open specification
for heterogeneous AI model interoperability. It defines a typed, versioned,
self-describing JSON message format that enables specialized AI models with
incompatible input/output schemas to compose into coherent pipelines.

## The problem this specification solves

Connecting N specialized AI models requires N×(N-1)/2 custom connectors.
Four models require 6 connectors. Ten models require 45.
Each breaks whenever either model's schema changes.

The canonical IR reduces this to 2N adapter functions: one ingress and one
egress per model. A model joining the ecosystem writes two functions once
and achieves permanent interoperability with every other registered model.

## Relationship to MCP and A2A

SYNAPSE does not compete with MCP or A2A. It builds on top of them.

- MCP (Anthropic / Linux Foundation): tool and data source connectivity
- A2A (Google / Linux Foundation): agent-to-agent messaging
- SYNAPSE Canonical IR: schema translation and capability routing between
  heterogeneous specialist models — the layer neither MCP nor A2A provides

## Specification documents

| Document | Description |
|----------|-------------|
| [§0 Conventions](spec/s0-conventions.md) | Notation and terminology |
| [§1 Canonical IR Schema](spec/s1-canonical-ir.md) | Full IR schema definition |
| [§2 Adapter SDK](spec/s2-adapter-sdk.md) | Adapter contract and validation rules |
| [§3 Capability Manifest](spec/s3-manifest.md) | Model registration schema |
| [§4 Registry API](spec/s4-registry-api.md) | Registry endpoints and routing |
| [§5 Versioning](spec/s5-versioning.md) | Compatibility guarantees |
| [§6 Error Codes](spec/s6-errors.md) | Standardized error responses |
| [§7 Quickstart](spec/s7-quickstart.md) | Getting started guide |
| [§8 Caching Architecture](spec/s8-caching.md) | Five-layer cache specification |
| [§9 Gap Resolution Register](spec/s9-gaps.md) | Pre-build audit and resolutions |

## Reference implementation

- [adapter-sdk](https://github.com/synapse-ir/adapter-sdk) — Python + TypeScript SDK
- [registry](https://github.com/synapse-ir/registry) — FastAPI registry server
- [adapters](https://github.com/synapse-ir/adapters) — Community adapter collection

## Versioning

The specification follows semantic versioning. The current version is 1.0.0.
Breaking changes (MAJOR), new optional fields (MINOR), and clarifications (PATCH)
are handled per the policy defined in §5.

## Contributing

Proposals to extend the specification (new task_type or domain values,
new compliance tags, IR schema additions) are submitted as GitHub pull requests.
See CONTRIBUTING.md for the proposal process.
Review target: 14 days for uncontroversial additions.

## Governance

The SYNAPSE Canonical IR Specification is currently maintained by the SYNAPSE
project team. Linux Foundation governance is a target milestone.

## License

The SYNAPSE Canonical IR Specification is licensed under
[Creative Commons Attribution 4.0 International (CC BY 4.0)](LICENSE).

Implementations of this specification may be licensed under any terms.
Attribution required: "Implements the SYNAPSE Canonical IR Specification"
with a link to this repository.
