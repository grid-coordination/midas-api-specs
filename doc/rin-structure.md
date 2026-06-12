# Rate Identification Number (RIN)

The Rate Identification Number (RIN) is a compound identifier that uniquely identifies a rate schedule, GHG emission signal, or Flex Alert signal in the CEC's [MIDAS](https://midasapi.energy.ca.gov/) system.

RINs were introduced as part of the California [Load Management Standards](https://www.energy.ca.gov/programs-and-topics/topics/load-flexibility/load-management-standards) (CEC Docket 23-LMS-01). Since April 2024, California utilities are required to print each customer's RIN on their bill in both text and QR code format, enabling machine-readable access to their rate schedule via the MIDAS API.

## Format

A RIN consists of four hyphen-delimited segments encoding six fields:

```
CCSS-DDEE-RRRR-LLLLLLLLLL
```

| Segment | Fields | Characters | Description |
|---------|--------|------------|-------------|
| 1 | Country + State | 2 + 2 | Country and state codes |
| 2 | Distribution + Energy | 2 + 2 | Distribution utility code + energy provider code |
| 3 | Rate | 4 | Rate schedule or signal type identifier |
| 4 | Location | 1–10 | Location or instance identifier |

**Regex** (from the [JSON Schema](../apis/value-data/schemas/midas-rin-list-entry.schema.json)):

```
^[A-Z]{2}[A-Z]{2}-[A-Z0-9]{2}[A-Z0-9]{2}-[A-Z0-9]{4}-[A-Z0-9]{4,10}$
```

> **Figure reference**: The [CEC MIDAS documentation](https://github.com/california-energy-commission/MIDAS) includes a diagram (Figure 1) illustrating the RIN structure using the example `USCA-PGPG-TOU4-0000000000`.

## Segment Details

### Segment 1: Country + State (4 letters)

The first two characters are a country code and the next two are a state code. In practice, all current RINs begin with `USCA` (United States, California). The format allows for expansion to other jurisdictions, though no non-California RINs are known to exist.

### Segment 2: Distribution + Energy (4 alphanumeric)

Encodes the distribution utility (first 2 characters) and the energy provider (last 2 characters). In California's unbundled electricity market, the company that owns the wires (distribution) can differ from the company that provides generation (energy).

**Bundled customers** (same company for distribution and generation):

| Code | Utility |
|------|---------|
| `PGPG` | Pacific Gas & Electric (PG&E) |
| `SCSC` | Southern California Edison (SCE) |
| `SDSD` | San Diego Gas & Electric (SDG&E) |
| `SMSM` | Sacramento Municipal Utility District (SMUD) |
| `LALA` | Los Angeles Department of Water and Power (LADWP) |
| `BNBN` | City of Banning |

**CCA (Community Choice Aggregation) customers** have different distribution and energy codes because the distribution utility and the generation provider are separate entities:

| Code | Distribution | Energy Provider |
|------|-------------|-----------------|
| `SDEA` | SDG&E | Clean Energy Alliance |

CCA customers receive **two RINs** on their bill — one for the distribution component and one for the generation component.

The valid codes can be queried from the MIDAS API's lookup tables (`Distribution` and `Energy`).

### Segment 3: Rate (4 alphanumeric)

Identifies the rate schedule or signal type. Known patterns:

**Electricity rates:**

| Code | Description |
|------|-------------|
| `TTOU`, `TOU4`, `ETOU` | Time-of-use rate schedules |
| `EVT2` | Event-based rate |
| `TEST` | Test rate |

**Flex Alert signals** (from CAISO):

| Code | Description | Versions |
|------|-------------|----------|
| `FXRT` | Real-time | v1.0 only |
| `FXFC` | Forecast | v1.0 only |
| `FXHT` | Historical | v1.0 only (non-functional) |
| `ALRT` | Unified Flex Alert (binary hourly: 1=active, 0=inactive) | v2.0 |

**SGIP GHG emissions** (from WattTime):

| Code | Description | Versions |
|------|-------------|----------|
| `SGRT` | Real-time | v1.0 only |
| `SGFC` | Forecast | v1.0 only |
| `SGHT` | Historical | v1.0 only |
| `MOER` | Marginal Operating Emissions Rate (combines past observed, current, and forecast in one continuous 5-minute series) | v2.0 |

> **v2.0 consolidation** (effective 2026-06-22): The three `SG*` GHG codes are replaced by a single `MOER` code per region, and the three `FX*` Flex Alert codes are replaced by a single `ALRT` code. The v2.0 `realtime` query returns a 72-hour window covering past, present, and forecast in one call. See [doc/v2-migration.md](v2-migration.md) and the [Flex Alerts doc](flex-alerts.md).

### Segment 4: Location (1–10 alphanumeric)

A location or instance identifier. For most rate RINs this is `0000` or `0000000000`. For GHG emissions RINs, the location identifies a [WattTime grid region](https://sgipsignal.com/grid-regions) — there are 11 regions across California, yielding 33 v1.0 GHG RINs (real-time, forecast, and historical for each region) and 11 v2.0 `MOER` RINs (one per region).

**SGIP GHG region codes** (used in segment 4 of `MOER` RINs, and in the v1.0 `SG*` RINs):

| Code | Region |
|------|--------|
| `PACW` | PacifiCorp West |
| `SDGE` | CAISO — SDG&E |
| `PGE` | CAISO — PG&E |
| `P2` | BANC — P2 (v2.0; v1.0 used `BANC` for the same region) |
| `TID` | Turlock Irrigation District |
| `SMUD` | BANC — SMUD |
| `IID` | Imperial Irrigation District |
| `NVENERGY` | NV Energy |
| `WALC` | Western Area Lower Colorado |
| `SCE` | CAISO — SCE |
| `LADWP` | LADWP |

## Special RINs

Certain RINs have fixed, well-known values:

| RIN | Description | Versions |
|-----|-------------|----------|
| `USCA-FLEX-ALRT-0000` | Flex Alert — unified binary hourly signal | v2.0 |
| `USCA-SGIP-MOER-{region}` | GHG emissions for a WattTime region — combined past/current/forecast | v2.0 |
| `USCA-FLEX-FXRT-0000` | Flex Alert — real-time (pass-through from CAISO) | v1.0 |
| `USCA-FLEX-FXFC-0000` | Flex Alert — forecast (pass-through from CAISO) | v1.0 |
| `USCA-FLEX-FXHT-0000` | Flex Alert — historical (non-functional in v1.0) | v1.0 |
| `USCA-SGIP-SGxx-{region}` | GHG emissions for a WattTime region (`xx` = RT/FC/HT) | v1.0 |
| `USCA-TSTS-TTOU-TEST` | Test rate (used in API documentation examples) | both |

## Examples

| RIN | Description |
|-----|-------------|
| `USCA-PGPG-TOU4-0000` | PG&E bundled customer, TOU rate schedule TOU4 |
| `USCA-SDSD-ETOU-0000` | SDG&E bundled customer, Energy TOU rate |
| `USCA-LALA-TTOU-0000` | LADWP, TOU rate |
| `USCA-BNBN-EVT2-0000` | City of Banning, event-based rate |
| `USCA-SDEA-TTOU-0000` | SDG&E distribution / Clean Energy Alliance generation |
| `USCA-TSTS-TTOU-TEST` | Test distributor / test energy / test rate |

## Properties

- **Stable**: RINs do not change over time. Rate values change, but the RIN remains the same unless a customer changes tariffs or switches providers.
- **Assigned on first upload**: RINs are assigned when rate information is first uploaded by the LSE through the MIDAS API. Subsequent uploads to the same rate must use the same RIN.
- **Signal type association**: Each RIN is associated with a signal type — Rates (1), GHG (2), or Flex Alert (3). These can be queried via the `SignalType` parameter on the `/ValueData` endpoint.
- **On customer bills**: Since April 2024, California utilities must print the customer's RIN on their bill in text and QR code format per the Load Management Standards.
- **Description field**: RIN list entries include a human-readable `Description` field (e.g., "Rate Data for Distributor: LADWP, Energy Company: LADWP").

## Regulatory Context

MIDAS was developed to support the CEC's [Load Management Standards](https://www.energy.ca.gov/programs-and-topics/topics/load-flexibility/load-management-standards) (California Code of Regulations, Title 20, Sections 1621–1625). The standards require the state's largest IOUs, POUs, and CCAs to populate the MIDAS database with all time-varying rate information.

The CPUC's [Demand Flexibility Rulemaking (R.22-07-005)](https://www.cpuc.ca.gov/industries-and-topics/electrical-energy/electric-costs/demand-response-dr/demand-flexibility-rulemaking) complements this by advancing dynamic pricing and demand flexibility through electric rates. Utilities are implementing billing system enhancements to compute and upload prices to MIDAS and to place RINs on customer bills.

The Load Management Standards also require utilities and large CCAs to collaboratively develop a single statewide standard tool for authorized third-party rate data access (Section 1623(c)), due 18 months after the regulation effective date.

## Sources

- [CEC MIDAS GitHub Repository](https://github.com/california-energy-commission/MIDAS) — primary documentation with RIN structure diagram (Figure 1)
- [CEC Load Management Standards](https://www.energy.ca.gov/programs-and-topics/topics/load-flexibility/load-management-standards) — regulatory mandate for MIDAS and RINs
- [CEC Docket 23-LMS-01](https://efiling.energy.ca.gov) — Load Management Standards compliance filings
- [CPUC R.22-07-005](https://www.cpuc.ca.gov/industries-and-topics/electrical-energy/electric-costs/demand-response-dr/demand-flexibility-rulemaking) — Demand Flexibility Rulemaking
- [MIDAS API Help](https://midasapi.energy.ca.gov/Help) — upstream API documentation
- [Clean Energy Alliance — RIN](https://thecleanenergyalliance.org/rin/) — CCA dual-RIN example
- [SMUD — Rate Identification Number](https://www.smud.org/Customer-Support/Understand-Your-Bill/Rate-Identification-Number) — consumer-facing explanation
- [MyKWhNow — Help me find my RIN](https://mykwhnow.com/blog/post-008-help-me-find-my-RIN) — consumer-facing guide
