# Datetime and timezone semantics

The MIDAS API mixes two datetime conventions on the wire and does not document
which fields are which. This document records what each datetime field means,
verified empirically against the live API.

> **TL;DR — v1.0.** Fields whose names end in `_UTC` (or whose ISO 8601 strings
> carry a `Z` suffix or `±HH:MM` offset) are UTC. Bare `DateStart`/`TimeStart`
> inside `ValueInformation` are UTC for electricity-rate signals but
> **`America/Los_Angeles` local wall-clock time** for SGIP GHG and Flex Alert
> signals — a known bug carried through from the WattTime and CAISO upstream
> APIs. The CEC does not state this anywhere in the v1.0 documentation.
> Consumers parsing v1.0 GHG or Flex Alert responses must hardcode the PT
> assumption (or, ideally, configure it per-instance with
> `America/Los_Angeles` as the default).
>
> **TL;DR — v2.0 (effective 2026-06-22).** **All datetime fields on the wire
> are UTC**, including the bare `DateStart`/`TimeStart` in `ValueInformation`
> for every signal type. Query window boundaries remain PT-aligned but are
> expressed in UTC: a `realtime` query "today" returns its first datapoint
> with `TimeStart: "07:00:00"` (= midnight Pacific during PDT) or
> `"08:00:00"` (= midnight Pacific during PST). The v1.0 PT-on-wire behavior
> for SGIP GHG and Flex Alert was acknowledged as a bug by the MIDAS team
> and fixed in v2.0 by converting upstream-provider timestamps to UTC before
> delivery. Source: CEC clarification, 2026-06-12.

## Field inventory

### v1.0 (current behavior, until 2026-06-22)

| Field | Format on wire | Zone | Example |
|---|---|---|---|
| `SystemTime_UTC` (ValueData response) | ISO 8601 with `Z` | UTC | `"2026-05-05T15:31:50.0781000Z"` |
| `SignupCloseDate` (ValueData response) | ISO 8601 with `Z` | UTC | `"2024-12-31T00:00:00.000Z"` |
| `LastUpdated` (ValueData RIN-list entries) | Bare ISO 8601, **no zone suffix** | **PT** (`America/Los_Angeles`) | `"2023-06-07T15:57:48.023"` |
| `DateOfHoliday` (Holiday response) | Bare ISO 8601, no zone suffix | **PT** (date portion only is meaningful) | `"2025-12-25T00:00:00"` |
| `DateStart` / `DateEnd` (ValueInformation, electricity-rate RINs) | Bare `YYYY-MM-DD` | **UTC** (calendar date) | `"2026-05-05"` |
| `TimeStart` / `TimeEnd` (ValueInformation, electricity-rate RINs) | Bare `HH:MM:SS` | **UTC** (wall-clock time) | `"08:00:00"` |
| `DateStart` / `DateEnd` (ValueInformation, SGIP GHG and Flex Alert RINs) | Bare `YYYY-MM-DD` | **PT** (calendar date) — upstream-provider passthrough bug | `"2026-05-05"` |
| `TimeStart` / `TimeEnd` (ValueInformation, SGIP GHG and Flex Alert RINs) | Bare `HH:MM:SS` or `HH:MM` | **PT** (wall-clock time) — upstream-provider passthrough bug | `"08:30:00"` |

### v2.0 (effective 2026-06-22)

| Field | Format on wire | Zone | Example |
|---|---|---|---|
| `SystemTime_UTC` (ValueData response) | ISO 8601 with `Z` | UTC | unchanged from v1.0 |
| `SignupCloseDate` (ValueData response) | ISO 8601 with `Z` | UTC | unchanged from v1.0 |
| `LastUpdated` (ValueData RIN-list entries) | Bare ISO 8601, no zone suffix | TBD — verify after release | TBD |
| `DateStart` / `DateEnd` (ValueInformation, **all** signal types) | Bare `YYYY-MM-DD` | **UTC** | first realtime datapoint of "today" carries the UTC date that maps to midnight Pacific |
| `TimeStart` / `TimeEnd` (ValueInformation, **all** signal types) | Bare `HH:MM:SS` | **UTC** | first realtime datapoint is `"07:00:00"` (PDT) or `"08:00:00"` (PST), i.e. midnight Pacific converted to UTC |
| `DateOfHoliday` (Holiday response — if `/Holiday` endpoint is retained) | TBD | TBD | retirement decision pending per CEC |

## How this was verified (v1.0)

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

