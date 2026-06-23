---
name: antibot-recon
description: >-
  Reverse-engineer and assess any client-side anti-bot / bot-detection platform, then determine
  whether its "human" credential can be produced programmatically. Covers the full landscape, not
  one vendor: token systems (TrustSig X-TrustSig-Response, Kasada x-kpsdk, hCaptcha/Turnstile),
  cookie systems (DataDome `datadome`, Akamai `_abck`/`bm_sz`, Imperva `reese84`, PerimeterX/HUMAN
  `_px`), proof-of-work challenges, and passive behavioral scoring. Use when triaging which
  anti-bot vendor/shape a site uses, deobfuscating an anti-bot SDK (string-array OR VM/control-flow
  obfuscation), cracking a WASM challenge, recovering a sensor cookie/token, identifying an
  attestation-token scheme (Private Access Tokens / Privacy Pass), reproducing or rendering a hard
  fingerprint surface (canvas, WebGL/shaders, audio) from a real browser, or judging the attacker
  capability needed to automate a protected site.
---

# Anti-bot platform recon & bypass assessment

A repeatable, vendor-agnostic methodology. Anti-bot SDKs differ a lot in surface detail — token vs
cookie, WASM vs pure-JS VM, active challenge vs passive scoring — but they share one loop, and the
same triage-then-pilot strategy works across all of them. **Triage first; don't assume a shape.**

## The universal model

Every client-side anti-bot system runs the same four-step loop. Only step 3 (the *credential*)
varies, and that variation is what you must identify before committing to an approach:

1. **Collect signals** — device/browser fingerprint (canvas/WebGL/audio/fonts/navigator/screen),
   behavior (pointer/keystroke/timing), and environment/automation tells (`webdriver`, headless,
   software renderer).
2. **Prove to the server** — a sensor / handshake / collect request, sometimes gated on a
   server-issued **challenge** or a **proof-of-work**.
3. **Receive a credential** — *this is the part that varies by vendor:*
   - a **token** in a JSON body or response header (TrustSig `X-TrustSig-Response`, Kasada
     `x-kpsdk-ct`, hCaptcha/Turnstile/reCAPTCHA form token);
   - a **cookie** set on the sensor response (DataDome `datadome`, Akamai `_abck`/`bm_sz`,
     Imperva `reese84`/`incap_ses`, PerimeterX/HUMAN `_px3`, Cloudflare `cf_clearance`);
   - or **nothing explicit** — a passive *risk score* the server keeps per session/IP, gated only
     on the telemetry you keep sending;
   - or an **attestation token** minted by a trusted issuer rather than computed in the page (Private
     Access Tokens / Privacy Pass; an HTTP `WWW-Authenticate: PrivateToken` challenge). Its strength
     is the issuer's, not the page's — see the attestation section.
4. **Attach the credential** to protected requests (header / cookie / query param / form field),
   which the origin or edge validates and/or re-scores.

Your job: identify which shape you face, then produce the credential. **For almost every shape the
credential is producible by running the genuine client** — the open question is only whether you
also need to run it *offline / at scale*, which is where the real work lives.

## Phase A — Triage (do this BEFORE any deobfuscation)

Load the protected site in a real browser, behave like a user, and watch the **network + cookie
jar + storage**. Answer these before writing any analysis code:

- **What is attached to protected requests?** Diff a successful protected request against a blocked
  one. Is the differentiator a request header (`x-…`), a **cookie**, a query param, or a body
  field? That is the credential — name it exactly.
- **Where does it come from?** Find the response that *sets the cookie* or *returns the token* —
  the sensor/handshake/collect endpoint. First-party path or a third-party vendor domain?
- **What client tech?** Is there a `.wasm`? an iframe? a web worker? Or pure JS? Inspect the
  loader: commodity string-array obfuscation (obfuscator.io) vs a custom **VM / control-flow
  flattening** vs minified-only.
- **Is there a challenge or PoW?** A server-issued program the client must run, a compute/hash
  loop, or an interactive CAPTCHA that appears on suspicion.
- **How is failure enforced?** `403` + a vendor string, a JS challenge interstitial, a CAPTCHA
  redirect, or silent down-ranking. The block message often names the vendor.

