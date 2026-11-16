---

description: "Task list for Global Market-State-Aware Position Valuation"
---

# Tasks: Global Market-State-Aware Position Valuation

**Input**: Design documents from `/specs/001-global-market-valuation/`

**Prerequisites**: plan.md, spec.md, research.md, data-model.md, contracts/, quickstart.md

**Tests**: Included. Constitution Principle V requires replay tests written first; they MUST fail
before the implementation task that follows them.

**Organization**: Tasks are grouped by user story. `RF-nn` refers to the review finding number in
spec Appendix A.

## Format: `[ID] [P?] [Story] Description`

- **[P]**: Can run in parallel (different files, no dependencies)
- **[Story]**: User story from spec.md (US1–US6)

---

## Phase 1: Setup (Shared Infrastructure)

- [ ] T001 Create project layout per plan.md (`src/shefa/`, `tests/`, `config/`) and `pyproject.toml` for Python 3.12 with `blpapi`, `pydantic>=2`, `asyncpg`, `prometheus-client`, `structlog`
- [ ] T002 [P] Configure ruff, mypy (strict) and pytest with `hypothesis` in `pyproject.toml`
- [ ] T003 [P] Add PostgreSQL 16 service for local tests in `docker-compose.yml`
- [ ] T004 [P] Copy schemas from `specs/001-global-market-valuation/contracts/` to `config/schemas/` and add `shefa config validate` in `src/shefa/cli/config.py`

---

## Phase 2: Foundational (Blocking Prerequisites)

**⚠️ No user story work starts until this phase is complete.**

- [ ] T005 Populate `config/field_registry.yaml` from research.md R1, including `NOT_FOUND` and `OTHER_VENDOR` entries with `replaces` links (FR-001, RF-08, RF-25)
- [ ] T006 [P] Implement registry model and loader in `src/shefa/registry/model.py`; status enum "`VERIFIED`, `NAME_FOUND`, `NOT_FOUND`, `OTHER_VENDOR`"; `evidence_matrix` required when status is `VERIFIED`
- [ ] T007 [P] Implement venue rule model and loader in `src/shefa/rules/model.py`; reject overlapping `[effective_from, effective_to)` for equal eligibility; reject overlapping phases; require `source` URL; validate `iana_tz` with `zoneinfo` (FR-011)
- [ ] T008 [P] Implement Instrument and Venue models in `src/shefa/model/instrument.py` (`role`: "`PRIMARY`, `REPORTING`, `COMPOSITE`")
- [ ] T009 [P] Implement calendar loader (SETTLEMENT_CALENDAR_CODE, CALENDAR_NON_SETTLEMENT_DATES) and special sessions in `src/shefa/rules/calendar.py` (FR-014)
- [ ] T010 Implement `blpapi` session wrapper and message decoder that records `MKTDATA_EVENT_TYPE` and `MKTDATA_EVENT_SUBTYPE` in `src/shefa/ingest/session.py` and `src/shefa/ingest/decoder.py`
- [ ] T011 Implement registry start-up gate (refuse start on non-`VERIFIED` production field) in `src/shefa/registry/gate.py` (FR-002)
- [ ] T012 Build replay harness that feeds recorded or synthetic messages, including partial updates and session events, in `tests/replay/harness.py`
- [ ] T013 [P] Write replay test for partial updates (quote-only, status-only, close-only messages) in `tests/replay/test_field_cache.py` (RF-29)
- [ ] T014 Implement per-field cache with `updated_at`, `received_at`, `event_type`, `event_subtype`, `seq` in `src/shefa/ingest/cache.py` (FR-004)
- [ ] T015 [P] Write replay test for feed-health transitions (DataLoss, SlowConsumerWarning, SubscriptionTerminated, silence) in `tests/replay/test_feed_health.py` (RF-27)
- [ ] T016 Implement feed-health tracker (`state`: "`OK`, `SLOW`, `DATA_LOSS`, `TERMINATED`, `SILENT`") in `src/shefa/ingest/feed_health.py` (FR-008)
- [ ] T017 [P] Write tests for state precedence feed health ► instrument ► venue phase ► schedule in `tests/unit/test_market_state.py` (RF-27)
- [ ] T018 Implement enum-map loader (`config/enum_maps/<MIC>.yaml`, unmapped values counted as unknown) in `src/shefa/state/enum_maps.py` (FR-003, RF-26)
- [ ] T019 Implement schedule resolver (dated rule + eligibility + IANA time, zero default buffer) in `src/shefa/rules/schedule.py` (FR-012, FR-015, RF-02)
- [ ] T020 Implement market-state resolver with `deciding_tier` and `reason` in `src/shefa/state/resolver.py` (FR-017)
- [ ] T021 [P] Implement drop/quarantine/conflict counters keyed by venue and reason in `src/shefa/telemetry/metrics.py` (FR-039, RF-22)
- [ ] T022 [P] Create PostgreSQL schema for closes, quarantine cases and audit log in `src/shefa/db/migrations/001_init.sql`
- [ ] T023 Implement mark publisher validated against `mark-output.schema.json` in `src/shefa/valuation/publisher.py`

