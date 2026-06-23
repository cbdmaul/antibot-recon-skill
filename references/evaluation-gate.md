# Phase 2 + 7 reference — The evaluation gate (ground truth or nothing)

A bypass that "works once" against a live endpoint proves nothing. Anti-bot scoring is probabilistic,
stateful, and rate-aware, so a single pass can be luck, soft-score headroom, or IP reputation you have
not spent yet. **Establish ground truth first, or you cannot establish success at all.** Stand this gate
up right after triage (Phase 2) and run every later build stage through it (it is also Phase 7).

## The negative control (the discriminator)

Prove the credential genuinely passes — a result without a control is a guess:

- **token** — send none / garbage / **tampered** (flip one byte) / yours → only *yours* should reach
  the app layer; tampered should be rejected (proves an integrity check, not just presence).
- **cookie** — replay with no cookie / a forged cookie / yours → only *yours* should succeed.
- **score** — compare a fresh-session request against your reused session; confirm the difference.

If a "should-be-blocked" case also succeeds, the gate is not validating and your result is a false
positive.

## Capture a golden trace

From an environment the vendor genuinely accepts — a real browser on real hardware (or a real attested
device for PAT), on the allowed origin — record one full successful interaction end to end: the
handshake/sensor exchange, the challenge, the credential issued, the request/response headers, the TLS
fingerprint, the device-fingerprint values the client read, the behavioral telemetry, and the protected
request that was accepted. Confirm acceptance with the negative control. This trace is your oracle.

Capture the *recipe*, not just the values. Single-use nonces mean the credential in the trace is already
spent; what you keep is the inputs (challenge, device profile, behavior) and the accepted output, so you
can regenerate and compare rather than replay a dead token.

## Gate each stage against it

As you build (Phases 4–6), confirm or refute each component before moving on:

- **Trace is live.** Re-run the golden capture against the endpoint. If it no longer passes (expired
  nonce, vendor rebuild, burned IP), the ground truth is stale — re-capture before trusting any
  comparison.
- **Field diff.** For every value your system produces — credential, fingerprint fields, header set and
  order, TLS signature, behavioral stream — diff against the golden trace. Exact match where the value
  should be stable; distributional match where it legitimately varies.
- **Localize divergence.** When your replica is rejected, the field diff points at the stage that
  differs from known-good. That is the bug. A pass/fail at the endpoint alone does not localize; the
  trace does.
- **Differential test.** Run the golden trace and your replica through the same endpoint under the same
  conditions and compare outcomes. Same input, different verdict means your divergence is the cause.
- **Discriminate acceptance.** Classify every response with the negative control as calibration:
  garbage/tampered must BLOCK, the golden trace must PASS, yours must land with the golden trace and not
  with the garbage.

## Truth discipline

- One pass is not success. Require repeated passes, and require the negative control to be discriminating
  in the same run, or you are measuring noise.
- Validate at the threshold and volume you will operate at. A request can pass while accumulating risk;
  soft scores and rate limits only surface under load.
- Acceptance drifts. Re-capture ground truth on a schedule and treat a drop in the golden trace's
  acceptance as the signal that the vendor shipped a new build, which re-triggers recon (back to
  Phase 1).
- Keep a regression suite of golden traces (across builds, device profiles, origins) and re-run it on
  every change to the bypass.

## The rule

No golden trace from a genuinely-accepted environment means no oracle, which means you can confirm a
refutation (a BLOCK) but never confirm success. Treat "no ground truth" as "not testable," and get the
trace before you build.

## Deliverables

- `store/golden/<vendor>.trace` — the recipe + accepted output, plus the negative-control cases.
- `control/` — the acceptance classifier (PASS / BLOCK / soft) calibrated by the negative control, and
  the field-diff comparator. Run continuously, not once.
