# MIDAS v2.0 — Spec Migration Reference

The California Energy Commission is releasing **MIDAS v2.0 on 2026-06-22**. This document is the spec-level delta between v1.0 and v2.0: which schema files, OpenAPI paths, and JSON fields change, and what verification is still outstanding. For end-user migration narrative, see the CEC's Change Guide for Data Consumers and the upstream announcement email.

This repo's `main` branch tracks the v1.0 spec; the **`v2` branch carries the breaking changes** (see bd issue `midas-api-specs-b6k`). The `v2` branch was cut early — on 2026-06-19, three days ahead of release — so downstream API repos (e.g. `clj-midas`) can stage their own v2.0 feature branches against it. The breaking changes on `v2` reflect the CEC change guide and the resolved clarifications (§7); a live smoke-test against the production v2.0 API on release day (2026-06-22) is the remaining gate before merge. Safe additive changes (enum extensions, doc notes flagging upcoming behavior) already landed on `main` ahead of time.

## Sources

- **Upstream announcement**: MIDAS team email, 2026-06-11, "MIDAS v2.0 — Updates for Data Consumers (June 22nd Release)"
- **Consumer change guide**: CEC's "MIDAS v2.0 — Change Guide for Data Consumers" distributed with the announcement — reproduced verbatim at [cec-v2-change-guide.md](cec-v2-change-guide.md)
- **Contact for questions**: <midas@energy.ca.gov>

## 1. Authentication

| Endpoint | v1.0 | v2.0 |
|---|---|---|
| `GET /valuedata` (any query shape) | Bearer token required | Unauthenticated |
| `GET /historicaldata/{rate_id}` | Bearer token required | Unauthenticated |
| `POST` uploads (LSE rate submission) | Bearer token | Bearer token (unchanged) |
| `POST /Token` | Basic auth (unchanged) | Basic auth (unchanged) |

**Spec impact:** in `apis/value-data/openapi.yaml` and `apis/historical/openapi.yaml`, drop `security: [bearerAuth]` from GET operations. Keep the `bearerAuth` scheme defined in `components.securitySchemes` for upload operations and for any lookup-table calls that still accept it.

## 2. Endpoint path/shape changes

| v1.0 | v2.0 | Notes |
|---|---|---|
| `GET /ValueData?ID={rin}&QueryType={alldata\|realtime}` | `GET /ValueData?ID={rin}&QueryType={alldata\|realtime}` | v2.0 paths are case-insensitive; CEC docs and this spec use PascalCase `/ValueData` for consistency. |
| `GET /HistoricalData?id={rin}&startdate=&enddate=` | `GET /HistoricalData/{rate_id}?startdate=&enddate=` | RIN moves from query param to path param. Max range: 6 months per call. |
| `GET /HistoricalList?DistributionCode=&EnergyCode=` | **Removed.** Use `GET /ValueData?SignalType=0` for the full active RIN list. | |
| `GET /ValueData?LookupTable=Holiday` | **Removed.** | |
| `GET /ValueData?LookupTable=TimeZone` | **Removed.** | |
| `GET /Holiday` (standalone endpoint, `apis/holiday/openapi.yaml`) | **Kept for now, retirement planned.** Per CEC clarification, the standalone endpoint is retained at release but is on a deprecation path; final decision to come in the next CEC documentation update. | |

## 3. Response shape changes

### RIN list (`?SignalType=N`)

v1.0 returned a bare array of `RinListEntry`. v2.0 returns a keyed object:

```json
{ "Rates":         [ { "RateID": "...", ... }, ... ] }   // SignalType=1
{ "GHGEmissions":  [ ... ] }                              // SignalType=2
{ "FlexAlerts":    [ ... ] }                              // SignalType=3
{ "All":           [ ... ] }                              // SignalType=0
```

**Spec impact:** new schema `apis/value-data/schemas/midas-rin-list-response.schema.json`; `oneOf` branch in `apis/value-data/openapi.yaml` references it.

The per-entry `SignalType` field value also changes substantially:

| v1.0 value | v2.0 value | Applies to |
|---|---|---|
| `"Rates"` | `"Electricity Rates"` | SignalType=1 entries |
| `null` (on the live v1.0 API, despite docs implying `"GHG"`) | `"Greenhouse Gas Emissions"` | SignalType=2 (now `MOER`) entries |
| `null` (on the live v1.0 API, despite docs implying `"Flex Alert"`) | `"California Independent System Operator Flex Alert"` | SignalType=3 (now `ALRT`) entries |

