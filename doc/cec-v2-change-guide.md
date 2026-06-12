# CEC MIDAS v2.0 — Change Guide for Data Consumers

> **Provenance.** This document is the California Energy Commission's official
> "MIDAS v2.0 — Change Guide for Data Consumers," authored by the CEC MIDAS
> team and distributed to MIDAS users alongside the release announcement email
> on 2026-06-11. It is reproduced here verbatim for the convenience of
> readers of this spec repository who may not have received the upstream
> distribution. All rights and authorship remain with the CEC; the content
> below is **not** covered by this repository's MIT license. For the latest
> authoritative version, contact <midas@energy.ca.gov> or watch the
> [CEC MIDAS repository](https://github.com/california-energy-commission/MIDAS).
>
> For the spec-level delta (which schema files and fields change, with
> resolved-question notes from CEC clarifications), see
> [v2-migration.md](v2-migration.md).

---

<!-- markdownlint-disable MD031 MD034 MD036 -->
<!-- The body below is reproduced verbatim from CEC; lint rules disabled to
     avoid modifying the upstream text. Re-enabled at end of file. -->

# MIDAS v2.0 — Change Guide for Data Consumers

**Audience:** Researchers, automation providers, technology manufacturers, analytics developers, and anyone who queries MIDAS data
**Release:** June 22, 2026
**Contact:** <midas@energy.ca.gov>

---

## Summary of What Changed for You

| Change | Impact |
|--------|--------|
| No registration required for GET requests | Low — simplification, no action required unless you want to remove auth code |
| GHG unit changed: kg → g (1,000× scale) | **High** — all GHG-consuming applications must update thresholds and logic |
| SGIP GHG RINs consolidated (33 → 11) | **High** |
| Flex Alert RINs consolidated (3 → 1) | **High** |
| RINList response format changed | Medium — update JSON parsing |
| Query window changed (`alldata` = 90 days) | Medium — audit downstream time-range assumptions |
| `HistoricalRINList` endpoint removed | Low |
| `Holiday` and `TimeZone` lookup tables removed | Low — most consumers did not use these |

---

## 1. Authentication Changes

### GET requests no longer require a token

In v2.0, all public GET calls are unauthenticated:

```
GET /api/valuedata?SignalType=1
GET /api/valuedata?ID={RIN}&QueryType=realtime
GET /api/valuedata?ID={RIN}&QueryType=alldata
GET /api/historicaldata/{rate_id}
```

You may simplify your code by removing token acquisition and management for these calls. If you still want to use a token (for example, to access lookup tables), it will still be accepted.

**Before:**
```python
token = get_token(username, password)
resp = requests.get(url, params=params, headers={"Authorization": f"Bearer {token}"})
```

**After (simplified):**
```python
resp = requests.get(url, params=params)  # no auth needed
```

---

## 2. GHG Emissions — Unit Change (Critical)

### What changed

The unit of measure for all SGIP GHG emissions signals changed from **kilograms per kilowatt-hour** (`kg/kWh CO2`) to **grams per kilowatt-hour** (`g/kWh CO2`).

The `Unit` field in all SGIP GHG responses now reads `g/kWh CO2`. The numeric `Value` is **1,000 times larger** than the same physical reading was in v1.0.

| | v1.0 | v2.0 |
|--|------|------|
| `Unit` field | `kg/kWh CO2` | `g/kWh CO2` |
| Example value | `0.333` | `333.0` |
| Scale | 1× | 1,000× |

### What to update

Go through every part of your application that reads GHG emissions values and apply the following changes:

**1. Thresholds and comparisons**

```python
# v1.0
CLEAN_THRESHOLD = 0.200  # kg/kWh CO2

# v2.0
CLEAN_THRESHOLD = 200.0  # g/kWh CO2 — same physical value

# If you have old thresholds stored in a database or config file:
old_value_kg = 0.200
new_value_g = old_value_kg * 1000  # = 200.0
```

**2. Chart axes and labels**

Update Y-axis labels from `kg/kWh CO2` to `g/kWh CO2`. Rescale existing chart data by 1,000.

**3. Stored historical data**

If you have stored v1.0 GHG values in your own database, you will need to multiply all stored values by 1,000 before comparing against new data from v2.0.

**4. Alert configurations**

Update any alert triggers that compare against fixed GHG thresholds.

**5. Unit labels shown to end users**

Update any UI that displays the unit string.

---

## 3. SGIP GHG RIN Migration

### What changed

33 RINs (11 regions × 3 signal types: SGRT/SGFC/SGHT) are replaced by 11 MOER RINs.

The new RINs use the naming pattern `USCA-SGIP-MOER-{REGION}` and support two query types: `realtime` (72-hour window) and `alldata` (90-day window). The `realtime` response seamlessly combines past observed data, current readings, and WattTime forecast data into one continuous 5-minute time series — no more separate calls for current vs. forecast.

### Complete mapping

| Region | Old v1.0 RINs | v2.0 RIN |
|--------|--------------|----------|
| PacifiCorp West | `USCA-SGIP-SGRT-PACW`, `USCA-SGIP-SGFC-PACW`, `USCA-SGIP-SGHT-PACW` | `USCA-SGIP-MOER-PACW` |
| CAISO – SDG&E | `USCA-SGIP-SGRT-SDGE`, `USCA-SGIP-SGFC-SDGE`, `USCA-SGIP-SGHT-SDGE` | `USCA-SGIP-MOER-SDGE` |
| CAISO – PG&E | `USCA-SGIP-SGRT-PGE`, `USCA-SGIP-SGFC-PGE`, `USCA-SGIP-SGHT-PGE` | `USCA-SGIP-MOER-PGE` |
| BANC – P2 | `USCA-SGIP-SGRT-BANC`, `USCA-SGIP-SGFC-BANC`, `USCA-SGIP-SGHT-BANC` | `USCA-SGIP-MOER-P2` |
| Turlock Irrigation District | `USCA-SGIP-SGRT-TID`, `USCA-SGIP-SGFC-TID`, `USCA-SGIP-SGHT-TID` | `USCA-SGIP-MOER-TID` |
| BANC – SMUD | `USCA-SGIP-SGRT-SMUD`, `USCA-SGIP-SGFC-SMUD`, `USCA-SGIP-SGHT-SMUD` | `USCA-SGIP-MOER-SMUD` |
| Imperial Irrigation District | `USCA-SGIP-SGRT-IID`, `USCA-SGIP-SGFC-IID`, `USCA-SGIP-SGHT-IID` | `USCA-SGIP-MOER-IID` |
| NV Energy | `USCA-SGIP-SGRT-NVENERGY`, `USCA-SGIP-SGFC-NVENERGY`, `USCA-SGIP-SGHT-NVENERGY` | `USCA-SGIP-MOER-NVENERGY` |
| Western Area Lower Colorado | `USCA-SGIP-SGRT-WALC`, `USCA-SGIP-SGFC-WALC`, `USCA-SGIP-SGHT-WALC` | `USCA-SGIP-MOER-WALC` |
| CAISO – SCE | `USCA-SGIP-SGRT-SCE`, `USCA-SGIP-SGFC-SCE`, `USCA-SGIP-SGHT-SCE` | `USCA-SGIP-MOER-SCE` |
| LADWP | `USCA-SGIP-SGRT-LADWP`, `USCA-SGIP-SGFC-LADWP`, `USCA-SGIP-SGHT-LADWP` | `USCA-SGIP-MOER-LADWP` |

### Code update example

```python
import requests

BASE_URL = "https://midasapi.energy.ca.gov"

# v1.0 — three separate calls for current + forecast + historical
# ghg_now  = requests.get(url, params={"ID": "USCA-SGIP-SGRT-SMUD", ...})
# ghg_fc   = requests.get(url, params={"ID": "USCA-SGIP-SGFC-SMUD", ...})
# ghg_hist = requests.get(url, params={"ID": "USCA-SGIP-SGHT-SMUD", ...})

# v2.0 — one call, continuous 72-hour series (past + present + forecast blended)
resp = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"ID": "USCA-SGIP-MOER-SMUD", "QueryType": "realtime"}
)
data = resp.json()

# Values are now in g/kWh CO2
for entry in data["ValueInformation"]:
    ghg_g_per_kwh = entry["Value"]          # e.g. 333.8 (was ~0.334 in v1.0)
    ghg_kg_per_kwh = ghg_g_per_kwh / 1000   # convert back if needed for legacy comparison
    print(f"{entry['TimeStart']}: {ghg_g_per_kwh:.1f} g/kWh CO2")
```

---

## 4. Flex Alert RIN Migration

### What changed

Three Flex Alert RINs (`FXRT`, `FXFC`, `FXHT`) are replaced by one unified RIN: `USCA-FLEX-ALRT-0000`. The `FXHT` RIN was previously non-functional; this is its first working replacement.

The new RIN delivers a clean hourly binary series: `Value=1` (Flex Alert active), `Value=0` (not active). Every hour in the requested window is present — no gaps.

### Before and after

**v1.0 — separate calls for current and forecast, no working historical:**

```python
# Check current status
resp_rt = requests.get(url, params={"ID": "USCA-FLEX-FXRT-0000", "QueryType": "realtime"})

# Check forecast
resp_fc = requests.get(url, params={"ID": "USCA-FLEX-FXFC-0000", "QueryType": "realtime"})

# Historical was broken — returned error
```

**v2.0 — one call for the full picture:**

```python
BASE_URL = "https://midasapi.energy.ca.gov"

# 72-hour window: current status + upcoming hours
resp = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"ID": "USCA-FLEX-ALRT-0000", "QueryType": "realtime"}
)
data = resp.json()

active_hours = [
    entry for entry in data["ValueInformation"]
    if entry["Value"] == 1
]
print(f"Found {len(active_hours)} active Flex Alert hours in the current 72 hours")

# Historical (past 90 days)
resp_hist = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"ID": "USCA-FLEX-ALRT-0000", "QueryType": "alldata"}
)
```

---

## 5. RINList Response Format Change

The `GET /api/valuedata?SignalType=N` response format changed from a bare array to a keyed object.

```python
# v1.0 response was a list
data = resp.json()
# data = [{"RateID": "...", ...}, ...]

for rin in data:
    print(rin["RateID"])

# v2.0 response is a keyed object
data = resp.json()
# data = {"Rates": [{"RateID": "...", ...}, ...]}

key = list(data.keys())[0]   # "Rates", "GHGEmissions", "FlexAlerts", or "All"
for rin in data[key]:
    print(rin["RateID"])

# More explicit:
for rin in data.get("Rates", []):
    print(rin["RateID"])
```

The `SignalType` field value changed from `"Rates"` to `"Electricity Rates"` for electricity rate RINs.

---

## 6. Query Window Changes

### Realtime window

The `realtime` query now always returns a consistent **72-hour window starting at midnight Pacific on the request date** — regardless of signal type. In v1.0, the GHG `SGRT` RIN returned only a single data point, and Flex Alert was a live pass-through with no defined window.

If your code expected a single data point from the GHG realtime RIN, update it to process a time series:

```python
# v1.0 — SGRT returned one point
resp = requests.get(url, params={"ID": "USCA-SGIP-SGRT-SMUD", "QueryType": "realtime"})
current_value = resp.json()["ValueInformation"][0]["Value"]

# v2.0 — MOER returns a full 72-hour series; get the most recent past value
import datetime

resp = requests.get(
    f"{BASE_URL}/api/valuedata",
    params={"ID": "USCA-SGIP-MOER-SMUD", "QueryType": "realtime"}
)
entries = resp.json()["ValueInformation"]

now_utc = datetime.datetime.now(datetime.timezone.utc)
past_entries = [
    e for e in entries
    if datetime.datetime.fromisoformat(f"{e['DateStart']}T{e['TimeStart']}+00:00") <= now_utc
]
current_value = past_entries[-1]["Value"] if past_entries else None
```

### Alldata window

`alldata` now returns a fixed **90-day window** ending at 23:59:59 Pacific on Day+2. Previously there was no defined cap and response sizes were variable.

For data beyond 90 days, use the historical endpoint:

```python
resp = requests.get(
    f"{BASE_URL}/api/historicaldata/USCA-SGIP-MOER-PGE",
    params={"startdate": "2025-01-01", "enddate": "2025-03-31"}
)
```

Maximum range per `historicaldata` request: 6 months.

---

## 7. Removed Endpoints

Remove any calls to these endpoints — they return `404 Not Found` in v2.0:

- `GET /api/historicallist` — The full list of historical RINs is no longer served as a separate endpoint. Use `GET /api/valuedata?SignalType=0` for the full active RIN list.
- `GET /api/valuedata?LookupTable=Holiday`
- `GET /api/valuedata?LookupTable=TimeZone`

---

## 8. Practical Migration Checklist

Work through this list before June 22, 2026:

- [ ] **GHG thresholds:** Search your codebase for any numeric GHG comparisons and multiply all `kg/kWh CO2` values by 1,000.
- [ ] **GHG unit labels:** Update display strings from `kg/kWh CO2` to `g/kWh CO2`.
- [ ] **GHG stored data:** If you have a database of historical GHG values from MIDAS v1.0, apply a 1,000× migration.
- [ ] **SGRT/SGFC/SGHT RINs:** Find and replace all 33 old GHG RINs with their MOER equivalents.
- [ ] **FXRT/FXFC/FXHT RINs:** Replace with `USCA-FLEX-ALRT-0000`.
- [ ] **RINList parsing:** Update code that reads the list response to expect a keyed object.
- [ ] **alldata window:** Audit any code that assumes a specific number of records in `alldata` responses.
- [ ] **Removed endpoints:** Remove or comment out calls to `HistoricalRINList`, `Holiday`, and `TimeZone` endpoints.
- [ ] **Token removal (optional):** Simplify code by removing token calls for public GET endpoints.
- [ ] **Base URL:** Confirm you are pointing at `https://midasapi.energy.ca.gov` (not the pre-release AppRunner URL).

---

## 9. Support

For questions or migration help: **midas@energy.ca.gov**

---

*California Energy Commission | MIDAS v2.0 Change Guide for Data Consumers | June 22, 2026*

<!-- markdownlint-enable MD031 MD034 MD036 -->
