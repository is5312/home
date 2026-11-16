# Research: Global Market-State-Aware Position Valuation

**Date**: 2026-11-14 | **Spec**: [spec.md](./spec.md) | **Plan**: [plan.md](./plan.md)

Evidence was gathered from Bloomberg's public pages, open-source Bloomberg API code (xbbg,
jBloomberg, findatapy, blp, OpenFIGI examples) and exchange or broker publications. Open-source
usage proves that a field name exists. It does not prove its meaning. Every field below still
needs FLDS / `//blp/apiflds` evidence before production (Constitution Principle I).

## R1. Field verification matrix

Status values: `NAME_FOUND` (name seen in Bloomberg API code; FLDS still required),
`NOT_FOUND` (no public evidence), `OTHER_VENDOR` (belongs to another vendor; never use).

| Field | Service | Status | Evidence | Use / note |
|---|---|---|---|---|
| ID_BB_GLOBAL | refdata | NAME_FOUND | OpenFIGI examples | Instrument key |
| ID_MIC_PRIM_EXCH | refdata | NAME_FOUND | xbbg `exchange.rs` | Primary MIC |
| EQY_PRIM_EXCH_SHRT | refdata | NAME_FOUND | xbbg | Primary exchange code |
| EXCH_CODE, COUNTRY_ISO | refdata | NAME_FOUND | xbbg | Venue resolution |
| MARKET_SECTOR_DES | refdata | NAME_FOUND | xbbg | Asset-class router |
| SECURITY_TYP | refdata | NOT_FOUND | — | Check in FLDS; needed for eligibility |
| IANA_TIME_ZONE | refdata | NAME_FOUND | xbbg | Venue time zone |
| TIME_ZONE_NUM | refdata | NAME_FOUND | xbbg | Cross-check only |
| TRADING_DAY_START_TIME_EOD / _END_TIME_EOD | refdata | NAME_FOUND | xbbg | Baseline only; xbbg adds a manual Japan 09:00–15:30 rule |
| FUT_TRADING_HRS | refdata | NAME_FOUND | xbbg | Futures sessions |
| SETTLEMENT_CALENDAR_CODE | refdata | NAME_FOUND | findatapy | Calendar code |
| CALENDAR_NON_SETTLEMENT_DATES | refdata | NAME_FOUND | findatapy | Holidays |
| PX_CLOSE_1D | refdata | NAME_FOUND | finmars Bloomberg provider | Meaning not public; do not use for today's close or holidays |
| PX_OFFICIAL_CLOSE (non-RT) | refdata | NOT_FOUND | — | Only the `_RT` form is found |
| DVD_EX_DT | refdata | NAME_FOUND | several repos | Single date, no amount |
| DVD_HIST_ALL | refdata (bulk) | NAME_FOUND | xbbg | Ex-dates and amounts |
| EQY_SPLIT_DT, EQY_SPLIT_RATIO | refdata | NAME_FOUND | several repos | Split adjustment |
| EQY_SPLIT_ADJ | refdata | NAME_FOUND | Data License field lists | Meaning not public |
| CAPITAL_CHANGE_DT | refdata | NOT_FOUND | — | Do not use |
| RT_TRADING_PERIOD | mktdata | NAME_FOUND | xbbg, jBloomberg | Enum values not public |
| RT_SIMP_SEC_STATUS | mktdata | NAME_FOUND | xbbg, jBloomberg | Enum values not public |
| RT_EXCH_MARKET_STATUS | mktdata | NAME_FOUND | xbbg, jBloomberg | Venue status |
| RT_EXCH_TRADE_STATUS | mktdata | NAME_FOUND | jBloomberg | Instrument status |
| TRADING_PERIOD_TIME | mktdata | NAME_FOUND | jBloomberg | Phase timing |
| PRE_POST_MARKET_TIME | mktdata | NAME_FOUND | jBloomberg | Extended-hours timing |
| TRADING_DT_REALTIME | mktdata | NAME_FOUND | xbbg, jBloomberg | Session date for close check |
| LAST_PRICE, BID, ASK | mktdata | NAME_FOUND | jBloomberg, xbbg | Summary fields only |
| SIZE_LAST_TRADE | mktdata | NAME_FOUND | jBloomberg | Summary field |
| EVT_TRADE_PRICE_RT, EVT_TRADE_SIZE_RT | mktdata | NAME_FOUND | blp quickstart | Trade event values |
| EVT_TRADE_TIME_RT, EVT_TRADE_DATE_RT | mktdata | NAME_FOUND | blp quickstart, B-PIPE capture | Trade timestamp |
| EVT_TRADE_IDENTIFIER_RT | mktdata | NAME_FOUND | blp quickstart, xbbg | Duplicate and cancel matching |
| EVT_TRADE_CONDITION_CODE_RT | mktdata | NAME_FOUND | blp quickstart | Condition codes (venue-specific values) |
| EVT_TRADE_LOCAL_EXCH_SOURCE_RT | mktdata | NAME_FOUND | blp quickstart | Reporting venue |
| PX_OFFICIAL_CLOSE_RT | mktdata | NAME_FOUND | xbbg, Bloomberg Server API demo | Official close |
| OFFICIAL_CLOSE_AUCTION_PRICE_RT | mktdata | NAME_FOUND | xbbg | Closing auction price |
| OFFICIAL_OPEN_AUCTION_PRICE_RT | mktdata | NAME_FOUND | xbbg | Opening auction price |
| IN_AUCTION_RT, AUCTION_TYPE_REALTIME | mktdata | NAME_FOUND | xbbg | Auction state |
| PX_SETTLE_ACTUAL_RT, PX_SETTLE_LAST_DT_RT | mktdata | NAME_FOUND | jBloomberg | Futures settlement candidates |
| conditionCode | IntradayTickRequest element | NAME_FOUND | jBloomberg | Tick history |
| EVT_TRADE_DATE_TIME | — | NOT_FOUND | — | Replace with DATE_RT + TIME_RT |
| EVT_TRADE_ID_RT | — | NOT_FOUND | — | Replace with EVT_TRADE_IDENTIFIER_RT |
| SECURITY_STATUS | — | NOT_FOUND | — | Replace with RT_SIMP_SEC_STATUS |
| PRE_MKT_START_TIME / POST_MKT_END_TIME | — | NOT_FOUND | — | Replace with PRE_POST_MARKET_TIME |
| IS_OFFICIAL_CLOSE | — | NOT_FOUND | — | Use PX_OFFICIAL_CLOSE_RT updates |
| OFFICIAL_CLOSE_RT | — | NOT_FOUND | — | Correct name is PX_OFFICIAL_CLOSE_RT |
| TRADE_CONDITION | — | NOT_FOUND | — | Use EVT_TRADE_CONDITION_CODE_RT |
| MKT_STATUS_IND | — | OTHER_VENDOR | Refinitiv RTSDK | Never use |
| RT_LAST_VOL | — | OTHER_VENDOR | Wind | Never use |
| MMT_FLAGS / MMT_CLASS | — | OTHER_VENDOR | MMT_CLASS is Refinitiv (FIX MMT slides) | Bloomberg MMT field unknown |

