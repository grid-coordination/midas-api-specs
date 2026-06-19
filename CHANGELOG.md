# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html). The spec version tracks the MIDAS API major version it documents: `1.0.0` covers MIDAS v1.0; `2.0.0` covers MIDAS v2.0 (CEC release 2026-06-22). These are unofficial, best-effort specifications — see the [README disclaimer](README.md#disclaimer).

## [2.0.0] — Unreleased (CEC release target 2026-06-22)

Staged on the `v2` branch ahead of release so downstream consumers can build against it; pending a live smoke-test against the production v2.0 API before merge to `main`. See [doc/v2-migration.md](doc/v2-migration.md) for the full v1.0 → v2.0 delta and [doc/cec-v2-change-guide.md](doc/cec-v2-change-guide.md) for the CEC's official consumer change guide.

### Changed

- **Breaking — GET endpoints are unauthenticated.** `GET /ValueData` and `GET /historicaldata/{rate_id}` now declare `security: [{}, {bearerAuth: []}]` (no token required; a token is still accepted, e.g. for lookup tables).
- **Breaking — RIN-list response is a keyed object.** `GET /ValueData?SignalType=N` returns `{Rates|GHGEmissions|FlexAlerts|All: [...]}` instead of the v1.0 bare array.
- **Breaking — value field renamed `value` → `Value`** (capital V) across schemas, OpenAPI, and examples, correcting the v1.0 lowercase quirk.
- **Breaking — SGIP GHG unit changed `kg/kWh CO2` → `g/kWh CO2`**; numeric values are 1000× larger for the same physical reading.
- **Breaking — HistoricalData moves to a path parameter.** `GET /HistoricalData?id={rin}` → `GET /historicaldata/{rate_id}`; maximum range 6 months per call.
- **Breaking — RIN-list `SignalType` is now populated for every entry** (`Electricity Rates`, `Greenhouse Gas Emissions`, `California Independent System Operator Flex Alert`); v1.0 returned `null` for GHG and Flex Alert entries.
- **Breaking — SGIP GHG and Flex Alert RIN inventory consolidated.** SGIP GHG: 33 RINs (11 regions × SGRT/SGFC/SGHT) → 11 `USCA-SGIP-MOER-{REGION}`. Flex Alert: 3 RINs (FXRT/FXFC/FXHT) → 1 `USCA-FLEX-ALRT-0000`.
- `realtime` is now a fixed 72-hour window and `alldata` a fixed 90-day window; all wire datetimes are normalized to UTC across every signal type (the v1.0 PT-on-wire passthrough for SGIP GHG and Flex Alert is gone).
- RIN pattern widened from `{4,10}` to `{2,10}` on the trailing segment to admit v2.0 short region codes (`P2`, `PGE`, `SCE`, `TID`, `IID`).
- `info.version` bumped to `2.0.0` across all five specs (ValueData, Historical, Holiday, Token, Registration).

### Added

- `apis/value-data/schemas/midas-rin-list-response.schema.json` modeling the v2.0 keyed RIN-list object.
- `doc/v2-migration.md` (spec-level v1.0 → v2.0 delta) and `doc/cec-v2-change-guide.md` (CEC's official consumer change guide, reproduced verbatim).
- Enum extensions in `midas-enums.schema.json`: `signalType` gains the v2.0 populated labels, `rateType` gains `MOER`, `unit` gains `g/kWh CO2`.

### Deprecated

- The v1.0 SGIP GHG (`SGRT`/`SGFC`/`SGHT`) and Flex Alert (`FXRT`/`FXFC`/`FXHT`) RINs, replaced by the consolidated `MOER` and `ALRT` RINs above.
- The standalone `GET /Holiday` endpoint — retained at release but flagged retirement-planned per CEC clarification (final decision pending; live confirmation tracked for release day).

### Removed

- **Breaking — the `HistoricalList` operation.** Use `GET /valuedata?SignalType=0` for the full active RIN list.
- The `?LookupTable=Holiday` and `?LookupTable=TimeZone` lookup tables (return `404` in v2.0).

## [1.0.0] — 2026-03-19

Initial machine-readable spec set for the MIDAS v1.0 API — OpenAPI 3.1 and JSON Schema (draft 2020-12) covering endpoints the CEC documents only in prose.

### Added

- OpenAPI specs: ValueData (RIN list / rate values / lookup tables / upload schema), Token (Basic-auth bearer issuance), Registration (public POST), Holiday, and HistoricalData & HistoricalList.
- Standalone JSON Schema files for rate data, RIN-list entries, lookup entries, and centralized enums.
- Sample response examples for the ValueData endpoint.
- Documentation: `doc/rin-structure.md` (RIN format), `doc/flex-alerts.md` (Flex Alert signals and history), and `doc/datetime-and-timezone.md` (empirical UTC-vs-PT wire conventions).

[2.0.0]: https://github.com/grid-coordination/midas-api-specs/tree/v2
[1.0.0]: https://github.com/grid-coordination/midas-api-specs/releases/tag/v1.0.0
