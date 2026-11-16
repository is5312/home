# Implementation Plan: Global Market-State-Aware Position Valuation

**Branch**: `001-global-market-valuation` | **Date**: 2026-11-14 | **Spec**: [spec.md](./spec.md)

**Input**: Feature specification from `/specs/001-global-market-valuation/spec.md`

## Summary

Build a valuation service that turns Bloomberg reference and real-time data into three separate
values per instrument: a validated intraday `live_mark`, a versioned `primary_official_close`, and
a policy-selected `valuation_mark`. Market state is resolved from feed health, instrument status,
venue phase and dated, eligibility-keyed venue rules. Ticks pass an event router, a venue-aware
scrubber and a corporate-action-aware anomaly gate that never auto-promotes an unconfirmed price.
The plan contains every fix from the three design reviews (spec Appendix A); field and venue facts
are in [research.md](./research.md).

## Technical Context

**Language/Version**: Python 3.12 (decision R10; Java 21 is the fallback if load exceeds targets)

**Primary Dependencies**: official `blpapi` Python SDK; `pydantic` v2 (config and contracts);
`zoneinfo` (IANA time zones); `asyncpg`; `prometheus-client`; `structlog`

**Storage**: PostgreSQL 16 (close store, quarantine cases, audit); YAML in git (venue rules, field
registry, condition maps, special sessions)

**Testing**: `pytest` with a replay harness that feeds recorded or synthetic `blpapi` messages;
`hypothesis` for state-machine properties

**Target Platform**: Linux server with B-PIPE connectivity

**Project Type**: single service (library core + service runner + CLI tools)

**Performance Goals**: ≤ 20,000 instruments; ≤ 5,000 updates/s peak; tick-to-mark p99 ≤ 50 ms

**Constraints**: no fixed UTC schedules; no unverified fields in production; no auto-promotion;
subscription count within B-PIPE entitlement

**Scale/Scope**: ~40 venues at launch; equities, listed futures, FX, government and corporate bonds

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate | Pre-research | Post-design |
|---|---|---|---|
| I. Verified Data Contracts | Field registry with status; start-up check rejects non-`VERIFIED`; enum maps captured, not hard-coded | PASS (design) | PASS: `contracts/field-registry.schema.yaml`, FR-001–FR-003, tasks T005, T006, T011, T018, T078 |
| II. Effective-Dated Market Rules | All rules dated, eligibility-keyed, IANA time | PASS | PASS: `contracts/venue-rule.schema.yaml`, Appendix B rules |
| III. Observation Separate From Valuation | Three distinct values | PASS | PASS: `data-model.md` entities Mark, OfficialCloseRecord |
| IV. Fail Safe, Never Silent | Quality states, counters, no auto-promotion | PASS | PASS: `contracts/mark-output.schema.json`, quarantine state machine |
| V. Replay-Tested Behaviour | Tests first; replay harness | PASS | PASS: replay harness is a foundational task (T012) |

No violations. Complexity Tracking is empty.

## Architecture

```text
[1] Ingestion
    blpapi session ──► message decoder ──► field cache (per field: value, updated_at, seq, event)
    session events ──► feed-health tracker (DataLoss, SlowConsumerWarning, SubscriptionTerminated)

[2] Event router (by MKTDATA_EVENT_TYPE / SUBTYPE)
    STATUS  ──► market-state resolver
    TRADE   ──► trade scrubber ──► anomaly gate ──► live_mark
    CANCEL / CORRECTION ──► mark reverter
    QUOTE   ──► quote scrubber ──► mid candidate ──► anomaly gate
    CLOSE / SETTLE ──► versioned close store

[3] Market-state resolver (strict order)
    feed health ► instrument restriction ► venue phase (captured enum map) ► dated schedule + calendar
    disagreement ──► CALENDAR_FEED_CONFLICT + venue policy

[4] Valuation
    live_mark | primary_official_close | valuation_mark (book policy) ──► mark publisher
    value = local price × quantity × FX (resolved independently)

[5] Control plane
    per-venue bootstrap (IANA local time) · rule loader · field-registry gate · telemetry · ops CLI
```

### Design rules carried from the reviews

1. **Partial updates**: never combine fields from different events. Trade price and size come from
   the same trade event (`EVT_TRADE_PRICE_RT`, `EVT_TRADE_SIZE_RT`); duplicates and cancels match on
   `EVT_TRADE_IDENTIFIER_RT`.
