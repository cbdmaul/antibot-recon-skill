---
name: antibot-recon
description: >-
  Reverse-engineer and assess any client-side anti-bot / bot-detection platform, then determine
  whether its "human" credential can be produced programmatically. Covers the full landscape, not
  one vendor: token systems (TrustSig X-TrustSig-Response, Kasada x-kpsdk, hCaptcha/Turnstile),
  cookie systems (DataDome `datadome`, Akamai `_abck`/`bm_sz`, Imperva `reese84`, PerimeterX/HUMAN
  `_px`), proof-of-work challenges, passive behavioral scoring, and attestation tokens (Private
  Access Tokens / Privacy Pass). Use when triaging which anti-bot vendor/shape a site uses,
  deobfuscating an anti-bot SDK (string-array OR VM/control-flow obfuscation), cracking a WASM
  challenge, recovering a sensor cookie/token, identifying a PAT/attestation scheme, reproducing or
  rendering a hard fingerprint surface (canvas, WebGL/shaders, audio) from a real browser, judging the
  attacker capability needed to automate a protected site, or designing and building an end-to-end
  bypass system (recon -> profile -> tiered runner -> transport/scale) gated on a ground-truth golden
  trace. Reach for this whenever a site is protected by an invisible bot wall, a CAPTCHA-less SDK, a
  WASM/JS challenge, or a fingerprint/attestation check, even if the user does not name the vendor.
---

# Anti-bot recon & bypass assessment

A repeatable, vendor-agnostic methodology, run as a **phased pipeline**. Anti-bot SDKs differ a lot in
surface detail — token vs cookie, WASM vs pure-JS VM, active challenge vs passive scoring vs hardware
attestation — but they share one loop, and each phase below produces a **named artifact** that fills out
a single project layout. Run the phases in order; each has a gate that confirms or refutes before you
spend on the next. The depth for each phase lives in `references/`; this file is the spine.

For authorized security research, competitive analysis, and defense of systems you own or may test.

## The universal model

Every client-side anti-bot system runs the same four-step loop. Only step 3, the *credential*, varies,
and identifying that variation is the whole job of triage:

1. **Collect signals** — device/browser fingerprint (canvas/WebGL/audio/fonts/navigator/screen),
   behavior (pointer/keystroke/timing), automation tells (`webdriver`, headless, software renderer).
2. **Prove to the server** — a sensor/handshake/collect request, sometimes gated on a challenge or PoW.
3. **Receive a credential** — a **token** (header/body), a **cookie**, a passive **risk score**, or an
   **attestation token** minted by an issuer (PAT / Privacy Pass; `WWW-Authenticate: PrivateToken`).
4. **Attach the credential** to protected requests, which the edge validates and/or re-scores.

## Core principles

- **Triage first; defeat only what the target checks.** A site protected by a TLS fingerprint alone
  needs an impersonation client, not a browser farm. Do not pay for an axis the target does not use.
- **Ground truth or nothing.** No golden trace from a genuinely-accepted environment means no oracle:
  you can confirm a BLOCK but never confirm success. Treat "no ground truth" as "not testable."
- **Cost scales with the credential tier, not the orthogonal axes.** Hold each target at the lowest
  credential tier its profile allows.

## The artifact tree (what the phases fill out)

```
antibot-defeat/
  recon/{triage.json, deobf/, wasm/, surfaces.json}   # Phases 1, 3
  targets/<vendor>/profile.json                        # Phase 4 — the contract
  core/{orchestrator/, runners/{http,emulated,browser}, fingerprint/, behavior/, challenge/,
        transport/, scale/}                            # Phases 5, 6
  pool/                                                # Phase 6
  control/                                             # Phases 2, 7 — the evaluation gate
  store/                                               # golden traces, profiles, credential cache
```

Full annotated layout, the credential ladder (C0–C4), and the two orthogonal axes (transport, scale)
are in `references/architecture.md`.

## The pipeline

Each phase: a goal, parallelizable subagents where the work fans out, a deliverable artifact, and a gate.

### Phase 1 — Triage → `recon/triage.json`
Name the credential and its shape before writing any analysis code. The recon modalities are
independent, so **fan out a subagent team in one batch** — network-diff, storage-inspector,
loader-classifier, block-prober, attestation-probe — and merge their findings.
**Gate:** you can name the credential + transport and the attestation case. If PAT is hardware-backed,
stop — read `references/attestation.md`; the requirement becomes real hardware, not a browser.
Depth: `references/triage.md`, `references/attestation.md`.