Classify against the table, then pick your path.

## Recognition guide (common shapes)

| Vendor | Credential | Transport | Client tech | Tells |
|---|---|---|---|---|
| **TrustSig** | token | `X-TrustSig-Response` header | WASM + iframe + worker | challenge VM, server-minted tables |
| **DataDome** | cookie | `datadome` cookie | JS sensor (+WASM) | `/js/` POST; CAPTCHA fallback |
| **Akamai Bot Manager** | cookie | `_abck`, `bm_sz` | heavy JS sensor | `sensor_data`; `_abck` valid-flag |
| **Kasada** | token | `x-kpsdk-ct`/`-cd` | WASM (`ips.js`) + VM | proof-of-work; `/ips.js`,`/tl` |
| **PerimeterX / HUMAN** | cookie | `_px3`/`_px2` | JS sensor | risk cookie |
| **Imperva / Incapsula** | cookie | `reese84`/`incap_ses` | JS challenge | `reese84` sensor |
| **Cloudflare BM/Turnstile** | cookie/token | `cf_clearance` / token | JS + PoW (+WASM) | managed challenge |
| **hCaptcha / reCAPTCHA / Turnstile** | token | form field / header | iframe + behavioral/PoW | visible or invisible |

(Vendors rename and re-shape constantly — use the table as a recognition aid, confirm by triage.)

## Attestation tokens (Private Access Tokens / Privacy Pass)

Some sites do not fingerprint at all. They accept an *attestation token* minted by a trusted issuer —
the Private Access Token / Privacy Pass family (Apple devices, Cloudflare and Fastly as issuers, the
PACT proposal). This is a different shape from everything else here and it changes the whole strategy,
so catch it in triage before you build any fingerprint machinery.

**Identify it.**
- The server answers a protected request with an HTTP auth challenge: `WWW-Authenticate: PrivateToken
  challenge=…, token-key=…`, and the client returns `Authorization: PrivateToken token=…`.
- The token is fetched from an *issuer* (a CDN such as Cloudflare/Fastly, or a platform), not computed
  in the page. There is little or no JS sensor, no canvas/WebGL/audio probing, no behavioral beacon.
- On Apple platforms the OS handles it transparently with no UI. It is usually offered as a way to
  *skip a CAPTCHA*, so it sits next to a CAPTCHA/fingerprint fallback rather than replacing it.

**The issuer is the real gate — three cases, cheapest first.** A token is only as strong as the
issuer's attestation, so decide which case you are in:
- **PAT is optional (an accelerator).** The site still accepts the fallback path: solve the CAPTCHA,
  or pass the fingerprint/JS challenge on the C-ladder. Ignore the token and take the fallback; confirm
  with the negative control that the fallback is accepted.
- **The issuer's attestation is software-grade.** A CDN or identity-provider issuer with no device
  root of trust falls back on fingerprinting, account history, or a CAPTCHA to decide whether to sign.
  The real target is then the issuer's own check — recurse: triage *that* step and defeat it on the
  C-ladder. The PAT wrapper adds nothing new.
- **The issuer requires hardware attestation.** Apple's PAT signs only after the Secure Enclave attests
  the device, with an Apple account in good standing involved; Android's analog is hardware-backed key
  attestation / Play Integrity. There is no value to synthesize and no call site to hook. The token is
  produced by a hardware root of trust you do not control.

**When it is hardware-backed, real hardware is the requirement.** This is the one branch where piloting
a browser does not help, because the attestation is rooted in the device, not the page.
- **Use a genuine attested device** in good account standing. One real iPhone/Mac (or attested
  Android) mints tokens; a pool of them is the device-farm analog of a browser farm.
- **Scale is expensive and accountable.** Tokens bind to a device and an account the issuer can
  rate-limit, age-check, and revoke. Rotating identity means rotating real, provisioned,
  account-bound devices, not spinning a process. Cost jumps from "a real browser" to "a real,
  accountable device" — the strongest economic bar in this document.
- **Do not try to forge it.** Faking Secure Enclave / Play Integrity attestation is a platform
  root-of-trust exploit, out of scope for recon. If the device root of trust holds, the token cannot
  be produced off-device.