2. **Primary vs composite**: session state and official close from the primary subscription;
   intraday trades may come from a verified composite. Condition maps are keyed on
   `EVT_TRADE_LOCAL_EXCH_SOURCE_RT`. Korea: KRX and Nextrade are separate subscriptions until a
   composite is verified.
3. **Scrubber**: drop size ≤ 0 and indicative prices; keep auction uncross prints; venue condition
   map for off-book and late prints; quote checks (crossed, locked, spread, side age);
   non-positive prices allowed for futures, spreads and rates.
4. **Anomaly gate**: first trade vs CA-adjusted reference (`expected_ex_ref_price`, anomaly use
   only); later trades vs last trusted `live_mark`; tier thresholds (default 5% liquid, 10% small
   cap); exits: two distinct trade IDs in band within 2 s, or fresh two-sided quotes containing the
   price for 5 s with both sides newer than the candidate; TTL 30 s → hold, `QUARANTINE_TIMEOUT`,
   alert; audited manual promote.
5. **Close latch**: `PX_OFFICIAL_CLOSE_RT` / `OFFICIAL_CLOSE_AUCTION_PRICE_RT` from the primary
   venue, date-checked with `TRADING_DT_REALTIME`; revisions accepted until the venue's late window
   ends; then a reference request whose result must be dated today; otherwise hold and flag stale.
   Holidays use the stored last official close. `PX_CLOSE_1D` is not used for these decisions.
6. **Schedules**: zero default buffer; all auction cut-offs from dated venue rules; bootstrap per
   venue in local time.
7. **Multi-asset**: futures via `FUT_TRADING_HRS` and settlement fields; bonds via ordered pricing
   sources with staleness TTL; FX 24/5 with onshore exceptions.

## Rollout

| Phase | Exit gate |
|---|---|
| 0. Field evidence | FLDS / `//blp/apiflds` evidence matrix (8 columns: mnemonic, field ID, service, security coverage, entitlement check, raw enum map, date/time-zone semantics, null behaviour) complete for every registry entry across representative instruments (US/EU/APAC equities, KRX/NXT, NSE CAS/non-CAS, CME/OSE futures, BVAL/CBBT bonds, FX spot/forwards) |
| 1. Capture | Enum and condition-code value maps captured per venue for 5 trading days |
| 2. Shadow | ≥ 10 trading days incl. one month-end and the 2026-12-07 NSE change if in window; drop-reason counters reviewed |
| 3. Acceptance | SC-001 – SC-007 pass; capacity check (FR-042) pass |
| 4. Cutover | Region order chosen from shadow results; one region at a time; rollback = switch book policy source |

## Project Structure

### Documentation (this feature)

```text
specs/001-global-market-valuation/
├── plan.md
├── research.md
├── data-model.md
├── quickstart.md
├── contracts/
│   ├── field-registry.schema.yaml
│   ├── venue-rule.schema.yaml
│   └── mark-output.schema.json
├── checklists/requirements.md
└── tasks.md
```

### Source Code (repository root)

```text
config/
├── field_registry.yaml
├── venues/                 # one file per venue, dated rules + sources
├── condition_maps/         # one file per reporting venue
├── enum_maps/              # captured status value maps
└── special_sessions.yaml

src/shefa/
├── registry/               # field registry + start-up gate
├── ingest/                 # blpapi session, decoder, field cache, feed health
├── router/                 # event classification
├── rules/                  # venue rule model, loader, schedule resolver, calendar
├── state/                  # market-state resolver
├── scrub/                  # trade and quote scrubbers
├── anomaly/                # CA reference, quarantine state machine
├── closes/                 # versioned close store, latch, fallback
├── valuation/              # marks, book policy, FX decoupling, publisher
├── assets/                 # futures, bonds, FX specifics
├── bootstrap/              # per-venue local-time scheduler
├── telemetry/              # metrics, alerts
└── cli/                    # ops: promote, capture, evidence, report

tests/
├── replay/fixtures/        # recorded and synthetic message streams
├── unit/
├── integration/
└── contract/
```

**Structure Decision**: single Python service with a library core under `src/shefa/`. Config is
data, versioned in git, and validated against the schemas in `contracts/`.

## Complexity Tracking

No constitution violations.
