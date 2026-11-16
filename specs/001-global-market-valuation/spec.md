# Feature Specification: Global Market-State-Aware Position Valuation

**Feature Branch**: `001-global-market-valuation`

**Created**: 2026-11-14

**Status**: Draft

**Input**: User description: "Create a plan with all the fixes and corrections from the design reviews
of the metadata-driven global valuation pipeline (Americas, EMEA, APAC, plus futures, FX and fixed
income) built on Bloomberg reference and real-time data."

## Background

The original design (v1) and its revisions (v2, v3) were reviewed three times. This specification
consolidates every accepted fix and correction. Appendix A maps each review finding to the
requirement that resolves it.

## User Scenarios & Testing *(mandatory)*

### User Story 1 - Correct official close for every position (Priority: P1)

A fund accountant needs every position valued at the correct official close (or settlement price)
of its primary venue for the current session, so that EOD and NAV figures match the exchanges.

**Why this priority**: A wrong close directly misstates NAV and P&L. All other stories depend on a
trusted close.

**Independent Test**: Replay one full trading day for a representative instrument set and compare
each `primary_official_close` with the exchange-published close.

**Acceptance Scenarios**:

1. **Given** a US equity at 16:00 ET, **When** the closing cross publishes, **Then** the stored close
   equals the primary-venue official close and is dated today.
2. **Given** an LSE equity whose closing auction is extended by price monitoring, **When** the
   auction ends inside the configured late window, **Then** the final close is stored as a new
   revision and marked final.
3. **Given** a security that publishes no real-time official close, **When** the late window ends,
   **Then** the system requests the reference close, accepts it only if dated today, and otherwise
   holds the last mark with `is_stale = true` and raises an alert.
4. **Given** a local market holiday, **When** valuation runs, **Then** the mark is the stored
   official close of the last session, flagged stale, and not a "previous close" field value.

---

### User Story 2 - Trustworthy intraday flash marks (Priority: P1)

A portfolio manager needs intraday marks that move only on real, price-forming trades or valid
quotes during continuous trading, so that the flash P&L does not show false spikes.

**Why this priority**: False spikes from auctions, off-book prints, extended hours and cancelled
trades are the main failure of the current approach.

**Independent Test**: Replay recorded sessions that contain auctions, halts, off-book prints,
cancels and a corporate-action ex-date, and check that no unflagged mark jump above 2% occurs.

**Acceptance Scenarios**:

1. **Given** a venue in an auction phase, **When** indicative prices arrive, **Then** `live_mark`
   does not change and the session state shows the auction phase.
2. **Given** an auction uncross trade, **When** it arrives, **Then** it is kept as the official
   open or close print and is not removed by the scrubber.
3. **Given** a trade that is later cancelled, **When** the cancel arrives, **Then** any mark set by
   that trade is reverted to the previous trusted value.
4. **Given** a tick more than the tier threshold away from the reference, **When** neither trade
   nor quote confirmation arrives within the time limit, **Then** the mark is held, the quality
   state is `QUARANTINE_TIMEOUT`, and operations is alerted; the price is never promoted
   automatically.
5. **Given** a stock that moves 6% gradually during the day, **When** each new trade arrives,
   **Then** it is compared with the last trusted live mark and is not quarantined.
6. **Given** a split ex-date, **When** the first trade arrives, **Then** it is compared with the
   split-adjusted prior close and is not quarantined.

---

### User Story 3 - Session rules that stay correct over time (Priority: P1)

A market-data engineer needs every venue session rule to carry effective dates and local time
zones, so that announced exchange changes take effect on the right day without code changes.

**Why this priority**: NSE changes its closing auction on 2026-12-07; KRX changed its after-market on
2026-09-14; TSE adds random closing on 2027-10-12. Undated rules silently produce wrong marks.

**Independent Test**: Evaluate the session schedule for each venue on dates before and after each
known rule change and compare with the exchange publications.

**Acceptance Scenarios**:

1. **Given** an NSE F&O-eligible stock on 2026-12-04, **When** the schedule is resolved, **Then**
   continuous trading ends at 15:15 IST and the closing auction runs 15:15–15:35 IST.