Values that are **not confirmed**: all status enum strings (`CONTINUOUS`, `HALT`, `SUSPENDED`,
`AUCTION_EXT`, ...) and all condition-code strings (`IND`, `AUC`, `BLK`, `OFF`, `XT`, `B`,
`OVR`, `ODD`, ...). Decision: capture value maps per venue in shadow mode (T-tasks in Phase 2).

"Security Status 2.0" is a real Bloomberg name for its normalized status logic, not a field.

## R2. Real-time event model

- **Decision**: Treat every `SubscriptionData` message as partial. Route by `MKTDATA_EVENT_TYPE`
  and `MKTDATA_EVENT_SUBTYPE` (`SUMMARY`/`INITPAINT`, `TRADE`, `QUOTE`, status, close).
  Keep a per-field cache with update time and the source message sequence.
- **Rationale**: One API description states that the initial SUMMARY contains all fields and later
  updates contain only changed fields. Event types and INITPAINT appear in open-source Bloomberg code.
- **Alternatives considered**: Rebuild a full record per message (rejected: stale fields look fresh).
- **Open**: Confirm the full list of trade subtypes. `CANCEL` appears in open-source code;
  `CORRECTION` was not found. Capture subtypes in shadow mode.

## R3. Feed health

- **Decision**: Drive `feed_health` from `DataLoss`, `SlowConsumerWarning` (and its cleared
  counterpart) and `SubscriptionTerminated` messages, plus a per-subscription heartbeat age.
- **Rationale**: These messages appear in open-source Bloomberg code; a stale feed makes every
  other status from that feed untrusted, so feed health ranks first (FR-017).

## R4. Venue rules

- **Decision**: Rules are YAML files, one per venue, with `effective_from`/`effective_to`,
  eligibility keys and a `source` URL. Initial rules are in spec Appendix B.
- **Key facts**:
  - NSE CAS for F&O stocks from 2026-08-03: continuous to 15:15, CAS 15:15–15:35 IST. From
    2026-12-07: CAS 15:30–15:40, random close in the final two minutes, matching 15:40–15:45,
    post-close 15:50–16:00 IST. Non-F&O stocks continue to 15:30.
  - KRX after-market 16:00–20:00 KST from 2026-09-14; ETFs/ETNs excluded. Closing call 15:20–15:30.
  - Nextrade: after-market 15:40–20:00 KST; both venues trade after 16:00.
  - TSE: continuous to 15:25, closing auction 15:25–15:30; random close 15:29:30–15:30 from 2027-10-12.
  - LSE: 5-minute PMEs; each call, PME and market-order extension is followed by up to 30 s random period.