**Checkpoint**: foundation ready.

---

## Phase 3: User Story 1 - Correct official close for every position (P1) 🎯 MVP

**Goal**: a correct, dated, versioned official close for every position.

**Independent Test**: replay one trading day and compare `primary_official_close` with exchange closes.

### Tests (write first, must fail)

- [ ] T024 [P] [US1] Replay test: US closing cross at 16:00 stored from primary venue and dated today in `tests/replay/test_close_us.py` (RF-01, RF-02)
- [ ] T025 [P] [US1] Replay test: LSE close with two PMEs produces revisions and `final = true` only after late window in `tests/replay/test_close_lse_pme.py` (RF-16)
- [ ] T026 [P] [US1] Replay test: no real-time close → reference close accepted only if dated today, else hold and stale in `tests/replay/test_close_fallback.py` (RF-15)
- [ ] T027 [P] [US1] Replay test: holiday uses stored last official close with `price_type = CARRIED_FORWARD` in `tests/replay/test_close_holiday.py` (RF-20)

### Implementation

- [ ] T028 [US1] Implement OfficialCloseRecord store keyed by (instrument_id, primary_venue, session_date, price_type) with `revision` and `final`; `price_type`: "`OFFICIAL_CLOSE`, `CLOSING_AUCTION`, `SETTLEMENT`, `REFERENCE_FALLBACK`, `CARRIED_FORWARD`" in `src/shefa/closes/store.py`
- [ ] T029 [US1] Implement close latch from the primary subscription using PX_OFFICIAL_CLOSE_RT and OFFICIAL_CLOSE_AUCTION_PRICE_RT, date-checked with TRADING_DT_REALTIME, in `src/shefa/closes/latch.py` (FR-029)
- [ ] T030 [US1] Implement per-venue late window and finalisation in `src/shefa/closes/finalise.py` (FR-030)
- [ ] T031 [US1] Implement reference fallback request (accept only if dated today) in `src/shefa/closes/fallback.py` (FR-031)
- [ ] T032 [US1] Implement holiday carry-forward in `src/shefa/closes/holiday.py`; do not read PX_CLOSE_1D (FR-032)
- [ ] T033 [US1] Implement book policy selector (`FLASH_INTRADAY`, `OFFICIAL_EOD`) and `valuation_mark` in `src/shefa/valuation/book_policy.py` (FR-028, FR-033, RF-30)

**Checkpoint**: US1 independently testable.

---

## Phase 4: User Story 2 - Trustworthy intraday flash marks (P1)

**Goal**: `live_mark` moves only on price-forming trades or valid quotes in continuous trading.

**Independent Test**: replay sessions with auctions, halts, off-book prints, cancels and an ex-date;
no unflagged jump > 2%.

### Tests (write first, must fail)