So: in triage, decide optional vs software-grade vs hardware-backed. That call is the difference
between "no browser needed," "defeat the issuer's check on the C-ladder," and "real devices needed."

## Phase B — The universal strategy: pilot the genuine client

For **any** shape, the lowest-cost path is to run the unmodified vendor client in a real browser on
the protected origin, let it mint the credential, and read it off. This needs **zero** understanding
of the obfuscation — you run *their* code, not a replica.

- **Token systems** — drive the SDK API (or just let the page run), then capture the token from the
  JSON/header/JS global. (TrustSig: `fetch+eval` core.js → `init` → `scan` → `getResponse`.)
- **Cookie systems** — load the page, let the sensor run, read the credential **cookie** from the
  jar (`document.cookie`, or CDP `Network.getAllCookies` for HttpOnly), then replay protected
  requests carrying that cookie (with matching UA/headers/TLS).
- **Behavioral / score systems** — there is no artifact to copy; you must keep generating plausible
  telemetry from a real(istic) session and reuse that session.

Requirements that recur: a **real engine**, usually a **real GPU** (software renderers are flagged),
and the **allowed origin** (many edges bind the credential to `Origin`/host). Keep the credential-
producing step *and* the protected request in the same browser context when you can, so TLS/JA3,
`Origin`, and cookies are all genuinely the engine's and need no impersonation.

## Phase C — Deep analysis (only if piloting a full browser per request is too costly)

Do this when you want a lighter/offline/scaled replica. Pick branches by what triage found — not
every target has WASM, a challenge, or a token.

- **C1 · Deobfuscate the client JS, structurally.** Two families: **(a) commodity string-array**
  (obfuscator.io: string-array + property/numeric decoders, per-build renaming) — detect decoders
  by *shape*, not name, so you survive rebuilds; build a DAG of passes run to a fixpoint (sandboxed
  V8 eval, cross-file decoder resolution, member-fold, dead-code elimination). **(b) custom VM /
  control-flow flattening** (Kasada, parts of Akamai) — recover the dispatch table and opcode
  handlers; substantially more work; consider dynamic tracing over static rewriting.