- **Alternatives considered**: Derive everything from `TRADING_DAY_*_TIME_EOD` (rejected: xbbg
  needs a manual Japan rule; fields carry no eligibility or effective dates).

## R5. Bootstrap schedule

- **Decision**: One bootstrap job per venue, scheduled in the venue's IANA time zone at a
  configurable local time (default 2 hours before the earliest session phase).
- **Rationale**: Fixed UTC times break at DST. On 2026-11-14, local midnight is 15:00 UTC in Tokyo,
  13:00 UTC in Sydney and 05:00 UTC in New York.

## R6. Composite and primary subscriptions

- **Decision**: Subscribe to the primary venue for session state and close. Subscribe to a
  composite only where Bloomberg offers one and it is verified (US composite). Key condition maps on
  `EVT_TRADE_LOCAL_EXCH_SOURCE_RT`.
- **Open**: No Bloomberg composite code for Korea that includes Nextrade was found. Until verified,
  subscribe to KRX and Nextrade separately and merge in the service.
- **Constraint**: Two subscriptions per instrument roughly doubles subscription count; check
  B-PIPE limits (FR-042).

## R7. Corporate actions

- **Decision**: Build the anomaly-only reference from `DVD_HIST_ALL` (ex-date, amount) and
  `EQY_SPLIT_DT` / `EQY_SPLIT_RATIO`. Do not use `CAPITAL_CHANGE_DT`. Verify `EQY_SPLIT_ADJ` before use.
- **Rationale**: `DVD_EX_DT` returns one date without an amount.

## R8. Futures settlement

- **Decision**: Candidates `PX_SETTLE_ACTUAL_RT` and `PX_SETTLE_LAST_DT_RT` (date check). Verify
  in FLDS. Settlement can arrive hours after trading ends; the late window for futures venues is
  configured separately.

## R9. Fixed income and FX

- **Decision**: Bonds use an ordered list of pricing sources (default BVAL, CBBT, BGN) subscribed as
  `ticker@source`, with a staleness TTL per asset tier. These are pricing sources, not fields;
  entitlements must be checked.
- **Decision**: FX spot is continuous Sunday 17:00 to Friday 17:00 America/New_York. Onshore
  restricted currencies (for example KRW, INR, TWD, BRL) get their own session rules.

## R10. Implementation language and runtime

- **Decision**: Python 3.12 with the official `blpapi` Python SDK.
- **Rationale**: Bloomberg ships official SDKs for C++, Java, .NET and Python, not Go. Python gives
  the fastest path for the validation phase and replay tooling. Expected scale (≤ 20,000 instruments,
  ≤ 5,000 updates/s peak) fits a single process with an asyncio event loop.
- **Alternatives considered**: Java 21 (official SDK, better hot-path throughput; preferred if peak
  load exceeds the target); Go via cgo on the C API (no official SDK, higher maintenance).

## R11. Storage and telemetry

- **Decision**: PostgreSQL 16 for the official close store, quarantine cases and audit log. Venue
  rules and field registry in version-controlled YAML. Prometheus metrics; structured JSON logs.
- **Alternatives considered**: Kafka-only event log (rejected for v1: revision queries are simpler in SQL).

## Sources

- Bloomberg B-PIPE page (Security Status 2.0): https://professional.content.cirrus.bloomberg.com/professional2023/?p=5264
- Bloomberg mktdata API description: https://apis.io/apis/bloomberg-apis/bloomberg-apis-mktdata-api/
- xbbg: https://github.com/underloam/xbbg
- jBloomberg: https://github.com/assylias/jBloomberg
- blp: https://github.com/matthewgilbert/blp
- findatapy: https://github.com/cuemacro/findatapy
- FIX MMT presentation: https://cdnws.fixtrading.org/wp-content/uploads/2023/06/FIX_Nordic_Trd_Conf_MMT_presentation_20230523-consol-23May23.pdf
- NSE CAS from 7 Dec 2026: https://www.kotakneo.com/news/regulations/nse-new-closing-auction-and-f-and-o-trading-rules-from-7-december/
- NSE CAS from 3 Aug 2026: https://www.outlookbusiness.com/markets/sebi-closing-auction-session-new-stock-market-timings-from-august-3
- KRX after-market: https://biz.heraldcorp.com/article/10859829
- Nextrade: https://nextrade.co.kr/en/main.do
- Korean market hours: https://stockhub.kr/en/guide/kr-market-hours
- JPX trading hours and random closing: https://www.jpx.co.jp/english/equities/trading/strengthening/index.html
- LSE price monitoring: https://docs.londonstockexchange.com/sites/default/files/documents/LSE_ETP_Price_Monitoring_Mechanisms_factsheet.pdf