**CEC confirmation (post-verification):** The CEC MIDAS team confirmed on
2026-06-12 that v1.0 SGIP GHG and Flex Alert RINs were delivered in PT on
the wire as an oversight — those datapoints came directly from the
WattTime and CAISO upstream APIs, both of which operate in Pacific Time,
and MIDAS v1.0 forwarded them without converting to UTC. v1.0
electricity-rate RINs were always UTC. v2.0 fixes the bug by converting
all upstream-provider timestamps to UTC before delivery.

## Consumer guidance

When parsing MIDAS responses:

1. **Z-suffixed fields** (`SystemTime_UTC`, `SignupCloseDate`): parse as
   `OffsetDateTime` / `Instant`. The instant is unambiguous in both v1.0
   and v2.0.

2. **Bare `DateStart`/`DateEnd`/`TimeStart`/`TimeEnd` inside
   `ValueInformation`:**

   - **v2.0 (all signal types)** — parse as UTC. To produce a
     `ZonedDateTime`:

     ```java
     LocalDateTime.of(LocalDate.parse(dateStart), LocalTime.parse(timeStart))
         .atZone(ZoneOffset.UTC)
         .withZoneSameInstant(ZoneId.of("America/Los_Angeles"))  // for PT display
     ```

   - **v1.0 electricity-rate RINs** — parse as UTC (same as above).

   - **v1.0 SGIP GHG and Flex Alert RINs** — parse as
     `America/Los_Angeles` local (upstream-provider bug):

     ```java
     LocalDateTime.of(LocalDate.parse(dateStart), LocalTime.parse(timeStart))
         .atZone(ZoneId.of("America/Los_Angeles"))
     ```

3. **Other bare fields** (`LastUpdated`, `DateOfHoliday`): empirically PT
   in v1.0. v2.0 behavior for these specific fields was not addressed by
   the CEC explicitly — to be re-verified after the release. Until
   verified, continue treating them as `America/Los_Angeles` local.

4. **Display in another zone**: parse first, then `atZoneSameInstant` to
   the target zone. The instant is preserved; only the wall-clock
   representation changes.

### Why `ZonedDateTime`, not `Instant` or `OffsetDateTime`

For arithmetic that involves local-day boundaries (operating-day windows,
DST-transition days, calendar-aware "next billing period"), only
`ZonedDateTime` carries the IANA zone rules needed to be correct.
`OffsetDateTime` carries a fixed offset (e.g. `-08:00`) and does not know
about DST transitions; `.plusDays(1)` on a PT timestamp lands on the wrong
wall-clock time across spring-forward / fall-back.

### DST behaviour

MIDAS interval data on a DST-transition day reflects the local 23-hour or
25-hour PT day, regardless of how the timestamps are encoded on the wire.
On 2026-03-08 (spring-forward) a 24-hour TOU schedule will publish 23
hourly intervals; on 2026-11-01 (fall-back) it publishes 25.

For v2.0 (UTC-on-wire), DST handling is straightforward: parse the bare
`DateStart`/`TimeStart` as UTC, convert to PT for display. The instant
arithmetic is correct by construction; the 23/25-hour day shows up as a
gap or duplicate in the PT-rendered view.

For v1.0 SGIP/Flex (PT-on-wire bug), code that reads `DateStart` +
`TimeStart` directly and attaches `America/Los_Angeles` is correct across
DST transitions by construction.

## Why the v1.0 inconsistency existed

MIDAS itself always intended bare `DateStart`/`TimeStart` to be UTC, and
the electricity-rate path delivered them that way. The SGIP GHG and Flex
Alert paths, however, were thin pass-throughs of the WattTime and CAISO
upstream APIs respectively; both of those upstreams operate natively in
Pacific Time, and v1.0 forwarded their timestamps unchanged. The CEC has
acknowledged this as an oversight and v2.0 normalizes everything to UTC
by converting upstream-provider timestamps before delivery.

The remaining `LastUpdated` and `DateOfHoliday` PT-on-wire behavior is
likely a similar artifact (administrative timestamps written in PT and
never tagged with a zone) and may or may not be addressed in v2.0 — TBD.

## References

- [MIDAS upstream documentation](https://github.com/california-energy-commission/MIDAS)
  — does not state the timezone convention.
- [grid-coordination/clj-price-server `doc/time-alternatives.md`](https://github.com/grid-coordination/clj-price-server/blob/main/doc/time-alternatives.md)
  — broader context on the `ZonedDateTime` end-to-end discipline adopted
  across the grid-coordination ecosystem.
- [grid-coordination/clj-midas#1](https://github.com/grid-coordination/clj-midas/issues/1)
  — `clj-midas` epic that prompted this verification.