- **C2 · If there's WASM.** Extract any runtime string table by **executing the real glue+WASM in a
  Node `vm`** (don't reverse the cipher — run it; supply `crypto.getRandomValues` via `webcrypto`).
  Decompile with `wabt`/`wasm-tools`; map the `wasm-bindgen` import trampolines; name probes by
  correlating the runtime `table#index` sequence with the deobfuscated glue.
- **C3 · Map the credential pipeline & telemetry.** The handshake/sensor request, any signing/AEAD,
  and how the token/cookie is assembled. For behavioral systems, enumerate the telemetry fields
  (event schema, timing, entropy) you'd need to synthesize.
- **C4 · Reproduce the fingerprint surface (the hard part).** The client reads canvas/WebGL/audio/
  font/navigator signals — often **synchronously** — and the server **scores** them. An offline
  replica must emit a *self-consistent, real-looking* set. This is *possible* (replay captured
  values, or render with real native libs) but it is a fidelity arms race against server-side
  scoring — which is exactly why piloting a real browser (perfect fidelity, free) usually wins on
  cost. The conclusion to reach is a **capability tier**, not "possible/impossible" (see below).

## Hard-to-emulate surfaces: recognize, evaluate, source from a real browser

When an offline replica is the goal, the cost concentrates in a few surfaces whose values you cannot
cheaply fabricate. Find them, rate them, and decide which to satisfy with a real browser instead of
synthesizing.

**Why a surface is hard.** It is expensive to emulate when its output depends on the real machine and
the server can tell. Rate each surface on:

- **Hardware-derived output** — produced by the GPU, audio DSP, or codecs, and varies by
  driver/renderer/OS (canvas, WebGL, audio). Synthetic values rarely match any real population.
- **Synchronous read** — the client reads it inline during execution, so there is no lazy hook point
  to inject a value computed elsewhere without intercepting the exact call.
- **Internal consistency** — the value is cross-checked against the rest of the profile. A UA that
  claims macOS must pair with an Apple WebGL renderer, a matching DPR/screen, a plausible font set.
  One faked value that does not agree with the others is the tell.
- **Server-side scoring against a known-good corpus** — the vendor compares your value to a
  population of real ones. Novel or software-renderer values score as bot even if internally valid.
- **Dynamic / per-challenge input** — the server dictates *what* to render or compute this session
  (a string to draw, a shader to run, a nonce to hash), so any previously captured value is stale.

High on several of these means source it live from a real browser. Low on all means synthesize or
replay a captured value.

**Per-surface read:**

- **Canvas 2D (text + geometry)** — renderer- and font-stack-dependent anti-aliasing. Moderate to
  hard, and the *content is often dynamic* (server-specified text/shapes), which kills static replay.
  Render live in a real engine per challenge.
- **WebGL / shaders** — `UNMASKED_RENDERER` and `VENDOR`, plus `readPixels` of a rendered scene.
  GPU/driver/ANGLE-backend specific. Hard. Software fallback (SwiftShader, llvmpipe) is itself a bot
  signal, so you need a real GPU or a real browser on real hardware.
- **Audio (`OfflineAudioContext`)** — oscillator → compressor → `getChannelData` sum. Float-path and
  build dependent but stable per engine build. Moderate; capture once per (browser build, OS) and
  replay if the server does not vary the graph.
- **Fonts / `measureText` / enumeration** — installed fonts plus rasterizer. Moderate; stable per
  machine, so capture-and-reuse works unless it is cross-checked against the UA/platform.
- **Media codecs (`canPlayType`), WebRTC, `getClientRects`, plugins** — enumerable and mostly static.
  Easy to mirror from one captured real profile.
- **Timing, `performance.now` jitter, GC behavior** — behavioral, usually scored loosely. Easy to
  approximate.

**Approaches to source the hard surfaces from a real browser:**

- **Full pilot (default).** Run the genuine client in a real browser on real hardware. Every surface
  is authentic and self-consistent with no extra work; heaviest per request (Phase B).
- **Hybrid render-and-inject.** Keep the cheap pipeline headless/offline, and stand up a real browser
  as a *rendering oracle* for the hard surface only: feed it the server's challenge input (the
  text/shader/params), let it produce the canvas/WebGL/audio value on real hardware, capture the
  value, and inject it at the exact call site in your emulated client. Worth it only when the hard
  surface is small and well isolated and the rest of the pipeline is cheap to run.
- **Capture-and-replay.** For *static* surfaces (no per-session variation), record the real value
  once per (browser build, OS, GPU) and reuse it. Confirm with the negative control that the server
  is not varying the input; if it is, the replay is stale and you must render live.
- **Real-GPU configuration.** Headless Chrome defaults to SwiftShader. For authentic GPU output, run
  headed on a GPU box, run headless with GPU enabled and the right ANGLE backend
  (`--use-angle=metal`/`d3d11`/`gl`), or `puppeteer.connect` to a real desktop Chrome. Match the
  GPU/driver to the device profile you are claiming.
- **Remote real browsers / farms.** A pool of real (or real-GPU) browsers behind an emulated front
  end gives throughput. Rotate device profiles and proxies so the authentic fingerprints do not
  cluster on one machine.

**The call that decides everything:** is the hard surface's *input* dynamic? If the server fixes what
to render or compute each session (common for canvas-text and shader challenges), captured values are
stale and you need a live real engine per request. If the input is static, capture a profile once and
replay. Make this determination early, with the negative control, because it sets whether you are
running a browser per request or just a one-time capture.

## Identifying behavioral signal collection

Behavioral scoring watches how a user moves, types, and touches. It is the other collection branch
next to the device fingerprint, and you recognize and rate it the same way.

**Find the listeners (static).** In the deobfuscated loader, look for `addEventListener` (or `on*`
handlers) on `document` / `window` / `body` for: `mousemove`, `mousedown`/`mouseup`, `click`,
`wheel`, `scroll`, `keydown`/`keyup`/`keypress`, `pointer*`, `touchstart`/`touchmove`/`touchend`,
`devicemotion`, `deviceorientation`, `focus`/`blur`, `visibilitychange`. Trace each listener to the
buffer it appends to, and that buffer to the field it serializes into the sensor payload.

