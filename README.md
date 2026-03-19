# MIDAS API Specifications

Machine-readable specifications for the California Energy Commission's [Market Informed Demand Automation Server (MIDAS)](https://midasapi.energy.ca.gov/), which provides access to utilities' time-varying rates, GHG emission signals, and California ISO Flex Alerts.

## Disclaimer

**These are not official CEC artifacts.** The OpenAPI specifications and JSON Schemas in this repository were derived from the publicly available [MIDAS documentation](https://github.com/california-energy-commission/MIDAS) and are provided here on a best-effort basis. They may be incomplete, inaccurate, or out of date relative to the actual API behavior.

In an ideal world, API providers would publish machine-readable specifications alongside their documentation — enabling client code generation, automated testing, and robust integrations. Since the CEC has not yet seen fit to do so, we have done it for them. You're welcome, CEC.

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
  holiday/           # Holiday API (authenticated)
    openapi.yaml
    examples/
  historical/        # HistoricalData & HistoricalList APIs (authenticated)
    openapi.yaml
```

Each API directory follows the same convention:

| File/Dir | Purpose |
|----------|---------|
| `openapi.yaml` | OpenAPI 3.1 spec for the API |
| `schemas/` | Standalone JSON Schema files for reuse by client libraries and tooling |
| `examples/` | Sample request/response JSON for testing |

## API Overview

| API | Auth | Status | Description |
|-----|------|--------|-------------|
| **ValueData** | Bearer token | Specified | Query rate, GHG, and Flex Alert data by RIN; list RINs; retrieve lookup tables |
| **Token** | Basic Auth | Specified | Retrieve short-lived (10-minute) bearer tokens |
| **Registration** | None | Specified | Create new user and LSE accounts (POST only) |
| **Holiday** | Bearer token | Specified | Retrieve utility holiday schedules |
| **HistoricalData** | Bearer token | Specified | Retrieve archived rate information by RIN and date range |
| **HistoricalList** | Bearer token | Specified | List RINs with available historical data by provider |

See the [MIDAS documentation](https://github.com/california-energy-commission/MIDAS) for the upstream API docs (such as they are).

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

## License

[MIT License](LICENSE) — Copyright (c) 2026 Clark Communications Corporation