- [ ] T034 [P] [US2] Replay test: trade price and size combined only from the same event; duplicate EVT_TRADE_IDENTIFIER_RT ignored in `tests/replay/test_trade_event.py` (FR-006, RF-18, RF-14)
- [ ] T035 [P] [US2] Replay test: cancel of the mark-setting trade reverts `live_mark` in `tests/replay/test_cancel.py` (RF-28)
- [ ] T036 [P] [US2] Replay test: indicative auction prices ignored; uncross print kept in `tests/replay/test_auction.py` (RF-04)
- [ ] T037 [P] [US2] Replay test: condition map applied by reporting venue (EVT_TRADE_LOCAL_EXCH_SOURCE_RT) for a composite feed in `tests/replay/test_condition_map.py` (RF-17)
- [ ] T038 [P] [US2] Replay test: quote rejected when crossed, locked, too wide or one side stale in `tests/replay/test_quote_scrub.py`
- [ ] T039 [P] [US2] Replay test: negative futures price accepted; negative equity price dropped in `tests/replay/test_negative_price.py` (RF-19)
- [ ] T040 [P] [US2] Replay test: gradual 6% move not quarantined; split ex-date first trade not quarantined in `tests/replay/test_anomaly_baseline.py` (RF-13, RF-06)
- [ ] T041 [P] [US2] Property test: quarantine never auto-promotes on timeout; exits only by 2 distinct trade IDs or fresh two-sided quotes in `tests/unit/test_quarantine.py` (RF-05, RF-31)

### Implementation

- [ ] T042 [US2] Implement event router by type/subtype (TRADE, QUOTE, STATUS, CLOSE/SETTLE, CANCEL, CORRECTION, SUMMARY) in `src/shefa/router/classify.py` (FR-005)
- [ ] T043 [US2] Implement subscription manager: primary for state and close, verified composite for intraday, KRX and Nextrade as separate subscriptions in `src/shefa/ingest/subscriptions.py` (FR-009, FR-010, RF-03, RF-17)
- [ ] T044 [P] [US2] Create condition-map loader and empty per-venue files in `src/shefa/scrub/condition_maps.py` and `config/condition_maps/`
- [ ] T045 [US2] Implement trade scrubber (size ≤ 0, indicative, venue condition map, keep uncross, asset-class price sign rule) in `src/shefa/scrub/trades.py` (FR-019, FR-020, FR-022)
- [ ] T046 [P] [US2] Implement quote scrubber (side sign for equities, crossed, locked, spread tier, side age) in `src/shefa/scrub/quotes.py` (FR-021)
- [ ] T047 [P] [US2] Implement CA reference from DVD_HIST_ALL and EQY_SPLIT_DT / EQY_SPLIT_RATIO (anomaly use only) in `src/shefa/anomaly/reference.py` (FR-023)
- [ ] T048 [US2] Implement quarantine state machine (`outcome`: "`PROMOTED_TRADES`, `PROMOTED_QUOTES`, `TIMEOUT`, `MANUAL_PROMOTE`, `SUPERSEDED`"; defaults 5%/10% tiers, 2 s trade window, 5 s quote hold, 30 s TTL) in `src/shefa/anomaly/quarantine.py` (FR-024–FR-027)
- [ ] T049 [US2] Implement live-mark engine (TRADE → MID → hold) gated by market state in `src/shefa/valuation/live_mark.py`
- [ ] T050 [US2] Implement cancel/correction reverter in `src/shefa/valuation/revert.py` (FR-007)
- [ ] T051 [US2] Implement audited manual promote command in `src/shefa/cli/promote.py` (FR-027)

**Checkpoint**: US1 and US2 independently testable.

---

## Phase 5: User Story 3 - Session rules that stay correct over time (P1)

**Goal**: dated, eligibility-keyed venue rules and a DST-safe bootstrap.

**Independent Test**: resolve schedules before and after each known rule change (quickstart Scenarios 2–3).

### Tests (write first, must fail)