### Phase 2 — Establish the evaluation gate → `store/golden/`, `control/`
Get ground truth before building. Capture a **golden trace** (the recipe + accepted output) from a
genuinely-accepted environment, and build the acceptance classifier calibrated by the negative control.
This capture is already a working **pilot**: if a real browser per request is affordable, skip to the
`browser/` runner in Phase 5 — Phases 3–4 are only for a lighter/offline/scaled replica.
**Gate:** the golden trace passes live AND the negative control discriminates (garbage/tampered BLOCK).
No golden trace = do not proceed. Depth: `references/evaluation-gate.md`.

### Phase 3 — Decompose & inventory (only for an offline/scaled replica) → `recon/deobf/`, `recon/wasm/`, `recon/surfaces.json`
Learn what is checked and whether each input is **static or dynamic** — the tag that sets the credential
tier. Independent branches, so **fan out**: deobfuscator, wasm-analyst, one per-surface analyst each for
canvas/WebGL/audio/fonts, and a behavioral-analyst.
**Gate:** every checked surface/behavior is tagged static|dynamic with evidence diffed against the
golden trace. Depth: `references/decompose.md`, `references/surfaces.md`.

### Phase 4 — Write the profile → `targets/<vendor>/profile.json`
Collapse recon into the single contract the runtime consumes: credential tier (C0–C4), orthogonal axes
present, attestation case, per-surface static|dynamic, endpoints, header set/order.
**Gate:** the profile predicts the golden trace — replaying its recipe reproduces the accepted
credential. Depth: `references/architecture.md`.

### Phase 5 — Build the credential runner to the profile's tier → `core/`
Produce the accepted credential at the lowest tier the profile allows. **Fan out one subagent per
component**, each graded against the golden trace before merge: `fingerprint/`, `behavior/` (only if it
gates), `challenge/`, and the tier runner (`http` | `emulated` | `browser`); wire them under
`orchestrator/`.
**Gate:** each component field-matches the golden trace; the assembled runner's credential PASSES via
the control and garbage BLOCKs. Depth: `references/architecture.md`, `references/surfaces.md`.

### Phase 6 — Add the orthogonal axes if present → `core/transport/`, `core/scale/`, `pool/`
Match transport fidelity **only if** the edge fingerprints TLS/JA3/headers (free inside a real browser;
an impersonation client otherwise). Add scale **only for volume** (proxy + device-profile rotation,
warm-browser `pool/`).
**Gate:** transport signature field-matches the golden trace; at volume, acceptance holds and identities
do not cluster. Depth: `references/architecture.md`.

### Phase 7 — Verify & operate → continuous `control/`
Keep it honest. Run the full system through the gate **at operating volume**, keep a golden-trace
regression suite, and monitor acceptance.
**Gate:** repeated passes at operating volume with a discriminating control. A drop in golden-trace
acceptance re-triggers Phase 1 for that target. Depth: `references/evaluation-gate.md`.

## What you'll conclude

For most platforms you can produce a working credential by **piloting the genuine client** — piloting
needs no understanding, so the obfuscation and the cryptographic story add little durable cost. The real
cost is conventional anti-bot economics: a real engine, a plausible fingerprint, an allowed origin, and
(to scale) rotated device profiles plus proxies. The exception is hardware-backed attestation (PAT),
where the bar is genuine attested devices. State the result as an **attacker-capability tier** — pure
replay, offline replica, single real browser, rotated browser farm, real-device farm — and what each
costs. That is what a defender needs to model, and it beats a "bypassed: yes/no" verdict.

## Reference index

- `references/triage.md` — Phase 1: triage questions, recon subagent team, recognition guide.
- `references/attestation.md` — PAT / Privacy Pass identification and hardware-attestation reasoning.
- `references/decompose.md` — Phase 3: structural JS deobfuscation, WASM extraction/decompile.
- `references/surfaces.md` — Phase 3: hard device surfaces + behavioral collection; static|dynamic call.
- `references/architecture.md` — Phases 4–6: piloting, credential ladder, orthogonal axes, code layout.
- `references/evaluation-gate.md` — Phases 2 & 7: negative control + golden trace + truth discipline.
