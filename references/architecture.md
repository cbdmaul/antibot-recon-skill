# Phases 4–6 reference — Profile, complexity ladder, and architecture

The recon output is a per-target *profile*. It drives one decision — how much real browser you must
spend per request — and the runtime is built around that decision, not around a single technique. The
expensive axis (generating an accepted credential) is separate from two cheaper, orthogonal axes
(transport fidelity and scale). Conflating them is how you end up building a browser farm to beat a site
whose only defense is a TLS fingerprint.

## Piloting the genuine client (the default, cheapest credential path)

For any shape, the lowest-cost path is to run the unmodified vendor client in a real browser on the
protected origin, let it mint the credential, and read it off. This needs **zero** understanding of the
obfuscation — you run their code, not a replica.

- **Token systems** — drive the SDK API (or just let the page run), then capture the token from the
  JSON/header/JS global. (TrustSig: `fetch+eval` core.js → `init` → `scan` → `getResponse`.)
- **Cookie systems** — load the page, let the sensor run, read the credential cookie from the jar
  (`document.cookie`, or CDP `Network.getAllCookies` for HttpOnly), then replay protected requests
  carrying that cookie.
- **Behavioral / score systems** — no artifact to copy; keep generating plausible telemetry from a
  realistic session and reuse it.

Keep the credential-producing step and the protected request in the same browser context when you can,
so TLS/JA3, `Origin`, and cookies are the engine's and need no impersonation. If piloting per request is
affordable, this *is* your production runner (the `browser/` runner) — the rest of this reference is for
when you need a lighter or scaled replica.

## Credential ladder (the main axis)

Each tier is a superset of the ones below. Build to the highest tier recon found, no further.

| Tier | Credential defense in recon | What the system must do | Component |
|---|---|---|---|
| C0 | none, or a static credential | send the request | HTTP client |
| C1 | credential derived by simple JS, no hardware FP | run the derivation offline | JS-sandbox runner (`mini-racer`/node `vm`) |
| C2 | device fingerprint required, **static** challenge input | inject a captured real profile | profile store + emulated runner |
| C3 | **dynamic** per-session challenge, real-GPU FP | render the hard surface live | real-browser oracle (hybrid) or full pilot |
| C4 | behavioral scoring or automation detection that **gates** the credential | supply plausible behavior, hide hooks | behavior engine + stealthed real browser |

Attestation tokens (PAT / Privacy Pass) are not a C-tier — see `attestation.md`. A hardware-backed
issuer needs a real attested device, not a browser; an optional or software-grade one drops back to a
fallback or to the issuer's own check.

## Two orthogonal axes — add only if recon shows them

Not higher tiers. Usually cheap, and either one can be the *entire* defense.

- **Transport fidelity** — TLS/JA3/JA4, HTTP/2 settings, header order. If the edge fingerprints the
  connection, a plain HTTP client is rejected before any JS runs. Match it with a browser-TLS
  impersonation client (`curl-impersonate`, `utls`, `tls-client`) and correct header order. Cheap,
  independent of credential complexity, and a real browser supplies it for free. **If transport
  fingerprinting is the only defense (no JS/cookie/token challenge), that is the whole job — use an
  impersonation client, not a browser.**
- **Scale and identity** — origin binding, single-use nonces, rate-limit/clustering. Origin binding is
  cheap: send the allowed `Origin`/host and the right cookies. Single-use nonces mean one credential per
  request, never replayed. Rate-limit and fingerprint-clustering matter only when you need *volume*;
  then add proxy rotation and a pool of device profiles so authentic identities do not cluster on one
  host. None of this changes how a single credential is generated.

The triage principle holds: defeat only what the target checks. JA3 alone → a TLS-impersonation client,
not a browser. A WASM challenge with no transport check → a real browser but no impersonation.

## Decision flow (per request)