2. **Given** the same stock on 2026-12-07, **When** the schedule is resolved, **Then** continuous
   trading ends at 15:30 IST and the closing auction runs 15:30–15:40 IST.
3. **Given** a KRX ETF on any date, **When** the after-market window is evaluated, **Then** the ETF
   is not eligible for KRX after-market trading.
4. **Given** the DST change in New York, **When** the daily bootstrap is scheduled, **Then** it runs
   at the same local time before the local open on both sides of the change.

---

### User Story 4 - Visible data quality on every mark (Priority: P2)

A risk manager needs every mark to carry its source, age, session state and quality state, so that
frozen, stale or quarantined positions are visible on risk screens.

**Why this priority**: Correct handling is not enough; consumers must see when a value is held.

**Independent Test**: Subscribe to the mark output during a replay that includes a feed data-loss
event and a halt, and check that each affected mark shows the right quality state.

**Acceptance Scenarios**:

1. **Given** a feed data-loss event, **When** marks are published, **Then** affected marks show
   `quality_state = FEED_DEGRADED` and their age keeps increasing.
2. **Given** a calendar that says closed while the feed says continuous, **When** this is detected,
   **Then** a `CALENDAR_FEED_CONFLICT` alert is raised and the configured policy is applied.

---

### User Story 5 - Multi-asset valuation (Priority: P2)

A fund accountant needs futures, FX and bonds valued with rules that fit each asset class, so that
the system is not limited to cash equities.

**Why this priority**: The original design claimed multi-asset coverage but had equity-only rules.

**Independent Test**: Replay one day for a listed future, an FX spot pair, an onshore restricted
currency and a government bond, and compare marks with the expected sources.

**Acceptance Scenarios**:

1. **Given** a listed future, **When** the exchange publishes the settlement price, **Then** it is
   stored as the official close of type `SETTLEMENT`.
2. **Given** a position in a foreign-listed equity after its venue closes, **When** FX moves,
   **Then** the local price stays at the official close and only the FX component of P&L changes.
3. **Given** a bond with no update within its staleness limit, **When** the mark is published,
   **Then** it is flagged stale.

---

### User Story 6 - Safe rollout with measurable gates (Priority: P3)

The platform owner needs a field-verification gate, shadow mode and numeric acceptance gates, so
that cutover happens only when the new marks are proven.

**Why this priority**: It controls risk but delivers no new valuation logic by itself.

**Independent Test**: Run shadow mode and produce the acceptance report; cutover is blocked if any
gate fails.

**Acceptance Scenarios**:

1. **Given** a field without FLDS evidence, **When** a build that uses it is proposed, **Then** the
   build gate fails.
2. **Given** two weeks of shadow results including a month-end, **When** the report is produced,
   **Then** it shows match rate against exchange closes, mismatch against custodian marks, and drop
   counts per venue and reason.

### Edge Cases

- A partial update that contains only quote fields, only status fields or only a close field.
- A trade price and a trade size that arrive in different messages.
- Two fills with the same trade identifier.
- A trade cancel or correction for a trade that set the current mark.
- A feed data-loss, slow-consumer warning or subscription termination during an auction.
- No official close published by the end of the late window.
- An auction extended more than once (LSE price monitoring extensions).
- A half-day session (US day after Thanksgiving, Christmas Eve) or a special session (Diwali Muhurat).
- Negative prices for futures, spreads or rates.
- Quote with zero, crossed or locked sides; quote with one side older than the other.
- Composite and primary feeds disagree on session state.
- A venue with no mapped status values (falls back to the schedule).
- KRX and Nextrade both trading the same stock after 16:00 KST.
- A rule change date falls inside the shadow-mode window.

## Requirements *(mandatory)*

### Functional Requirements

**Data contracts**

- **FR-001**: The system MUST read every source field through a field registry that records name,
  service, status (`VERIFIED`, `NAME_FOUND`, `NOT_FOUND`, `OTHER_VENDOR`), evidence and the meaning
  confirmed in FLDS.