**Confirm at runtime (dynamic).** Wrap `EventTarget.prototype.addEventListener` (log type + target)
before the SDK loads, or read `getEventListeners(document)` in DevTools. Then interact and diff the
sensor payload: move the mouse, type, scroll, and watch which bytes grow. The fields that change
under interaction are the behavioral channel.

**Watch the beacon.** Behavioral telemetry is batched and shipped on a timer or on
`visibilitychange`/`beforeunload`, often via `navigator.sendBeacon` or a periodic `fetch`/XHR with an
encoded body. A payload that grows with interaction and flushes periodically is the signature.

**Rate it on two axes.** (1) *What is captured* — raw event streams (x, y, t per move; key dwell and
flight times; touch radius/pressure; scroll cadence; idle gaps) versus coarse aggregates (counts,
entropy, presence flags). Raw streams are harder to fake convincingly. (2) *Does it gate the
credential* — produce the credential with **zero** behavioral input and run the negative control. If
it still passes, behavior is a soft server-side score you can defer; if it blocks, you must supply
plausible behavior. (TrustSig collected pointer/keyboard/touch in the loader but did not gate the
token on it; many vendors only score it server-side.)

**Synthesis difficulty.** Convincing behavior is a distribution-matching problem, not a value to
copy. Cheapest first: ignore it if it does not gate; replay recorded real human sessions, rescaled
and retimed to the page; generate physics- or Bézier-based mouse paths with human key/touch timing;
or pilot a real browser and drive synthetic-but-plausible events through it so the rest of the
fingerprint stays authentic. Interactive challenges (slider, click-the-images) are the same signal
under a controlled task, with a perception problem stacked on top of the motion problem.

## The decisive constraint (why a real engine is the cheap path)

Fingerprint reads are commonly synchronous and must *be* a real engine's; software renderers and
headless tells are scored against you. So the cost gradient is: pure HTTP replay (cheapest, usually
blocked) → offline JS/WASM replica with synthesized fingerprints (hard, fidelity-bound) → **drive a
real browser** (easy, high-fidelity, but heavier per request). Most engagements land on the last
rung because it's the path of least resistance, not because the others are impossible.

## Negative control (mandatory, every shape)

Prove the credential genuinely passes — a result without a control is a guess:
- **token:** send none / garbage / **tampered** (flip one byte) / yours → only *yours* should reach
  the app layer; tampered should be rejected (proves an integrity check, not just presence).
- **cookie:** replay with no cookie / a forged cookie / yours → only *yours* should succeed.
- **score:** compare a fresh-session request against your reused session; confirm the difference.
If a "should-be-blocked" case also succeeds, the gate isn't validating and your result is a false
positive.

## The evaluation gate (ground truth or nothing)

A bypass that "works once" against a live endpoint proves nothing. Anti-bot scoring is probabilistic,
stateful, and rate-aware, so a single pass can be luck, soft-score headroom, or IP reputation you have
not spent yet. **Establish ground truth first, or you cannot establish success at all.** Stand this
gate up right after recon and run every build stage through it.

**Capture a golden trace.** From an environment the vendor genuinely accepts — a real browser on real
hardware (or a real attested device for PAT), on the allowed origin — record one full successful
interaction end to end: the handshake/sensor exchange, the challenge, the credential that was issued,
the request/response headers, the TLS fingerprint, the device-fingerprint values the client read, the
behavioral telemetry, and the protected request that was accepted. Confirm acceptance with the negative
control. This trace is your oracle.

Capture the *recipe*, not just the values. Single-use nonces mean the credential in the trace is
already spent; what you keep is the inputs (challenge, device profile, behavior) and the accepted
output, so you can regenerate and compare rather than replay a dead token.

**Gate each stage against it.** As you build, confirm or refute each component before moving on:
- **Trace is live.** Re-run the golden capture against the endpoint. If it no longer passes (expired
  nonce, vendor rebuild, burned IP), the ground truth is stale — re-capture before trusting any
  comparison.
