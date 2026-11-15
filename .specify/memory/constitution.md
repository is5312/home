# Shefa Constitution

## Core Principles

### I. Verified Data Contracts (NON-NEGOTIABLE)

- Production code MUST use only Bloomberg fields, enum values and event subtypes that are listed
  in the field registry with status `VERIFIED`.
- A field is `VERIFIED` only when FLDS or `//blp/apiflds` evidence is attached for the target
  entitlement tier. Open-source usage proves that a name exists; it does not prove its meaning.
- Fields from other vendors (for example Refinitiv `MKT_STATUS_IND`, `MMT_CLASS`; Wind
  `RT_LAST_VOL`) MUST NOT appear in the codebase.
- Enum values (for example `RT_TRADING_PERIOD`, `RT_SIMP_SEC_STATUS`) MUST come from a captured
  value map, never from hard-coded guesses.

Rationale: wrong or guessed field names fail at runtime or, worse, return nulls that look valid.

### II. Effective-Dated Market Rules

- Every session, auction, eligibility and holiday rule MUST carry `effective_from` and an optional
  `effective_to` date.
- All local times MUST be resolved through the IANA time zone of the venue. Fixed UTC constants
  for schedules are forbidden.
- Rules MUST be keyed on venue plus instrument eligibility, never on venue alone.

Rationale: exchanges change hours often (TSE 2024, KRX 2026-09-14, NSE 2026-08-03 and
2026-12-07, TSE random close 2027-10-12). Undated rules expire silently.

### III. Observation Separate From Valuation

- The system MUST keep three distinct values per instrument: `live_mark` (observation),
  `primary_official_close` (versioned authoritative close or settlement) and `valuation_mark`
  (selected by book policy).
- No component may overwrite an official close with an intraday observation, or the reverse.

Rationale: an intraday flash view and an EOD/NAV close need different rules.

### IV. Fail Safe, Never Silent

- When data is missing, stale, conflicting or unconfirmed, the system MUST hold the last trusted
  value, set an explicit quality state and emit an alert or counter.
- Every dropped tick MUST increment a counter keyed by venue and drop reason.
- The system MUST NOT promote an unconfirmed price automatically. Manual promotion MUST be
  audited.

Rationale: a silent wrong mark is more expensive than a visible stale mark.

### V. Replay-Tested Behaviour

- Every venue rule, scrubber rule and state transition MUST have a test that replays recorded or
  synthetic feed messages, including partial updates, cancels and data-loss events.
- Tests MUST be written and fail before the implementation that makes them pass.
- No cutover without the shadow-mode acceptance gates in the feature plan.

Rationale: feed behaviour is complex and venue-specific; only replay tests make it repeatable.

## Data Source Constraints

- Session state and the official close come from the primary venue subscription. Intraday
  marks may come from a composite subscription.
- Condition-code rules are keyed on the reporting venue of each trade.
- Subscription counts MUST stay within the B-PIPE entitlement limits; capacity is checked before
  each rollout phase.
- Every external market fact used in a rule (hours, auction times, exclusions) MUST cite a source
  document in the rule file.

## Development Workflow & Quality Gates

- Each change to market rules is a pull request that updates the rule file, its source citation,
  its effective dates and its replay tests together.
- The FLDS evidence matrix is a pre-build gate for any new field.
- Shadow-mode telemetry (drop reasons, quarantines, calendar/feed conflicts) MUST be reviewed
  before each regional cutover.

## Governance

- This constitution supersedes other practices for this project.
- Amendments require a pull request with rationale, a version bump and an update to any affected
  spec, plan or task files.
- Versioning: MAJOR for removed or redefined principles; MINOR for new principles or sections;
  PATCH for wording.
- Every pull request review MUST check compliance with Principles I–V.

**Version**: 1.0.0 | **Ratified**: 2026-11-14 | **Last Amended**: 2026-11-14