- **FR-002**: The system MUST refuse to start if production configuration references a field whose
  status is not `VERIFIED`.
- **FR-003**: Status enum values MUST come from a value map captured from the live feed per venue;
  unmapped values MUST be counted and treated as unknown.

**Ingestion and event handling**

- **FR-004**: The system MUST treat every real-time update as partial and keep a per-field cache
  with its own update time and source message.
- **FR-005**: The system MUST route each message by event type and subtype (trade, quote, status,
  close or settlement, summary) before any filtering.
- **FR-006**: The system MUST evaluate trade price and trade size only when both come from the same
  trade event, and MUST use the trade identifier to detect duplicates.
- **FR-007**: The system MUST handle trade cancels and corrections by reverting or replacing any
  mark that the affected trade set.
- **FR-008**: The system MUST track feed health per subscription from data-loss, slow-consumer and
  subscription-terminated events.

**Venue and session model**

- **FR-009**: The system MUST resolve each instrument's primary venue and its reporting venues, and
  use the primary venue for session state and official close.
- **FR-010**: The system MUST allow a separate composite subscription for intraday marks, and MUST
  key condition-code rules on the reporting venue of each trade.
- **FR-011**: Session rules MUST be keyed on venue plus instrument eligibility (for example security
  type, F&O eligibility, ETF/ETN status) and MUST carry effective dates.
- **FR-012**: All local times MUST be resolved through the venue's IANA time zone.
- **FR-013**: The system MUST load the initial venue rules listed in Appendix B, each with its
  source citation.
- **FR-014**: The system MUST support half-day and special sessions from a dated special-session
  list, and MUST use the exchange holiday calendar.
- **FR-015**: The default schedule gate MUST use no open or close buffer; auction cut-offs MUST come
  only from venue rules.
- **FR-016**: Each venue's metadata bootstrap MUST run on a schedule in the venue's local time,
  before the local open.

**Market state resolution**

- **FR-017**: Market state MUST be resolved in this order: feed health, instrument restriction
  (halt, suspension), venue session phase, schedule and calendar.
- **FR-018**: When the calendar and the feed disagree, the system MUST raise
  `CALENDAR_FEED_CONFLICT` and apply the configured policy for that venue.

**Scrubbing**

- **FR-019**: The system MUST drop trades with non-positive size and indicative or theoretical
  prices, and MUST keep auction uncross prints.
- **FR-020**: The system MUST drop off-book, negotiated or late benchmark prints according to the
  condition map of the reporting venue.
- **FR-021**: The system MUST reject quotes with a non-positive side (equities only), a crossed or
  locked market, a spread above the tier limit, or one side older than the configured limit.
- **FR-022**: Non-positive prices MUST be allowed for asset classes where they are valid (futures,
  spreads, rates).

**Anomaly gate**

- **FR-023**: The first trade of a session MUST be compared with an anomaly-only reference price
  adjusted for splits and dividends on the ex-date; this adjustment MUST NOT change booked P&L.
- **FR-024**: Later trades MUST be compared with the last trusted `live_mark`.
- **FR-025**: Thresholds MUST be configurable by liquidity tier.
- **FR-026**: A quarantined price MUST be promoted only by two distinct trade events inside the
  confirmation band and window, or by fresh two-sided quotes that contain the price for the
  configured duration.
- **FR-027**: When the quarantine time limit expires, the system MUST hold the last trusted mark,
  set `QUARANTINE_TIMEOUT` and alert; it MUST provide an audited manual promote action.

**Marks and close store**

- **FR-028**: The system MUST keep `live_mark`, `primary_official_close` and `valuation_mark`
  separately for each instrument.
- **FR-029**: Official closes and settlements MUST be stored with instrument, primary venue,
  session date, price type, revision number and finality.
- **FR-030**: The system MUST accept official close revisions until the venue's late window ends.
- **FR-031**: After the late window, if no close dated today exists, the system MUST request the
  reference close and accept it only if dated today; otherwise it MUST hold and flag stale.