- **Field diff.** For every value your system produces — credential, fingerprint fields, header set
  and order, TLS signature, behavioral stream — diff against the golden trace. Exact match where the
  value should be stable; distributional match where it legitimately varies.
- **Localize divergence.** When your replica is rejected, the field diff points at the stage that
  differs from known-good. That is the bug. A pass/fail at the endpoint alone does not localize; the
  trace does.
- **Differential test.** Run the golden trace and your replica through the same endpoint under the same
  conditions and compare outcomes. Same input, different verdict means your divergence is the cause.
- **Discriminate acceptance.** Classify every response with the negative control as calibration:
  garbage/tampered must BLOCK, the golden trace must PASS, yours must land with the golden trace and
  not with the garbage.

**Truth discipline.**
- One pass is not success. Require repeated passes, and require the negative control to be
  discriminating in the same run, or you are measuring noise.
- Validate at the threshold and volume you will operate at. A request can pass while accumulating risk;
  soft scores and rate limits only surface under load.
- Acceptance drifts. Re-capture ground truth on a schedule and treat a drop in the golden trace's
  acceptance as the signal that the vendor shipped a new build, which re-triggers recon.
- Keep a regression suite of golden traces (across builds, device profiles, origins) and re-run it on
  every change to the bypass.

**The rule.** No golden trace from a genuinely-accepted environment means no oracle, which means you
can confirm a refutation (a BLOCK) but never confirm success. Treat "no ground truth" as "not
testable," and get the trace before you build.

## Tools
- JS: `tree-sitter` + `tree-sitter-javascript`, a V8 sandbox (`mini-racer`) for safe decoder eval,
  `jsbeautifier`; a DAG-based pass pipeline for the string-array family.
- WASM: `wabt` + `wasm-tools` (`brew install wabt wasm-tools`); Node `vm` + native
  `WebAssembly`/`TextDecoder`/`webcrypto` for the offline harness.
- Piloting: `puppeteer`/`playwright` (+ `puppeteer-extra-plugin-stealth`); prefer
  `puppeteer.connect` to a real GPU Chrome, or CDP `Network.getAllCookies` for HttpOnly cookies.
  For replaying an extracted credential from outside a browser, match the TLS fingerprint
  (`curl-impersonate` / `utls` / `tls-client`) and header order.
- Fidelity (offline path): `@napi-rs/canvas` (Skia), `node-web-audio-api` — note these emit the
  *library's* fingerprint, not a browser's.

## From recon to a system: complexity ladder and architecture

The recon output is a per-target *profile*: the credential, its transport, the endpoints, which
device surfaces and behaviors are checked, and whether each is **static or dynamic**. Solve exactly
what the profile shows. The expensive axis — generating an accepted credential — is separate from two
cheaper, orthogonal axes — transport fidelity and scale. Conflating them is how you end up building a
browser farm to beat a site whose only defense is a TLS fingerprint.

**Credential ladder (the main axis).** This is what most of the recon is about: producing one accepted
credential. Each tier is a superset of the ones below. Build to the highest tier recon found, no
further.

| Tier | Credential defense in recon | What the system must do | Component |
|---|---|---|---|
| C0 | none, or a static credential | send the request | HTTP client |
| C1 | credential derived by simple JS, no hardware FP | run the derivation offline | JS-sandbox runner (`mini-racer`/node `vm`) |
| C2 | device fingerprint required, **static** challenge input | inject a captured real profile | profile store + emulated runner |
| C3 | **dynamic** per-session challenge, real-GPU FP | render the hard surface live | real-browser oracle (hybrid) or full pilot |
| C4 | behavioral scoring or automation detection that **gates** the credential | supply plausible behavior, hide hooks | behavior engine + stealthed real browser |

Attestation tokens (PAT / Privacy Pass) are not a C-tier. They are a separate branch: a hardware-backed
issuer (Apple Secure Enclave, Android Play Integrity) needs a real attested device, not a browser,
while an optional or software-grade one drops back to a fallback or to the issuer's own check. See the
attestation section, and make the call in triage.

**Two orthogonal axes — add only if recon shows them, at whatever credential tier you are on.** These
are not higher tiers. They are usually cheap, and either one can be the *entire* defense.

