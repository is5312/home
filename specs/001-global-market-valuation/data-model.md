# Data Model: Global Market-State-Aware Position Valuation

**Date**: 2026-11-14 | **Spec**: [spec.md](./spec.md)

## FieldRegistryEntry

| Attribute | Type | Rule |
|---|---|---|
| mnemonic | string | Unique |
| service | enum `refdata`, `mktdata`, `tick` | |
| status | enum `VERIFIED`, `NAME_FOUND`, `NOT_FOUND`, `OTHER_VENDOR` | Production requires `VERIFIED` (FR-002) |
| field_id | string? | From `//blp/apiflds` |
| meaning | string | From FLDS, not from the name |
| evidence | list of URL or artifact path | ≥ 1 for `VERIFIED` |
| replaces | string? | Name this entry replaces (for example `EVT_TRADE_ID_RT`) |

## Venue

| Attribute | Type | Rule |
|---|---|---|
| mic | string | ISO 10383 |
| bbg_exch_code | string | For example `UN`, `JT` |
| iana_tz | string | Valid `zoneinfo` key |
| calendar_code | string | `SETTLEMENT_CALENDAR_CODE` |
| role | enum `PRIMARY`, `REPORTING`, `COMPOSITE` | |
| late_window_min | int | Per venue (FR-030) |
| bootstrap_local_time | time | Local time (FR-016) |

## SessionRule

| Attribute | Type | Rule |
|---|---|---|
| venue_mic | string | FK Venue |
| eligibility | map | For example `{security_typ: Equity, fo_eligible: true, etp: false}` |
| effective_from | date | Required |
| effective_to | date? | Exclusive; no overlap for the same eligibility |
| phases | list of Phase | Ordered, non-overlapping, local time |
| source | URL | Required |

**Phase**: `name` (`PRE_OPEN`, `OPENING_AUCTION`, `CONTINUOUS`, `LUNCH`, `CLOSING_AUCTION`,
`CLOSING_CALL`, `POST_CLOSE`, `AFTER_MARKET`, `CLOSED`), `start`, `end`, `live_marking` (bool),
`random_end_s` (int?, for example 30).

## SpecialSession

`venue_mic`, `date`, `type` (`HALF_DAY`, `SPECIAL`, `CLOSED`), `phases`, `source`.

## Instrument

`instrument_id` (ID_BB_GLOBAL), `asset_class` (from MARKET_SECTOR_DES), `primary_venue`,
`reporting_venues`, `composite_ticker?`, `eligibility` flags, `liquidity_tier`, `currency`.

## FieldCacheEntry

`subscription_id`, `field`, `value`, `updated_at` (feed time), `received_at`, `event_type`,
`event_subtype`, `seq`. A value is never combined with a value from another event (FR-006).

## FeedHealth

`subscription_id`, `state` (`OK`, `SLOW`, `DATA_LOSS`, `TERMINATED`, `SILENT`), `since`, `last_message_at`.

## MarketState

`instrument_id`, `state` (`CONTINUOUS`, `AUCTION`, `HALTED`, `CLOSED`, `EXTENDED`, `UNKNOWN`),
`deciding_tier` (`FEED_HEALTH`, `INSTRUMENT`, `VENUE_PHASE`, `SCHEDULE`), `reason`, `as_of`.

Transitions: any state → `UNKNOWN` when feed health is not `OK`; `CLOSED` ↔ `CONTINUOUS` only
through rule phases or mapped venue status; conflict between tiers 3 and 4 → alert
`CALENDAR_FEED_CONFLICT`.

## QuarantineCase

`case_id`, `instrument_id`, `candidate_price`, `reference_price`, `reference_kind`
(`CA_ADJUSTED_PRIOR_CLOSE`, `LIVE_MARK`), `opened_at`, `confirming_trade_ids`,
`quote_confirm_since?`, `outcome` (`PROMOTED_TRADES`, `PROMOTED_QUOTES`, `TIMEOUT`,
`MANUAL_PROMOTE`, `SUPERSEDED`), `closed_at`, `actor?` (manual only).

State machine: `OPEN` → `PROMOTED_TRADES` | `PROMOTED_QUOTES` | `TIMEOUT`; `TIMEOUT` →
`MANUAL_PROMOTE` (audited) | `SUPERSEDED` (new trusted mark).

## OfficialCloseRecord

| Attribute | Type | Rule |
|---|---|---|
| instrument_id, primary_venue, session_date, price_type | key | `price_type`: `OFFICIAL_CLOSE`, `CLOSING_AUCTION`, `SETTLEMENT`, `REFERENCE_FALLBACK`, `CARRIED_FORWARD` |
| revision | int | Increments per update |
| price | decimal | |
| source_field | string | Registry entry |
| received_at | timestamp | |
| final | bool | True after late window or fallback |

## Mark

`instrument_id`, `kind` (`LIVE`, `VALUATION`), `price`, `source`, `session_state`,
`quality_state` (`OK`, `FROZEN_CLOSE`, `STALE`, `HALTED`, `QUARANTINED`, `QUARANTINE_TIMEOUT`,
`FEED_DEGRADED`, `CALENDAR_FEED_CONFLICT`), `is_stale`, `age_s`, `book_policy?`, `as_of`.

## BookPolicy

`book_id`, `policy` (`FLASH_INTRADAY`, `OFFICIAL_EOD`), `fx_source`, `custodian_tolerance_bps`.
