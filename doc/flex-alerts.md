# Flex Alerts

Flex Alerts are a call to consumers to voluntarily conserve energy when demand for power could outstrip supply. They are declared by the California ISO (CAISO) whenever there is expected stress on the grid, typically during heatwaves when electrical demand is high.

## Escalation Ladder

Flex Alerts sit at the lowest severity level of CAISO's grid emergency hierarchy:

| Level | Name | Description |
|-------|------|-------------|
| — | **Flex Alert** | Voluntary conservation call to consumers |
| — | **RMO** (Restricted Maintenance Operations) | Generators/transmission operators must postpone planned maintenance |
| — | **Transmission Emergency** | Line or transformer overloads or loss threatening grid capability |
| — | **EEA Watch** | Day-Ahead analysis forecasts energy deficiency |
| 1 | **EEA1** (Energy Emergency Alert 1) | Real-time analysis forecasts energy deficiency |
| 2 | **EEA2** (Energy Emergency Alert 2) | All resources in use; emergency load management needed |
| 3 | **EEA3** (Energy Emergency Alert 3) | Cannot meet reserve requirements; firm load shedding (rolling blackouts) |

## MIDAS RINs

MIDAS provides three Flex Alert RINs, all pass-through from CAISO:

| RIN | Description |
|-----|-------------|
| `USCA-FLEX-FXRT-0000` | **Real-time** — queries CAISO at the moment of the API call; returns whether a Flex Alert is currently active |
| `USCA-FLEX-FXFC-0000` | **Forecast** — queries CAISO for any planned upcoming Flex Alerts |
| `USCA-FLEX-FXHT-0000` | **Historical** — archived Flex Alert records from the HistoricalData table |

### Data shape

Each Flex Alert value interval contains:

| Field | Example | Notes |
|-------|---------|-------|
| Name | `"Statewide Flex Alert 09.05.2022"` | Human-readable label |
| Date | `2022-09-05` | Date of the alert |
| Time window | `16:00–21:00` | Conservation window (typically late afternoon to evening) |
| Price | `1.0` | Binary indicator — `1.0` = alert active, `0.0` = no alert |
| Unit | `"Event"` (real-time) or `""` (historical) | Inconsistent across RIN types |

### Historical data limitations

The MIDAS HistoricalData table contains only **10 Flex Alert records**, all from the **September 2022 heatwave** (August 31 – September 9, 2022). Querying the full date range back to 2000 returns only these 10 days — the 125 Flex Alert days from 2000–2021 were never backfilled into MIDAS.

For a complete historical record, see the [CAISO Grid Emergencies History Report](https://www.caiso.com/documents/grid-emergencies-history-report-1998-to-present.pdf) (PDF), which covers all grid emergency events from 1998 to present.

## Historical Summary

Flex Alerts were first issued in **2000** during the California energy crisis. The table below summarizes Flex Alert days by year (from the CAISO Grid Emergencies History Report):

**Pre-May 2022 system** (1998–April 2022):

| Year | Flex Alert Days | Notable |
|------|----------------|---------|
| 1998–1999 | N/A | Flex Alerts did not yet exist |
| 2000 | 20 | First year of Flex Alerts (energy crisis) |
| 2001 | 26 | Peak of California energy crisis |
| 2002 | 1 | |
| 2003 | 0 | |
| 2004 | 6 | |
| 2005 | 7 | |
| 2006 | 18 | |
| 2007 | 6 | |
| 2008 | 3 | |
| 2009 | 0 | |
| 2010 | 0 | |
| 2011 | 2 | |
| 2012 | 2 | |
| 2013 | 3 | |
| 2014 | 1 | |
| 2015 | 2 | |
| 2016 | 3 | |
| 2017 | 4 | |
| 2018 | 2 | |
| 2019 | 1 | |
| 2020 | 10 | August heatwave |
| 2021 | 8 | |
| **Total** | **125** | |

**Post-May 2022 system** (May 2022–present):

| Year | Flex Alert Days |
|------|----------------|
| 2022 | 11 |
| 2023 | 0 |
| 2024 | 0 |
| 2025 | 0 |
| 2026 | 0 |
| **Total** | **11** |

## Sources

- [CAISO Grid Emergencies History Report (1998–present)](https://www.caiso.com/documents/grid-emergencies-history-report-1998-to-present.pdf) — complete historical record of all grid emergency events including Flex Alerts
- [CAISO Flex Alert website](http://www.flexalert.org) — consumer-facing information, conservation tips, and notifications
- [CEC MIDAS Documentation](https://github.com/california-energy-commission/MIDAS) — Flex Alert RIN descriptions and API usage
