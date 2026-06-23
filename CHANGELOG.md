# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project follows [Semantic Versioning](https://semver.org/spec/v2.0.0.html). The spec version tracks the MIDAS API major version it documents: `1.0.0` covers MIDAS v1.0; `2.0.0` covers MIDAS v2.0 (CEC release 2026-06-22). These are unofficial, best-effort specifications — see the [README disclaimer](README.md#disclaimer).

## [2.0.0] — 2026-06-22

Released at the CEC v2.0 cutover (2026-06-22) and merged to `main`. The breaking changes below were verified against the production v2.0 API during `clj-midas` and `python-midas` 1.0.0 smoke-testing (see the **Fixed** entries for the live-test corrections). See [doc/v2-migration.md](doc/v2-migration.md) for the full v1.0 → v2.0 delta and [doc/cec-v2-change-guide.md](doc/cec-v2-change-guide.md) for the CEC's official consumer change guide.

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
- `info.version` bumped to `2.0.0` across all four specs (ValueData, Historical, Token, Registration).

### Added

- `apis/value-data/schemas/midas-rin-list-response.schema.json` modeling the v2.0 keyed RIN-list object.
- `doc/v2-migration.md` (spec-level v1.0 → v2.0 delta) and `doc/cec-v2-change-guide.md` (CEC's official consumer change guide, reproduced verbatim).
- Enum extensions in `midas-enums.schema.json`: `signalType` gains the v2.0 populated labels, `rateType` gains `MOER`, `unit` gains `g/kWh CO2`.

### Deprecated

- The v1.0 SGIP GHG (`SGRT`/`SGFC`/`SGHT`) and Flex Alert (`FXRT`/`FXFC`/`FXHT`) RINs, replaced by the consolidated `MOER` and `ALRT` RINs above. Confirmed retired at cutover. Note: the CEC "MIDAS v2.0 is Now Live" email (2026-06-22) said calls to the old RINs return `HTTP 410 Gone`, but the **live API returns `HTTP 404`** with `{"detail": "RIN not found: <RIN>"}` — a retired RIN is indistinguishable from a never-existed one (live smoke-test 2026-06-22, GitHub issue #5).

### Removed

- **Breaking — the standalone `GET /Holiday` endpoint** is removed from the public read surface. It is absent from the CEC's published OpenAPI, and the `apis/holiday/` spec and example are removed from this set. Note the route still exists at the routing layer (`/api/Holiday`) but is now **auth-gated**: an anonymous request returns `HTTP 401 {"detail": "Not authenticated"}`, not `404`/gone (live smoke-test 2026-06-22, GitHub issue #7). The `Holiday` day-type value in rate schedules (`8=Holiday`) is a separate concept and is unaffected.
- **Breaking — the `HistoricalList` operation.** Use `GET /valuedata?SignalType=0` for the full active RIN list.
- The `?LookupTable=Holiday` and `?LookupTable=TimeZone` lookup tables. These return `HTTP 400 {"detail": "Unsupported lookup table: <name>"}` in v2.0 (not `404` — live smoke-test 2026-06-22, GitHub issue #6).

### Fixed

Corrections from live v2.0 smoke-testing (`clj-midas`, 2026-06-22; GitHub issues #1–#4):

- **RIN-list wrapper key is always `Rates`** (issue #2), regardless of `SignalType` — the `GHGEmissions`/`FlexAlerts`/`All` keys do not appear on the wire. `RinListResponse` (OpenAPI + `midas-rin-list-response.schema.json`) now models a single required `Rates` key.
- **LookupTable response is a keyed object** `{ table_name, data: [LookupEntry] }`, not a bare array (issue #3). Added `LookupTableResponse` (OpenAPI) and `midas-lookup-table-response.schema.json`; `LookupEntry` now permits extra row columns (`PayloadDescriptor`, `UnitType` on the `Unit` table).
- **`RateType` wire value is inconsistent across signal types** (issue #1): electricity rates return the short `Ratetype` UploadCode (`TOU`, `CPP`, …) while GHG/Flex return the long Description (`Greenhouse Gas emissions`, `Flex Alert`). Earlier notes had this backwards. Corrected `RateInfo.RateType` docs and the `tou-rate`/`flex-alert` examples.
- **`LastUpdated` is UTC with a basic-format offset** `±HHMM` (e.g. `+0000`), resolving the documented v1.0 bare/zoneless-PT TBD (issue #4). Updated `doc/datetime-and-timezone.md`, `midas-rin-list-entry.schema.json`, and `rin-list-sample.json`; noted the RFC-3339 parsing caveat.

Error-semantics corrections from `python-midas` 1.0.0 live smoke-testing (2026-06-22; GitHub issues #5–#7) — the CEC announcement's status codes did not match the live API:

- **Retired RINs return `404`, not `410 Gone`** (issue #5): `{"detail": "RIN not found: <RIN>"}`.
- **Retired lookup tables return `400`, not `404`** (issue #6): `{"detail": "Unsupported lookup table: <name>"}`.
- **Standalone `/Holiday` returns `401`, not `404`/removed** (issue #7): the route persists at `/api/Holiday` but is auth-gated (`{"detail": "Not authenticated"}` for anonymous callers) — still gone from the public read surface.
- Added an `Error` schema and `400`/`404` responses to `apis/value-data/openapi.yaml`.

## [1.0.0] — 2026-03-19

Initial machine-readable spec set for the MIDAS v1.0 API — OpenAPI 3.1 and JSON Schema (draft 2020-12) covering endpoints the CEC documents only in prose.

### Added

- OpenAPI specs: ValueData (RIN list / rate values / lookup tables / upload schema), Token (Basic-auth bearer issuance), Registration (public POST), Holiday, and HistoricalData & HistoricalList.
- Standalone JSON Schema files for rate data, RIN-list entries, lookup entries, and centralized enums.
- Sample response examples for the ValueData endpoint.
- Documentation: `doc/rin-structure.md` (RIN format), `doc/flex-alerts.md` (Flex Alert signals and history), and `doc/datetime-and-timezone.md` (empirical UTC-vs-PT wire conventions).

[2.0.0]: https://github.com/grid-coordination/midas-api-specs/releases/tag/v2.0.0
[1.0.0]: https://github.com/grid-coordination/midas-api-specs/releases/tag/v1.0.0