```
 recon profile
      │
      ▼
 JS / cookie / token credential challenge?
      │ no ──► transport fingerprinted? ──yes──► TLS-impersonation HTTP client   (done)
      │                                  ──no───► plain HTTP client              (done)
      │ yes
      ▼
 pick the highest credential tier recon found:
   C1 JS-sandbox derive · C2 emulated + captured FP · C3 real-browser render/pilot · C4 + behavior/stealth
      │
      ▼
 transport fingerprinted?  ──yes──► add an impersonation client   (free inside a real browser)
      │
      ▼
 need volume?  ──yes──► proxy + device-profile rotation; keep credential step and protected request in one context
      │
      ▼
 control harness verifies acceptance ── BLOCK ──► escalate credential tier / fix transport / rotate / re-triage
```

## Code layout (the artifact tree the phases fill out)

```
antibot-defeat/
  recon/                        # Phase 1+3, per-target, one-time; output is the profile below
    triage.json                 # Phase 1: credential, transport, endpoints, vendor shape, PAT case
    deobf/                      # Phase 3: structural JS passes (string-array; VM/control-flow)
    wasm/                       # Phase 3: extract tables by execution; decompile; probe inventory
    surfaces.json               # Phase 3: device surfaces + behaviors checked, each static|dynamic
  targets/<vendor>/profile.json # Phase 4: the contract the runtime consumes
  core/
    orchestrator/               # Phase 5: reads profile, picks the credential tier + which axes apply
    runners/
      http/                     # Phase 5: C0-C1: replay + JS-sandbox credential derivation
      emulated/                 # Phase 5: C2: offline client with injected captured fingerprint
      browser/                  # Phase 5: C3-C4: real-browser pilot and rendering oracle
    fingerprint/                # Phase 5: capture, consistency-check, and serve device profiles
    behavior/                   # Phase 5: mouse/key/touch synthesis and human-session replay
    challenge/                  # Phase 5: parse the server challenge; route hard surfaces to the oracle
    transport/                  # Phase 6 ORTHOGONAL: TLS-impersonation + header-order (skip when piloting)
    scale/                      # Phase 6 ORTHOGONAL: proxy + device-profile rotation (only for volume)
  pool/                         # Phase 6: warm real(-GPU) browsers; bind profile + proxy per session
  control/                      # Phase 2+7: the evaluation gate (golden-trace diff + negative control)
  store/                        # captured profiles, golden traces, recorded sessions, credential cache
```

## `profile.json` — the Phase 4 contract

```json
{
  "credential_tier": "C0|C1|C2|C3|C4",
  "credential": { "kind": "...", "name": "...", "transport": "..." },
  "attestation": "none|optional|software|hardware",
  "axes": { "transport": true, "scale": false },
  "surfaces": [ { "name": "webgl", "input": "dynamic", "gates": true } ],
  "behavior_gates": false,
  "endpoints": { "sensor": "...", "protected": "..." },
  "headers": { "set": ["..."], "order": ["..."] }
}
```

## Design rules that fall out of the recon

- The orchestrator is profile-driven. Adding a vendor is writing a new `profile.json`, not new control
  flow.
- Keep the three axes separate. Many targets need only one. A TLS-only site is a `transport/` job at C0,
  with no browser anywhere.
- Whenever a tier uses a real browser, keep the credential step and the protected request in one context
  so `Origin`, cookies, and TLS/JA3 are the engine's for free — transport fidelity is then a side
  effect, not a component.
- Treat single-use nonces as a cache-invalidation rule: one credential per protected request, never
  replayed.
- The evaluation gate (golden trace + negative control) is a runtime component, not a one-off test.
  Nothing ships without a golden trace to grade it against. A drop in the golden trace's acceptance is
  how you learn the vendor shipped a new build, which re-triggers recon.
- Cost scales with the credential tier, not with the orthogonal axes. C0-C2 are cheap and parallel; C3+
  spends a real browser. Hold each target at the lowest credential tier its profile allows.

## Piloting / transport tools

`puppeteer`/`playwright` (+ `puppeteer-extra-plugin-stealth`); prefer `puppeteer.connect` to a real GPU
Chrome, or CDP `Network.getAllCookies` for HttpOnly cookies. For replaying an extracted credential from
outside a browser, match the TLS fingerprint (`curl-impersonate` / `utls` / `tls-client`) and header
order.