- **Transport fidelity** — TLS/JA3/JA4, HTTP/2 settings, header order. If the edge fingerprints the
  connection, a plain HTTP client is rejected before any JS runs. Match it with a browser-TLS
  impersonation client (`curl-impersonate`, `utls`, `tls-client`) and correct header order. It is
  cheap, independent of credential complexity, and a real browser supplies it for free. **If
  transport fingerprinting is the only defense (no JS/cookie/token challenge), that is the whole job —
  use an impersonation client, not a browser.**
- **Scale and identity** — origin binding, single-use nonces, rate-limit/clustering. Origin binding is
  cheap: send the allowed `Origin`/host and the right cookies. Single-use nonces mean one credential
  per request, never replayed. Rate-limit and fingerprint-clustering matter only when you need
  *volume*; then add proxy rotation and a pool of device profiles so authentic identities do not
  cluster on one host. None of this changes how a single credential is generated.

The triage principle holds: defeat only what the target checks. A site protected by JA3 alone needs a
TLS-impersonation client, not a browser. A WASM challenge with no transport check needs a real browser
but no impersonation. Do not pay for an axis the target does not use.

**Decision flow (per request).**

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

**Code layout.**

```
antibot-defeat/
  recon/                        # per-target, one-time; output is the profile below
    triage.json                 # credential, transport, endpoints, vendor shape
    deobf/                      # structural JS passes (string-array; VM/control-flow)
    wasm/                       # extract tables by execution; decompile; probe inventory
    surfaces.json               # device surfaces + behaviors checked, each static|dynamic
  targets/<vendor>/profile.json # the contract the runtime consumes
  core/
    orchestrator/               # reads profile, picks the credential tier + which axes apply
    runners/
      http/                     # C0-C1: replay + JS-sandbox credential derivation
      emulated/                 # C2: offline client with injected captured fingerprint
      browser/                  # C3-C4: real-browser pilot and rendering oracle
    fingerprint/                # capture, consistency-check, and serve device profiles
    behavior/                   # mouse/key/touch synthesis and human-session replay
    challenge/                  # parse the server challenge; route hard surfaces to the oracle
    transport/                  # ORTHOGONAL: TLS-impersonation + header-order (skip when piloting a browser)
    scale/                      # ORTHOGONAL: proxy + device-profile rotation (only for volume)
  pool/                         # warm real(-GPU) browsers; bind profile + proxy per session
  control/                      # the evaluation gate: golden-trace diff + negative-control acceptance
  store/                        # captured profiles, golden traces, recorded sessions, credential cache
```

**Design rules that fall out of the recon.**

- The orchestrator is profile-driven. Adding a vendor is writing a new `profile.json`, not new control
  flow.
- Keep the three axes separate. Many targets need only one. A TLS-only site is a `transport/` job at
  C0, with no browser anywhere.
- Whenever a tier uses a real browser, keep the credential step and the protected request in one
  context so `Origin`, cookies, and TLS/JA3 are the engine's for free — transport fidelity is then not
  a component you build, it is a side effect.
- Treat single-use nonces as a cache-invalidation rule: one credential per protected request, never
  replayed.
- The evaluation gate (golden trace + negative control) is a runtime component, not a one-off test.
  Nothing ships without a golden trace to grade it against. A drop in the golden trace's acceptance is
  how you learn the vendor shipped a new build, which re-triggers recon for that target.
- Cost scales with the credential tier, not with the orthogonal axes. C0-C2 are cheap and parallel;
  C3+ spends a real browser. Hold each target at the lowest credential tier its profile allows.

## What you'll conclude

For most platforms you can produce a working credential by **piloting the genuine client**. Piloting
doesn't require understanding, so the obfuscation and the cryptographic story add little durable cost
to that attack. The real cost is conventional anti-bot economics: a real engine, a plausible
fingerprint, an allowed origin, and (to scale) rotated device profiles plus proxies. State your
result as an **attacker-capability tier** (pure replay, offline replica, single real browser, rotated
browser farm) and what each one costs. That is what a defender needs to model, and it beats a
"bypassed: yes/no" verdict.