- [ ] T052 [P] [US3] Test: XNSE F&O on 2026-12-04 vs 2026-12-07; non-F&O to 15:30 in `tests/unit/test_rules_xnse.py` (RF-11, RF-24)
- [ ] T053 [P] [US3] Test: XKRX ETF/ETN has no AFTER_MARKET; equity has 16:00–20:00 from 2026-09-14 in `tests/unit/test_rules_xkrx.py`
- [ ] T054 [P] [US3] Test: XTKS random close from 2027-10-12; lunch 11:30–12:30 in `tests/unit/test_rules_xtks.py`
- [ ] T055 [P] [US3] Test: bootstrap runs at the same local time across the 2026-11-01 New York DST change in `tests/unit/test_bootstrap_dst.py` (RF-07, RF-12)

### Implementation

- [ ] T056 [P] [US3] Write `config/venues/XNYS.yaml` and `config/venues/XNAS.yaml` (continuous to 16:00, no buffer)
- [ ] T057 [P] [US3] Write `config/venues/XTKS.yaml` with 2024-11-05 and 2027-10-12 rules
- [ ] T058 [P] [US3] Write `config/venues/XKRX.yaml` and the Nextrade venue file (confirm Nextrade MIC first)
- [ ] T059 [P] [US3] Write `config/venues/XNSE.yaml` with the 2026-08-03 and 2026-12-07 rules and non-F&O rule (FR-013)
- [ ] T060 [P] [US3] Write `config/venues/XLON.yaml` with closing auction, PME and random-period late window
- [ ] T061 [US3] Generate baseline files for remaining venues from TRADING_DAY_START/END_TIME_EOD and audit each against the exchange rule book in `config/venues/` (RF-09)
- [ ] T062 [P] [US3] Write `config/special_sessions.yaml` (US half-days, Diwali Muhurat, others) with sources
- [ ] T063 [US3] Implement per-venue local-time bootstrap scheduler in `src/shefa/bootstrap/scheduler.py` (FR-016)
- [ ] T064 [US3] Implement `shefa rules resolve` CLI in `src/shefa/cli/rules.py`

**Checkpoint**: US1–US3 independently testable.

---

## Phase 6: User Story 4 - Visible data quality on every mark (P2)

**Goal**: every mark carries source, age, session state and quality state.

### Tests (write first, must fail)

- [ ] T065 [P] [US4] Replay test: DataLoss → affected marks `FEED_DEGRADED`, age increases in `tests/replay/test_quality_feed.py`
- [ ] T066 [P] [US4] Replay test: calendar closed vs feed continuous → `CALENDAR_FEED_CONFLICT` and venue policy applied in `tests/replay/test_calendar_conflict.py`

### Implementation

- [ ] T067 [US4] Implement quality-state assignment (`OK`, `FROZEN_CLOSE`, `STALE`, `HALTED`, `QUARANTINED`, `QUARANTINE_TIMEOUT`, `FEED_DEGRADED`, `CALENDAR_FEED_CONFLICT`) in `src/shefa/valuation/quality.py` (FR-034, RF-10)
- [ ] T068 [US4] Implement conflict policy per venue (`HOLD_AND_ALERT` default) in `src/shefa/state/conflict.py` (FR-018)
- [ ] T069 [P] [US4] Implement periodic re-publish (≤ 60 s) so age stays visible in `src/shefa/valuation/heartbeat.py`
- [ ] T070 [P] [US4] Implement alert sink in `src/shefa/telemetry/alerts.py`

---

## Phase 7: User Story 5 - Multi-asset valuation (P2)

**Goal**: futures, FX and bonds valued with asset-specific rules.

### Tests (write first, must fail)

- [ ] T071 [P] [US5] Replay test: futures settlement stored as `SETTLEMENT` with date check in `tests/replay/test_futures_settle.py`
- [ ] T072 [P] [US5] Replay test: equity frozen at close while FX moves; value = local × qty × FX in `tests/replay/test_fx_decoupling.py`
- [ ] T073 [P] [US5] Replay test: bond stale after TTL; onshore restricted currency follows its own session in `tests/replay/test_bonds_fx.py` (RF-23)

