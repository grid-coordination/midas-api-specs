# Datetime and timezone semantics

The MIDAS API mixes two datetime conventions on the wire and does not document
which fields are which. This document records what each datetime field means,
verified empirically against the live API.

> **TL;DR.** Fields whose names end in `_UTC` (or whose ISO 8601 strings carry
> a `Z` suffix or `±HH:MM` offset) are UTC. Every other datetime field is
> **`America/Los_Angeles` local wall-clock time** — Pacific Standard Time in
> winter, Pacific Daylight Time in summer. The CEC does not state this in
> the API documentation. Consumers must hardcode it (or, ideally, configure
> it per-instance with `America/Los_Angeles` as the default).

## Field inventory

| Field | Format on wire | Zone | Example |
|---|---|---|---|
| `SystemTime_UTC` (ValueData response) | ISO 8601 with `Z` | UTC | `"2026-05-05T15:31:50.0781000Z"` |
| `SignupCloseDate` (ValueData response) | ISO 8601 with `Z` | UTC | `"2024-12-31T00:00:00.000Z"` |
| `LastUpdated` (ValueData RIN-list entries) | Bare ISO 8601, **no zone suffix** | **PT** (`America/Los_Angeles`) | `"2023-06-07T15:57:48.023"` |
| `DateOfHoliday` (Holiday response) | Bare ISO 8601, no zone suffix | **PT** (date portion only is meaningful) | `"2025-12-25T00:00:00"` |
| `DateStart` / `DateEnd` (ValueInformation) | Bare `YYYY-MM-DD` | **PT** (calendar date) | `"2026-05-05"` |
| `TimeStart` / `TimeEnd` (ValueInformation) | Bare `HH:MM:SS` or `HH:MM` | **PT** (wall-clock time) | `"08:30:00"` |

## How this was verified

At UTC = `2026-05-05T15:31:50Z` (= 08:31:50 PDT, since California was on
Pacific Daylight Time, UTC−7), the following two real-time signals were
fetched simultaneously:

```
GET /ValueData?ID=USCA-SGIP-SGRT-PGE&QueryType=realtime
  → SystemTime_UTC: "2026-05-05T15:31:32.6133062Z"
  → ValueInformation[0]: { DateStart: "2026-05-05", TimeStart: "08:30",
                            ValueName: "SGIP GHG Realtime", ... }

GET /ValueData?ID=USCA-FLEX-FXRT-0000&QueryType=realtime
  → SystemTime_UTC: "2026-05-05T15:31:50.0781000Z"
  → ValueInformation[0]: { DateStart: "2026-05-05", TimeStart: "08:31",
                            ValueName: "No Active Flex Alert", ... }
```

Both real-time feeds publish a current-interval marker. If the bare
`TimeStart` were UTC, both signals would have read `15:30` / `15:31`. They
read `08:30` / `08:31`, which matches PDT (UTC−7) at that instant.

Conversely, `SystemTime_UTC` agreed (to the second) with Java's
`Instant.now()` UTC clock, confirming the `Z` suffix is honest.

## Consumer guidance

When parsing MIDAS responses:

1. **Z-suffixed fields** (`SystemTime_UTC`, `SignupCloseDate`): parse as
   `OffsetDateTime` / `Instant`. The instant is unambiguous.

2. **Bare fields** (`LastUpdated`, `DateOfHoliday`, `DateStart`/`DateEnd`,
   `TimeStart`/`TimeEnd`): treat as `America/Los_Angeles` local. To produce
   a `ZonedDateTime` from a bare datetime string `s`, use:

   ```java
   LocalDateTime.parse(s).atZone(ZoneId.of("America/Los_Angeles"))
   ```

   To produce one from `DateStart` + `TimeStart`:

   ```java
   LocalDateTime.of(LocalDate.parse(dateStart), LocalTime.parse(timeStart))
       .atZone(ZoneId.of("America/Los_Angeles"))
   ```

3. **Display in another zone**: parse first, then `atZoneSameInstant` to the
   target zone. The instant is preserved across the conversion; only the
   wall-clock representation changes.

### Why `ZonedDateTime`, not `Instant` or `OffsetDateTime`

For arithmetic that involves local-day boundaries (operating-day windows,
DST-transition days, calendar-aware "next billing period"), only
`ZonedDateTime` carries the IANA zone rules needed to be correct.
`OffsetDateTime` carries a fixed offset (e.g. `-08:00`) and does not know
about DST transitions; `.plusDays(1)` on a PT timestamp lands on the wrong
wall-clock time across spring-forward / fall-back.

### DST behaviour

MIDAS interval data on a DST-transition day reflects the local 23-hour or
25-hour day. On 2026-03-08 (spring-forward) a 24-hour TOU schedule will
publish 23 hourly intervals; on 2026-11-01 (fall-back) it publishes 25.
Code that infers wall-clock times from `SystemTime_UTC + N × duration`
gets this wrong; code that reads `DateStart` + `TimeStart` directly and
attaches `America/Los_Angeles` is correct by construction.

## Why the inconsistency exists

The MIDAS API is a California-only system serving California utilities,
all of which operate in the same zone. From the CEC's perspective, "the
zone" is implicit context that does not need to be transmitted. The
`_UTC` suffix on `SystemTime_UTC` exists because that field is a server
timestamp where instant-correctness matters across infrastructure; the
operating-day data fields are wall-clock by domain convention.

This is a reasonable design choice for a single-zone single-tenant API, but
is undocumented. Multi-state or multi-zone consumers (or any consumer
deriving an `Instant` from a bare field) must compensate.

## References

- [MIDAS upstream documentation](https://github.com/california-energy-commission/MIDAS)
  — does not state the timezone convention.
- [grid-coordination/clj-price-server `doc/time-alternatives.md`](https://github.com/grid-coordination/clj-price-server/blob/main/doc/time-alternatives.md)
  — broader context on the `ZonedDateTime` end-to-end discipline adopted
  across the grid-coordination ecosystem.
- [grid-coordination/clj-midas#1](https://github.com/grid-coordination/clj-midas/issues/1)
  — `clj-midas` epic that prompted this verification.