- **FR-032**: On a venue holiday, the system MUST use the stored official close of the last session.
- **FR-033**: `valuation_mark` MUST be selected by book policy (`FLASH_INTRADAY` or `OFFICIAL_EOD`).
- **FR-034**: Every published mark MUST include price, source, session state, quality state,
  `is_stale` and age in seconds.

**Multi-asset**

- **FR-035**: Futures MUST use futures trading hours and store the exchange settlement price.
- **FR-036**: Bonds MUST bypass venue session gates and use an ordered list of pricing sources with
  a staleness limit.
- **FR-037**: FX MUST be marked continuously from Sunday 17:00 to Friday 17:00 New York time, except
  onshore restricted currencies, which MUST have their own sessions.
- **FR-038**: Position value MUST be local price × quantity × FX rate, with local price and FX rate
  resolved independently.

**Telemetry and rollout**

- **FR-039**: The system MUST count every drop, quarantine, timeout and conflict by venue and reason.
- **FR-040**: The system MUST run in shadow mode without affecting live P&L.
- **FR-041**: The system MUST produce an acceptance report for the gates in the Success Criteria.
- **FR-042**: The system MUST check subscription capacity against entitlement limits before each
  rollout phase.

### Key Entities

- **Field Registry Entry**: a source field, its service, verification status, evidence and confirmed meaning.
- **Venue**: a trading venue with MIC, IANA time zone, calendar and role (primary, reporting, composite).
- **Session Rule**: a dated, eligibility-keyed set of session phases and windows for a venue.
- **Instrument**: an instrument with identifiers, asset class, primary venue and eligibility flags.
- **Field Cache Entry**: the last value of one field for one subscription, with update time and source event.
- **Market State**: the resolved state of an instrument with the deciding tier and reason.
- **Quarantine Case**: a held candidate price with trigger, confirmation evidence and outcome.
- **Official Close Record**: a versioned close or settlement with revision and finality.
- **Mark**: a published value (`live_mark` or `valuation_mark`) with source, age and quality state.
- **Book Policy**: the rule that selects the valuation mark for a book.

## Success Criteria *(mandatory)*

### Measurable Outcomes

- **SC-001**: For at least 99.9% of liquid positions, the stored official close equals the exchange
  official close exactly (0 bps), measured 30 minutes after the close over the shadow period.
- **SC-002**: The difference from custodian EOD marks is within a separate, documented tolerance
  for at least 99.5% of positions; every larger difference has a recorded cause.
- **SC-003**: Zero unflagged intraday mark jumps above 2% outside continuous trading during the
  shadow period.
- **SC-004**: 100% of marks published during feed degradation, halts or quarantines carry a
  non-normal quality state.
- **SC-005**: Session schedules match exchange publications for 100% of venues on test dates before
  and after each known rule change.
- **SC-006**: No production field lacks FLDS evidence at cutover.
- **SC-007**: Shadow mode runs at least 10 trading days, including one month-end, before any cutover.

## Assumptions

- The source system is Bloomberg B-PIPE (real-time) and the Bloomberg reference data service,
  with entitlements for all target venues.
- The custodian EOD tolerance for SC-002 is set by the fund accountants before shadow mode starts;
  a default of 5 bps is used until then.
- The book policy default is `OFFICIAL_EOD` for NAV books and `FLASH_INTRADAY` for PM views.
- Cutover order is decided from the shadow-mode results, not fixed in advance.
- The manual promote action is used by the market-data operations team.
- Implementation language and runtime are decided in the plan.

## Appendix A - Review Findings Traceability