### Implementation

- [ ] T074 [P] [US5] Implement futures sessions from FUT_TRADING_HRS and settlement latch (PX_SETTLE_ACTUAL_RT, PX_SETTLE_LAST_DT_RT after FLDS check) in `src/shefa/assets/futures.py` (FR-035)
- [ ] T075 [P] [US5] Implement bond pricing-source waterfall (BVAL, CBBT, BGN as `ticker@source`) with staleness TTL in `src/shefa/assets/bonds.py` (FR-036)
- [ ] T076 [P] [US5] Implement FX 24/5 and onshore currency session files in `src/shefa/assets/fx.py` and `config/venues/FX_*.yaml` (FR-037)
- [ ] T077 [US5] Implement position valuation with independent local price and FX in `src/shefa/valuation/position.py` (FR-038)

---

## Phase 8: User Story 6 - Safe rollout with measurable gates (P3)

**Goal**: field evidence, capture, shadow mode and acceptance gates.

- [ ] T078 [P] [US6] Implement FLDS evidence tool using `//blp/apiflds` FieldInfoRequest that writes the 8-column `evidence_matrix` into `config/field_registry.yaml` in `src/shefa/cli/evidence.py` (SC-006)
- [ ] T079 [P] [US6] Implement value capture tool for status enums, event subtypes (confirm CANCEL/CORRECTION) and condition codes per venue in `src/shefa/cli/capture.py` (RF-26)
- [ ] T080 [US6] Implement shadow mode (marks published to shadow topic only) in `src/shefa/valuation/shadow.py` (FR-040)
- [ ] T081 [US6] Implement acceptance report (SC-001–SC-007; custodian tolerance from BookPolicy) in `src/shefa/cli/report.py` (FR-041, RF-21)
- [ ] T082 [P] [US6] Implement subscription capacity check against entitlement limits in `src/shefa/cli/capacity.py` (FR-042)
- [ ] T083 [P] [US6] Write cutover and rollback runbook in `docs/runbooks/cutover.md` (region order chosen from shadow results)

---

## Phase 9: Polish & Cross-Cutting Concerns

- [ ] T084 [P] Performance test: 5,000 updates/s, tick-to-mark p99 ≤ 50 ms in `tests/perf/test_throughput.py`
- [ ] T085 [P] Write `README.md` with architecture and links to the spec
- [ ] T086 Run all quickstart.md scenarios and record results in `specs/001-global-market-valuation/quickstart-results.md`
- [ ] T087 Run `/speckit-analyze` and resolve findings

---

## Dependencies & Execution Order

- Phase 1 → Phase 2 → user stories.
- US1, US2 and US3 depend only on Phase 2 and can run in parallel. US3 rule files are needed for
  realistic US1/US2 replays of non-US venues; synthetic rules are enough for their unit tests.
- US4 depends on US2 (live marks) and Phase 2 resolver.
- US5 depends on US1 (close store) and US2 (live marks).
- US6 T078–T079 can start right after Phase 2 and SHOULD run early, because FLDS evidence and
  captured value maps gate production. T080–T081 depend on US1–US5.

## Parallel Examples

```text
US1 tests: T024, T025, T026, T027 together
US2 tests: T034–T041 together; then T044, T046, T047 together
US3 venue files: T056, T057, T058, T059, T060, T062 together
US5: T074, T075, T076 together
```

## Implementation Strategy

1. **MVP**: Phases 1–3 (US1) → correct, dated official closes for EOD/NAV.
2. **Increment 2**: US2 + US3 → trustworthy flash marks and correct venue rules.
3. **Increment 3**: US4 + US5 → quality visibility and multi-asset.
4. **Rollout**: US6 gates. Start T078/T079 in parallel with Phase 3 so that evidence is ready
   before shadow mode.
