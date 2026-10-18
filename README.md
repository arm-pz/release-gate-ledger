# release-gate-ledger

One answer to "can we deploy?" — where each check writes its own row, and the row is rejected the
moment it is wrong.

`web-preflight`, `pre-ship-security-red-team` and `pre-ship-performance-audit` answer different
questions about the same site. This skill sequences them and turns their three verdicts into one
release posture with a script, not a judgement call. **Three verdicts stay separate**: the posture is
a gate, never an average, and a preflight `PASS` never implies the application survived penetration
testing.

## Install

```
npx skills add arm-pz/release-gate-ledger --skill release-gate-ledger
```

Then `/release-gate-ledger https://example.com --app http://127.0.0.1:3000 --project .`

## What makes this fork different

Three forks share the same checker, ledger format and tests. Only the row-writing mechanism differs.

| Fork | Who writes the ledger |
|---|---|
| chair | the gate copies each lane's table verbatim afterwards |
| **ledger** (this repo) | each lane appends its own rows through a validating recorder |
| validator | runs nothing; ingests reports that already exist |

The recorder, `scripts/gate_record.mjs`, enforces the enums at write time:

```
REJECTED: --severity must be CRITICAL|HIGH|MEDIUM|LOW (map perf P0-P3 first)
REJECTED: --accepted-by is required for accepted-risk
REJECTED: --touches must be data|money|role|none|unknown — "none" is a claim, not a default
```

A bad value surfaces against the lane that produced it, instead of arriving at the checker as a
mystery. That is the real gain today, and it holds even without the patches below.

## The patches are unapplied, on purpose

`patches/` contains three additive diffs that teach the published skills to write their own rows when
a gate passes `--gate-ledger`. They are **not applied**: `web-preflight`,
`pre-ship-security-red-team` and `pre-ship-performance-audit` are live, installed skills, and editing
them is a separate decision with its own publication path (repo, marketplace, site page). Each patch
was dry-run verified with `git apply --check` against its repo; regenerate them if a repo has moved.

Until they merge, this fork degrades honestly: the chair calls the recorder on each lane's behalf
immediately after that lane reports. Same validation, one extra hop.

## Layout

| Path | What |
|---|---|
| `SKILL.md` | profile, run lanes, record, compute, deliver |
| `scripts/check_gate.mjs` | the posture rule as code |
| `scripts/gate_record.mjs` | the validating ledger writer |
| `references/ledger.md` | the shared two-file ledger contract |
| `patches/*.patch` | the three upstream handoffs, unapplied |
| `tests/posture.mjs` | 20 fixtures; 7 must come back `ready` |

## Posture rule

HOLD if: a lane is `BLOCK`, `INCONCLUSIVE`, `HOLD`, `fail` or `untested` (R1); CRITICAL/HIGH is
`open` or `fixed-unproven` (R2); the app lane lacks a `check_results.mjs` exit 0 (R3); a finding
touching another user's data, money or role is not `fixed-verified` (R4); a required lane has no row
(R5); a lane sits at a stale sha (R6); `accepted-risk` has no attribution (R7); CRITICAL/HIGH leaves
`touches=unknown` (R8). Exit `0` ready, `1` HOLD, `2` contract break.

MIT © 2026 Abdul Rahman P Z
