# Contributing to the SYNAPSE Canonical IR Specification

## How to propose a new task_type or domain value

New values for task_type and domain must be proposed via a GitHub pull 
request to this repository (github.com/synapse-ir/spec).

A proposal must include:
1. A clear definition of the new value
2. At least one reference implementation (a registered adapter that uses it)
3. A justification for why the existing set does not cover the use case

Review target: 14 days for uncontroversial additions.

## How to submit a specification amendment

Amendments to existing sections are submitted as pull requests with:
1. The specific section being amended (e.g. §1.3.1)
2. The proposed change with before/after comparison
3. The rationale for the change
4. Any backward compatibility implications

Breaking changes (additions that would make existing adapters 
non-conformant) require a MAJOR version bump and a migration guide.

## What makes a good proposal

- Concrete: describes a real use case, not a hypothetical
- Scoped: changes the minimum necessary to address the use case
- Compatible: does not break existing conformant implementations
- Referenced: links to at least one real-world example of the need

## Review process

All proposals are reviewed by the SYNAPSE specification team.
The review target is 14 days for uncontroversial additions.
Complex or breaking changes may require additional discussion.

Proposals that do not include a reference implementation will be 
held until one is available.

## License

By contributing to this repository, you agree that your contributions 
will be licensed under Creative Commons Attribution 4.0 International 
(CC BY 4.0), the same license as the specification itself.
