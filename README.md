# MIDAS API Specifications

Machine-readable specifications for the California Energy Commission's [Market Informed Demand Automation Server (MIDAS)](https://midasapi.energy.ca.gov/), which provides access to utilities' time-varying rates, GHG emission signals, and California ISO Flex Alerts.

> **This is the MIDAS v2.0 spec (CEC release 2026-06-22) — live and verified.** `main` now carries the v2.0 specification, smoke-tested against the production API by the `clj-midas` and `python-midas` 1.0.0 clients. GET endpoints are unauthenticated, GHG units changed from `kg/kWh CO2` to `g/kWh CO2` (values 1000× larger), the SGIP GHG and Flex Alert RINs are consolidated, the RIN-list and lookup-table responses are keyed objects, the value field is `Value`, and all wire datetimes are UTC. See [doc/v2-migration.md](doc/v2-migration.md) for the full v1.0 → v2.0 delta and [doc/cec-v2-change-guide.md](doc/cec-v2-change-guide.md) for the CEC's official consumer-facing change guide.

## Disclaimer

**These are not official CEC artifacts.** The OpenAPI specifications and JSON Schemas in this repository were derived from the publicly available [MIDAS documentation](https://github.com/california-energy-commission/MIDAS) and are provided here on a best-effort basis. They may be incomplete, inaccurate, or out of date relative to the actual API behavior.

As of the v2.0 release, the CEC publishes its own auto-generated OpenAPI document at [`https://midasapi.energy.ca.gov/openapi.json`](https://midasapi.energy.ca.gov/openapi.json) (Swagger UI at [`/docs`](https://midasapi.energy.ca.gov/docs)). That document is authoritative for the endpoint inventory, request parameters, and the write/admin side of the API — but it does not model the GET **response** bodies that data consumers actually parse (the keyed RIN-list object, `ValueInformation` intervals, the `Value` field, and the GHG and Flex Alert structures). Those consumer-facing response shapes — with strict validation (`additionalProperties: false`) and worked examples — are what this repository adds.

For additional perspective on MIDAS's architectural choices — including the fact that the MIDAS service protocol and RIN are not standards, that the protocol lacks publish/subscribe support, and that no other jurisdiction uses or is likely to use it — see our [response to CEC Docket #24-FDAS-03](https://grid-coordination.github.io/policy/24-fdas-03).

## Repository Layout

```
apis/
  value-data/        # ValueData API (authenticated) — the main one
    openapi.yaml     # OpenAPI 3.1 spec
    schemas/         # JSON Schema (draft 2020-12) files
    examples/        # Sample responses from the live API
  token/             # Token API (Basic Auth)
    openapi.yaml
  registration/      # Registration API (public, POST only)
    openapi.yaml
  historical/        # HistoricalData & HistoricalList APIs (authenticated)
    openapi.yaml
doc/
  rin-structure.md          # Rate Identification Number (RIN) format and structure
  flex-alerts.md            # Flex Alert signals, CAISO escalation ladder, and history
  datetime-and-timezone.md  # Wire-format datetime conventions (UTC vs PT — empirically verified)
  v2-migration.md           # MIDAS v2.0 (2026-06-22) spec-level delta and migration phases
  cec-v2-change-guide.md    # CEC's official consumer-facing change guide, reproduced verbatim
```

Each API directory follows the same convention:

| File/Dir | Purpose |
|----------|---------|
| `openapi.yaml` | OpenAPI 3.1 spec for the API |
| `schemas/` | Standalone JSON Schema files for reuse by client libraries and tooling |
| `examples/` | Sample request/response JSON for testing |

## API Overview

| API | Auth (v1.0) | Auth (v2.0) | Status | Description |
|-----|------|------|--------|-------------|
| **ValueData** | Bearer token | Open (GET) | Specified | Query rate, GHG, and Flex Alert data by [RIN](doc/rin-structure.md); list RINs; retrieve lookup tables. Response shape and RIN inventory change substantially in v2.0 — see [v2-migration.md](doc/v2-migration.md). |
| **Token** | Basic Auth | Basic Auth (still works; no longer required for GET data calls) | Specified | Retrieve short-lived (10-minute) bearer tokens |
| **Registration** | None | None | Specified | Create new user and LSE accounts (POST only) |
| **Holiday** | Bearer token | **Retired in v2.0** — removed from the CEC's published spec and from this set (the `Holiday` day-type in rate schedules is unaffected) | Removed | Retrieve utility holiday schedules |
| **HistoricalData** | Bearer token | Open | Specified | Retrieve archived rate information by RIN and date range. Path changes from `/HistoricalData?id=…` to `/historicaldata/{rate_id}` in v2.0. |
| **HistoricalList** | Bearer token | **Removed in v2.0** — use `/valuedata?SignalType=0` instead | Specified | List RINs with available historical data by provider |

See the [MIDAS documentation](https://github.com/california-energy-commission/MIDAS) for the upstream API docs, and the CEC's own [`/openapi.json`](https://midasapi.energy.ca.gov/openapi.json) for the request-side contract.

## Datetime conventions

The MIDAS API mixes UTC and bare wall-clock datetimes on the wire and does not document which fields are which. Fields whose names end in `_UTC` (or whose ISO 8601 strings carry a `Z` suffix) are UTC; **every other datetime field is `America/Los_Angeles` local** (PT — PST in winter, PDT in summer). See [doc/datetime-and-timezone.md](doc/datetime-and-timezone.md) for the empirical verification, the per-field inventory, and consumer guidance for parsing.

## Validation

```bash
# Lint all OpenAPI specs
npx @redocly/cli lint apis/*/openapi.yaml

# Validate sample JSON against the JSON Schema (Python)
pip install jsonschema referencing
python -c "
import json
from pathlib import Path
from referencing import Registry, Resource
from jsonschema import Draft202012Validator

schema_dir = Path('apis/value-data/schemas')
store = {}
for f in schema_dir.glob('*.schema.json'):
    schema = json.loads(f.read_text())
    store[schema['\$id']] = Resource.from_contents(schema)

registry = Registry().with_resources(store.items())
response_schema = json.loads((schema_dir / 'midas-value-data-response.schema.json').read_text())
sample = json.loads(Path('apis/value-data/examples/tou-rate-response-sample.json').read_text())

validator = Draft202012Validator(response_schema, registry=registry)
validator.validate(sample)
print('OK')
"
```

## Changelog

Release history and behavioral changes are tracked in [CHANGELOG.md](CHANGELOG.md). The spec version tracks the MIDAS API major version it documents (`1.0.0` = MIDAS v1.0; `2.0.0` = MIDAS v2.0).

## Contributing

Issues, Discussions, and pull requests are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the workflow (and the validation commands). In short:

- **Questions, modeling judgment calls, MIDAS-quirk interpretation** → [Discussions](https://github.com/grid-coordination/midas-api-specs/discussions)
- **Specs that disagree with the live API, missing coverage, schema bugs, doc errors** → [Issues](https://github.com/grid-coordination/midas-api-specs/issues) (please include the request and the actual JSON response)
- **Patches** → pull requests; open a Discussion or Issue first for non-trivial changes, and target the branch matching the MIDAS API version (`main` = v1.0, `v2` = v2.0)

## License

[MIT License](LICENSE) — Copyright (c) 2026 Clark Communications Corporation