| # | Finding | Source review | Resolved by |
|---|---|---|---|
| 1 | Close fallback used prior-day close (`PX_CLOSE_1D`) | Review 1 | FR-031, FR-032 |
| 2 | Fixed 5-minute closing buffer froze US marks 15:55–16:00 | Review 1 | FR-015 |
| 3 | "Never composite" lost most US volume | Review 1 | FR-009, FR-010 |
| 4 | Scrubber removed real auction uncross prints | Review 1 | FR-019 |
| 5 | Spike quarantine had no exit | Review 1 | FR-026, FR-027 |
| 6 | Corporate actions missing | Review 1 | FR-023 |
| 7 | 00:00 UTC bootstrap collided with the Tokyo open | Review 1 | FR-016 |
| 8 | Invented or other-vendor field names | Review 1, 2 | FR-001–FR-003 |
| 9 | Futures, bonds, half-days not designed | Review 1 | FR-014, FR-035, FR-036 |
| 10 | No staleness flag; no acceptance criteria | Review 1 | FR-034, FR-041, SC-001–SC-007 |
| 11 | NSE override expires on 2026-12-07 | Review 2 | FR-011, FR-013 |
| 12 | Bootstrap at fixed UTC times, wrong after DST | Review 2 | FR-012, FR-016 |
| 13 | Spike baseline stayed at prior close all day | Review 2 | FR-024 |
| 14 | Quarantine exits undefined; duplicate fills | Review 2 | FR-006, FR-026, FR-027 |
| 15 | No close fallback in v2 | Review 2 | FR-031 |
| 16 | Late window was a constant | Review 2 | FR-030 |
| 17 | Status read from composite; condition map keyed on primary MIC | Review 2 | FR-009, FR-010 |
| 18 | `LAST_PRICE` combined with size from another event | Review 2 | FR-006 |
| 19 | Negative prices rejected | Review 2 | FR-022 |
| 20 | Holiday marks used a field with unknown meaning | Review 2 | FR-032 |
| 21 | Acceptance gate compared exchange and custodian at 0 bps | Review 2 | SC-001, SC-002 |
| 22 | Drop-reason telemetry removed in v2 | Review 2 | FR-039 |
| 23 | Onshore restricted currencies traded as 24/5 | Review 2 | FR-037 |
| 24 | NSE timings out of date in v3 | Review 3 | FR-013 |
| 25 | `EVT_TRADE_ID_RT` does not exist | Review 3 | FR-001 |
| 26 | Status enum values were guessed | Review 3 | FR-003 |
| 27 | Feed health ranked below venue phase | Review 3 | FR-017 |
| 28 | No cancel or correction path | Review 3 | FR-007 |
| 29 | Partial updates processed as full records | Review 3 | FR-004, FR-005 |
| 30 | Observation and valuation in one variable | Review 3 | FR-028–FR-033 |
| 31 | Unconfirmed price auto-promoted on timeout | Review 3 | FR-027 |

## Appendix B - Initial Venue Rules (verified 2026-11-14)

| Venue | Eligibility | Effective | Rule | Source |
|---|---|---|---|---|
| XNYS / XNAS | Equity | current | Continuous to 16:00 ET; closing cross at 16:00; no buffer | Exchange rule books |
| XTKS | Equity | 2024-11-05 | 09:00–11:30, 12:30–15:25 continuous; closing auction 15:25–15:30 JST | JPX |
| XTKS | Equity | 2027-10-12 | Closing auction uncross at a random time 15:29:30–15:30:00 | JPX |
| XKRX | Equity (not ETF/ETN) | 2026-09-14 | Continuous to 15:20; closing call 15:20–15:30; after-market 16:00–20:00 KST | KRX, Korea Herald |
| XKRX | ETF / ETN | 2026-09-14 | Not eligible for after-market | Korea Herald |
| Nextrade | Eligible equities | current | Pre 08:00–08:50; main 09:00:30–15:20; after 15:40–20:00 KST | Nextrade |
| XNSE | F&O-eligible | 2026-08-03 to 2026-12-06 | Continuous to 15:15; CAS 15:15–15:35 IST | NSE / broker notices |
| XNSE | F&O-eligible | 2026-12-07 | Continuous to 15:30; CAS 15:30–15:40; matching to 15:45; post-close 15:50–16:00 IST | Kotak Neo summary of NSE circular |
| XNSE | Non-F&O | current | Continuous to 15:30 IST | NSE / broker notices |
| XLON | Equity | current | Closing auction with 5-minute price monitoring extensions, each followed by a random period of up to 30 s; late window per venue | LSE |
