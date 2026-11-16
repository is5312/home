# Quickstart: Validation Scenarios

**Spec**: [spec.md](./spec.md) | **Contracts**: [contracts/](./contracts/) | **Data model**: [data-model.md](./data-model.md)

## Prerequisites

- Python 3.12, PostgreSQL 16, the `blpapi` SDK
- B-PIPE credentials for live capture (replay scenarios need none)

## Setup

```bash
python -m venv .venv && . .venv/bin/activate
pip install -e ".[dev]"
createdb shefa_test
shefa config validate config/          # schemas in contracts/
```

## Scenario 1 - Field gate (FR-001, FR-002, SC-006)

```bash
shefa registry check --profile production
```

Expected: exit code ≠ 0 and a list of every referenced field whose status is not `VERIFIED`.
Before Phase 0 evidence is complete, this list is expected to be non-empty.

## Scenario 2 - Dated venue rules (FR-011–FR-013, SC-005)

```bash
shefa rules resolve XNSE --eligibility fo_eligible=true --date 2026-12-04
shefa rules resolve XNSE --eligibility fo_eligible=true --date 2026-12-07
shefa rules resolve XKRX --eligibility etp=true --date 2026-11-16
shefa rules resolve XTKS --eligibility security_typ=Equity --date 2027-10-12
```

Expected: CONTINUOUS ends 15:15 then 15:30 IST; KRX ETP has no AFTER_MARKET phase; TSE closing
auction has `random_end_s: 30`.

## Scenario 3 - Bootstrap schedule across DST (FR-016)

```bash
shefa bootstrap schedule --venue XNYS --from 2026-10-30 --to 2026-11-03
```

Expected: same local time each day; UTC time shifts by one hour on 2026-11-02.

## Scenario 4 - Replay suite (US 1, 2, 4)

```bash
pytest tests/replay -k "partial_update or cancel or auction_uncross or quarantine or data_loss or holiday"
```

Expected: all pass. Fixtures include a split ex-date, a gradual 6% move, an LSE double PME, a
missing official close and a calendar/feed conflict.

## Scenario 5 - Shadow acceptance report (FR-041, SC-001–SC-007)

```bash
shefa report acceptance --from <start> --to <end> --out reports/acceptance.html
```

Expected: report shows exact-match rate vs exchange closes, custodian differences vs tolerance,
unflagged jumps > 2%, quality-state coverage and drop counts by venue and reason. Cutover is
blocked if any gate fails.