### Value field casing

The wire field is `Value` (uppercase) in v2.0 — confirmed by the CEC. v1.0 returned lowercase `value` despite upstream docs showing uppercase; v2.0 fixes the casing.

**Spec impact:** rename `value` → `Value` in `apis/value-data/schemas/midas-value-data.schema.json`.

## 4. Field semantics

| Field / Schema | v1.0 | v2.0 |
|---|---|---|
| `Unit` (GHG entries) | `"kg/kWh CO2"` | `"g/kWh CO2"` |
| Value (GHG entries) | kilograms/kWh CO2 | grams/kWh CO2 — values are **1000× larger** for the same physical reading |
| `realtime` window | Variable per RIN; SGRT was a single point, FXRT was a live pass-through | Fixed **72-hour** window starting midnight Pacific on request date; expressed as UTC on wire (e.g. first datapoint `TimeStart: "07:00:00"` during PDT) |
| `alldata` window | Undocumented cap, variable response sizes | Fixed **90-day** window ending 23:59:59 PT on Day+2; expressed as UTC on wire |
| `historicaldata` max range per call | (varied) | **6 months** per call (confirmed by CEC) |
| Bare `DateStart`/`DateEnd`/`TimeStart`/`TimeEnd` (all signal types) | UTC for electricity rates; **PT for SGIP GHG and Flex Alert** (upstream-provider bug) | **UTC for all signal types** — MIDAS v2.0 converts WattTime (Pacific) and CAISO (Pacific) timestamps to UTC before delivery |

## 5. Enum changes

`apis/value-data/schemas/midas-enums.schema.json`:

| Enum | Additions (v2.0) | Removals (v2.0, breaking) |
|---|---|---|
| `signalType` | `"Electricity Rates"` | `"Rates"` (replaced by Electricity Rates) |
| `rateType` | `"MOER"` (4-char RIN segment-3 code for the new unified SGIP GHG signal) | `"GHG"` may remain for v1.0 compatibility but no live v2.0 RIN uses it |
| `unit` | `"g/kWh CO2"` | `"kg/kWh CO2"` (replaced; retain in spec only for historical-archive consumers) |
| `lookupTable` | — | `"Holiday"`, `"TimeZone"` |

Additive changes landed on `main` ahead of the release. Removals stage on the v2 branch.

## 6. RIN migration

### SGIP GHG: 33 RINs → 11 RINs

Old (v1.0): 11 regions × 3 signal types (`SGRT`/`SGFC`/`SGHT`) = 33 RINs.
New (v2.0): 11 regions × 1 signal type (`MOER`) = 11 RINs.

| Region | v1.0 RINs | v2.0 RIN |
|---|---|---|
| PacifiCorp West | `USCA-SGIP-{SGRT,SGFC,SGHT}-PACW` | `USCA-SGIP-MOER-PACW` |
| CAISO — SDG&E | `USCA-SGIP-{SGRT,SGFC,SGHT}-SDGE` | `USCA-SGIP-MOER-SDGE` |
| CAISO — PG&E | `USCA-SGIP-{SGRT,SGFC,SGHT}-PGE` | `USCA-SGIP-MOER-PGE` |
| BANC — P2 | `USCA-SGIP-{SGRT,SGFC,SGHT}-BANC` | `USCA-SGIP-MOER-P2` |
| Turlock Irrigation District | `USCA-SGIP-{SGRT,SGFC,SGHT}-TID` | `USCA-SGIP-MOER-TID` |
| BANC — SMUD | `USCA-SGIP-{SGRT,SGFC,SGHT}-SMUD` | `USCA-SGIP-MOER-SMUD` |
| Imperial Irrigation District | `USCA-SGIP-{SGRT,SGFC,SGHT}-IID` | `USCA-SGIP-MOER-IID` |
| NV Energy | `USCA-SGIP-{SGRT,SGFC,SGHT}-NVENERGY` | `USCA-SGIP-MOER-NVENERGY` |
| Western Area Lower Colorado | `USCA-SGIP-{SGRT,SGFC,SGHT}-WALC` | `USCA-SGIP-MOER-WALC` |
| CAISO — SCE | `USCA-SGIP-{SGRT,SGFC,SGHT}-SCE` | `USCA-SGIP-MOER-SCE` |
| LADWP | `USCA-SGIP-{SGRT,SGFC,SGHT}-LADWP` | `USCA-SGIP-MOER-LADWP` |

