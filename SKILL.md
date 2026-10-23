---
name: release-gate-ledger
description: >-
  Runs the three pre-launch passes in order and has each lane write its own rows into one
  release-gate ledger through a validating recorder, instead of a chair copying tables after
  the fact. Produces three separate verdicts and one release posture (HOLD or ready) computed
  by script. Use when the three lane skills carry the ledger patches below, or when someone
  asks for the ledger variant of the release gate, for a site or client app you own or are
  authorized to test; a plain go/no-go belongs to release-gate-chair. Ships the three upstream patches that teach web-preflight,
  pre-ship-security-red-team and pre-ship-performance-audit to append their own rows — unapplied,
  because those are published skills. A pass in one lane never covers another. Not for a
  single-lane check, not for the default go/no-go (use release-gate-chair), not for
  penetration testing of production, not for performing the deploy.
argument-hint: <production URL> [--app <isolated/local URL>] [--project <path>]
version: 0.1.0
---

# Release gate ledger

Same lanes, same ledger, same posture rule as the chair — one difference: **rows are written
through `scripts/gate_record.mjs`, which rejects a bad enum at the moment it is written**, by the
agent that has the context to fix it, rather than at the end by the one that does not.

`<skill-dir>` is this skill's directory.

## Inputs

| Input | Needed for | If absent |
|---|---|---|
| Production URL | exposure lane | Ask. A gate with no target is a template. |
| Isolated or local app URL | app-tests lane | Record the lane `untested`, or de-scope it with the user's reason. Never red-team production. |
| Project path | performance lane | Perf lane runs measurement-only and says so |

## 1. Profile the target

`static-site` (no auth, no datastore), `web-app` (auth, roles, quota, datastore), `api-or-worker`
(no page to measure in a browser), `preview-only` (not yet deployed). Write the profile into the run
directory. It decides which rows are `not-applicable`, and the profile that justified each one goes
in `notes`. A lane is never quietly shortened.

## 2. Open the run directory

```
node "<skill-dir>/scripts/gate_record.mjs" lane .release-gate/<run-id> --lane exposure --target <host> --verdict untested --sha <git-head>
```

The recorder creates the file with the correct header on first write. Add `.release-gate/` to
`.gitignore` before the first finding row.

## 3. Run the lanes, each recording its own rows

Pass `--gate-ledger .release-gate/<run-id>` to every lane. Order stays: exposure → app-tests → perf.
A lane's pass never authorises the next one to start early.

**Exposure** — `web-preflight`. Bounded GET/HEAD, so a live host is in scope. Records its host
verdict verbatim (`PASS`, `PASS WITH FIXES`, `BLOCK`, `INCONCLUSIVE`) and one row per `PREFLIGHT-<nn>`.

**App-tests** — `pre-ship-security-red-team`. Local or isolated only. Records `ready` or `HOLD` as
its own ship posture, plus `--checker <exit>` from:

```
node "<red-team-skill-dir>/scripts/check_results.mjs" <run-dir> --require-fixes
```

**Perf** — `pre-ship-performance-audit`. Records `measured` or `not-measured`, and one row per
`PERF-<nn>` with P0-P3 mapped to CRITICAL/HIGH/MEDIUM/LOW.

Each lane fills `touches` (`data` | `money` | `role` | `none` | `unknown`) from what the finding
says. The recorder will not accept a value outside the ladder, and `unknown` on a CRITICAL or HIGH
row holds the gate — which makes the lazy answer cost a HOLD instead of buying a false pass.

**Honest state of this fork.** The three published skills do not yet know about `--gate-ledger`; the
patches in `patches/` teach them and are deliberately **unapplied**, because they are live,
installed skills and applying them is a separate approved publication. Until they merge, the chair
calls the recorder on each lane's behalf immediately after that lane reports — same validation, same
rows, one extra hop. That behaviour is this fork's floor, not its ceiling: the value here today is
write-time validation, and the patches only change who types it.

## 4. Compute the posture

```
node "<skill-dir>/scripts/check_gate.mjs" .release-gate/<run-id> --sha <current-head-sha>
```

Exit `0` ready, `1` HOLD, `2` contract break. Report the rule codes that fired. `ready` means ready
for the coverage stated, and the coverage line is part of the verdict, not a footnote. Urgency does
not change posture; deploying a `HOLD` is the user's call, made in the open.

## 5. Deliver, and re-check after deploy

Write the register to `docs/release/<YYYY-MM-DD>-<target>-gate.md`, commit if the user wants it in
history, reuse `finding-id`s on reruns. Then offer the post-deploy re-check: one `exposure` row for
the live host in a new run directory. It can move `ready` to `HOLD`; it cannot move `HOLD` to
`ready`, because it adds no coverage the pre-deploy pass lacked.

## Does not

- Apply the upstream patches, deploy anything, or widen any lane's scope.
- Merge scores or average severities; three lanes keep three verdicts.
- Turn an unrun lane into a pass, red-team production, or reproduce a secret in the register.

Read `caffeinity-stack` before reporting Caffeinity work. W3 in `caffeinity-routing` is the sequence
this skill implements.

## Verify before trusting

```
node "<skill-dir>/tests/posture.mjs"
node "<skill-dir>/scripts/gate_record.mjs" lane /tmp/x --lane exposure --verdict MAYBE
```

The second must be **rejected** — a recorder that accepts anything is a typewriter, not a control.
