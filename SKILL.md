---
name: antibot-recon
description: >-
  Reverse-engineer and assess any client-side anti-bot / bot-detection platform, then determine
  whether its "human" credential can be produced programmatically. Covers the full landscape, not
  one vendor: token systems (TrustSig X-TrustSig-Response, Kasada x-kpsdk, hCaptcha/Turnstile),
  cookie systems (DataDome `datadome`, Akamai `_abck`/`bm_sz`, Imperva `reese84`, PerimeterX/HUMAN
  `_px`), proof-of-work challenges, and passive behavioral scoring. Use when triaging which
  anti-bot vendor/shape a site uses, deobfuscating an anti-bot SDK (string-array OR VM/control-flow
  obfuscation), cracking a WASM challenge, recovering a sensor cookie/token, reproducing or rendering a hard
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
     on the telemetry you keep sending.
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
device surfaces and behaviors are checked, and whether each is **static or dynamic**. That profile
drives one decision — how much real browser you must spend per request. Build the system around that
decision, not around a single technique.

**Complexity ladder (vendor defense observed → what the system must do).** Each tier is a superset of
the ones below. Find your target's highest checked tier in recon, build up to it, and no further.

| Tier | Defense observed in recon | What the system must do | Component |
|---|---|---|---|
| L0 | static credential, no JS | send the credential | HTTP client + credential cache |
| L1 | credential derived by simple JS, no hardware FP | run the derivation offline | JS-sandbox runner (`mini-racer`/node `vm`) |
| L2 | device fingerprint required, **static** challenge input | inject a captured real profile | profile store + emulated runner |
| L3 | **dynamic** per-session challenge, real-GPU FP | render the hard surface live | real-browser oracle (hybrid) or full pilot |
| L4 | behavioral scoring and/or automation detection | supply plausible behavior, hide hooks | behavior engine + stealthed real browser |
| L5 | origin binding + TLS/JA3 + header order + rate-limit/clustering | distribute over real browsers on the allowed origin, rotate identities | browser pool + proxy/profile rotation + browser-TLS |

**Decision flow (per request).**

```
 recon profile
      │
      ▼
 credential static? ───────────────yes──► HTTP replay                      (L0/L1)
      │ no
      ▼
 device FP needed? ────────────────no───► JS-sandbox derive                (L1)
      │ yes
      ▼
 challenge input dynamic? ─────────no───► emulated runner + captured FP    (L2)
      │ yes
      ▼
 behavior / automation scored? ───no───► real-browser render or pilot      (L3)
      │ yes
      ▼
 real browser + behavior engine + stealth                                  (L4)
      │
      ▼  (always, for scale + origin/TLS binding)
 assign device profile + proxy from pool; keep the credential step and the
 protected request in one context                                          (L5)
      │
      ▼
 control harness verifies acceptance ──BLOCK──► escalate tier / rotate profile / re-capture / re-triage
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
    orchestrator/               # reads profile, picks the tier/runner per request
    runners/
      http/                     # L0-L1: replay + JS-sandbox credential derivation
      emulated/                 # L2: offline client with injected captured fingerprint
      browser/                  # L3-L5: real-browser pilot and rendering oracle
    fingerprint/                # capture, consistency-check, and serve device profiles
    behavior/                   # mouse/key/touch synthesis and human-session replay
    challenge/                  # parse the server challenge; route hard surfaces to the oracle
    transport/                  # TLS-impersonation, header-order, proxy rotation
  pool/                         # warm real(-GPU) browsers; bind profile + proxy per session
  control/                      # negative-control + acceptance verification, run continuously
  store/                        # captured profiles, recorded sessions, single-use credential cache
```

**Design rules that fall out of the recon.**

- The orchestrator is profile-driven. Adding a vendor is writing a new `profile.json`, not new
  control flow.
- Whenever a tier uses a real browser, keep the credential step and the protected request in one
  context so `Origin`, cookies, and TLS/JA3 are the engine's for free.
- Treat single-use nonces as a cache-invalidation rule: one credential per protected request, never
  replayed.
- The control harness is a runtime component, not a one-off test. Acceptance drift is how you learn
  the vendor shipped a new build, which re-triggers recon for that target.
- Cost scales with tier. L0-L2 are cheap and parallel; L3+ spends a real browser. Hold each target at
  the lowest tier its profile allows, and re-evaluate when a rebuild changes the profile.

## What you'll conclude

For most platforms you can produce a working credential by **piloting the genuine client**. Piloting
doesn't require understanding, so the obfuscation and the cryptographic story add little durable cost
to that attack. The real cost is conventional anti-bot economics: a real engine, a plausible
fingerprint, an allowed origin, and (to scale) rotated device profiles plus proxies. State your
result as an **attacker-capability tier** (pure replay, offline replica, single real browser, rotated
browser farm) and what each one costs. That is what a defender needs to model, and it beats a
"bypassed: yes/no" verdict.
