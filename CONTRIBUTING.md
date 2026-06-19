# Contributing to midas-api-specs

Thanks for your interest in contributing! This repo provides unofficial, machine-readable specifications — OpenAPI 3.1 and JSON Schema (draft 2020-12) — for the California Energy Commission's [MIDAS](https://midasapi.energy.ca.gov/) API. The CEC publishes prose documentation in the [MIDAS repository](https://github.com/california-energy-commission/MIDAS) but no machine-readable specs; this repo fills that gap on a best-effort basis. The specs are derived from the upstream docs and validated against the live API — so the most valuable contributions are corrections grounded in observed API behavior.

## How to contribute

### Discussions

Use [Discussions](https://github.com/grid-coordination/midas-api-specs/discussions) for:

- Questions about how to read or use the specs — RIN structure, the overloaded ValueData endpoint, datetime/timezone conventions, signal types
- Judgment calls about how to model MIDAS behavior — "should the spec represent X this way?" when the upstream docs are silent or contradictory
- Interpretation of MIDAS quirks where the live API disagrees with the CEC docs and you want to scope what the spec should assert
- Coordination with downstream consumers (e.g. [clj-midas](https://github.com/grid-coordination/clj-midas)) and the upstream [CEC MIDAS repo](https://github.com/california-energy-commission/MIDAS)
- Sharing what you're building on top of these specs

Discussions are open-ended — a good place to think out loud or scope something before it becomes a concrete change. Aligned outcomes often turn into one or more Issues.

### Issues

Use [Issues](https://github.com/grid-coordination/midas-api-specs/issues) for actionable changes:

- A spec that disagrees with observed live-API behavior (a field shape, casing, nullability, enum value, or status code the spec gets wrong)
- Missing coverage — an endpoint, query parameter, response shape, or lookup table the specs don't yet model
- Schema bugs — a `$ref` that doesn't resolve, an over- or under-constrained `pattern`, a missing `required` field
- Example responses that no longer match the live API or fail schema validation
- Documentation errors or stale prose in `README.md`, `CHANGELOG.md`, or the `doc/` notes
- Discussion outcomes that have alignment and a clear scope

When filing a behavioral discrepancy, please include the request (endpoint + query parameters, with the RIN if applicable) and the relevant slice of the actual JSON response — the spec asserts what the API *does*, so a concrete observation is what lets us fix it. If you're not sure whether something is an Issue or a Discussion, start with a Discussion — we can convert it later.

### Pull requests

Pull requests are welcome.

- For small fixes (typos, broken links, a single enum value, an example correction), open a PR directly.
- For substantive changes (new endpoint coverage, a new schema file, a response-shape change, a MIDAS version bump), open a Discussion or Issue first so we can align on scope.
- All OpenAPI specs lint cleanly under `npx @redocly/cli lint apis/*/openapi.yaml`, and example JSON validates against the JSON Schemas (see [Development](#development)).
- Match the existing conventions: OpenAPI 3.1 in YAML, JSON Schema draft 2020-12 with `$id` for cross-file `$ref`, `additionalProperties: false` for strict validation, and enums centralized in `apis/value-data/schemas/midas-enums.schema.json`. One directory per API (`openapi.yaml`, `schemas/`, `examples/`).
- Record reader-visible changes in `CHANGELOG.md` under the appropriate version, and note the MIDAS API version a change tracks.
- One commit per logical change is fine; we don't require squash or any particular branch naming.

## Development

```bash
# Lint all OpenAPI specs
npx @redocly/cli lint apis/*/openapi.yaml

# Validate sample JSON against the JSON Schemas (Python) — see README "Validation"
# for the full referencing-registry snippet
pip install jsonschema referencing
```

Lint a single spec while iterating with `npx @redocly/cli lint apis/value-data/openapi.yaml`. Markdown is linted with `markdownlint-cli2`.

## Spec source of truth and branch model

The CEC's [MIDAS documentation](https://github.com/california-energy-commission/MIDAS) and the live API at <https://midasapi.energy.ca.gov/> are the source of truth for routes, parameters, and response shapes. Where the docs and the live API disagree, the spec follows the live API and notes the discrepancy. The empirically-verified datetime/timezone conventions, RIN format, and Flex Alert behavior are documented under `doc/`.

The spec version tracks the MIDAS API major version it documents. The `main` branch carries the **MIDAS v1.0** spec; the `v2` branch stages the **MIDAS v2.0** spec (CEC release 2026-06-22) so downstream consumers can build against it ahead of release. See [`doc/v2-migration.md`](doc/v2-migration.md) for the v1.0 → v2.0 delta. Target PRs at the branch matching the API version you're correcting.

## Code of conduct

Be respectful and constructive. We're a small project and appreciate everyone who takes the time to file an issue or send a PR.

## Important notice

These specifications are **not official CEC artifacts** and are provided on an "as-is" basis. Updates and maintenance, including responses to issues filed on GitHub, happen on an "as time and resources permit" basis. The specs are best-effort against the MIDAS API as documented by the [CEC](https://github.com/california-energy-commission/MIDAS) and observed on the live service; they may be incomplete, inaccurate, or out of date relative to actual API behavior. Independent verification against the live API is recommended for any consumer relying on these specs in production.