Note the BANC region code changes from `BANC` (v1.0) to `P2` (v2.0). All other region codes are preserved.

`MOER` realtime responses are a continuous 5-minute time series blending past observed data, current readings, and WattTime forecast.

### Flex Alert: 3 RINs → 1 RIN

| v1.0 RINs | v2.0 RIN |
|---|---|
| `USCA-FLEX-FXRT-0000`, `USCA-FLEX-FXFC-0000`, `USCA-FLEX-FXHT-0000` | `USCA-FLEX-ALRT-0000` |

`FXHT` was non-functional in v1.0; `ALRT` is its first working replacement. `ALRT` returns a clean hourly binary series — every hour in the requested window present with `Value=0` or `Value=1`.

## 7. Resolved questions (CEC reply, 2026-06-12)

The six pre-release open questions were clarified by the CEC MIDAS team on 2026-06-12. Summarized answers:

1. **`historicaldata` max range**: **6 months.** The CEC changed the initially-decided 1-year limit to 6 months and will communicate the change in the next user announcement.

2. **Field casing**: **`Value` (capital V).** Standardized in v2.0.

3. **Path casing**: **Case-insensitive in v2.0.** Both `/valuedata` and `/ValueData` work. CEC documentation will continue to use PascalCase `/ValueData` for consistency; this spec follows the same convention.

4. **`/Holiday` standalone endpoint**: **Kept for now, retirement planned.** Final decision will be shared in the next CEC documentation update. The spec retains `apis/holiday/` with a note flagging planned retirement.

5. **RIN list per-entry `SignalType` field**: **Populated, no longer null.**
   - GHG / `MOER` entries return `"Greenhouse Gas Emissions"`.
   - Flex Alert / `ALRT` entries return `"California Independent System Operator Flex Alert"`.

6. **Wire timezone for `DateStart`/`TimeStart` in `ValueInformation`**: **UTC in v2.0 for all signal types.** The v1.0 PT-on-wire behavior we observed for SGIP GHG and Flex Alert was a documented bug — those datapoints were thin pass-throughs of the WattTime and CAISO upstream APIs (both natively PT) and MIDAS v1.0 did not convert before delivery. v1.0 electricity-rate RINs were always UTC. v2.0 converts upstream-provider timestamps to UTC before delivery. Window boundaries remain PT-aligned but are expressed in UTC: midnight Pacific = `07:00:00` UTC during PDT, `08:00:00` UTC during PST. See `doc/datetime-and-timezone.md` for the per-field inventory and consumer guidance.

## 8. Migration phases

1. **Pre-release (now → 2026-06-22)** — `main` branch: safe additive changes. Doc updates and enum extensions that don't break v1.0 consumers. See bd issues `midas-api-specs-{34p,3g9,a30,4gh,1v6,2in}` (all closed).
2. **Staging (cut 2026-06-19, ahead of plan)** — `v2` branch (`midas-api-specs-b6k`) cut early so downstream repos can build against it. Breaking changes applied: GET endpoints unauthenticated (`1uu`); RIN-list keyed-object response + `midas-rin-list-response.schema.json` (`dmt`, `3os`); `value` → `Value` casing (`82u`); `realtime`/`alldata` window semantics (`cnv`); `/historicaldata/{rate_id}` path form + HistoricalList removed (`2x5`); Holiday/TimeZone already absent from the lookup enums (`0zo`, no-op); `/Holiday` flagged retirement-planned, pending live check (`ym3`). With §7 resolved by CEC, the only on-the-day verification needed is smoke-testing a live v2.0 response per signal type and confirming the documented behavior.
3. **Release day (2026-06-22)** — Live smoke-test `v2` against production per signal type; confirm `Value` casing, keyed RIN-list shape, UTC window boundaries, and the `/Holiday` endpoint's fate. Regenerate examples against live data (`midas-api-specs-cay`).
4. **Post-release** — Tag spec `v1.0.0` (`midas-api-specs-2d7`) as a frozen v1 baseline. Merge `v2` to `main` and bump the release version.
