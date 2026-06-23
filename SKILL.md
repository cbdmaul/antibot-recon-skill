---
name: antibot-recon
description: >-
  Reverse-engineer and assess any client-side anti-bot / bot-detection platform, then determine
  whether its "human" credential can be produced programmatically. Covers the full landscape, not
  one vendor: token systems (TrustSig X-TrustSig-Response, Kasada x-kpsdk, hCaptcha/Turnstile),
  cookie systems (DataDome `datadome`, Akamai `_abck`/`bm_sz`, Imperva `reese84`, PerimeterX/HUMAN
  `_px`), proof-of-work challenges, and passive behavioral scoring. Use when triaging which
  anti-bot vendor/shape a site uses, deobfuscating an anti-bot SDK (string-array OR VM/control-flow
  obfuscation), cracking a WASM challenge, recovering a sensor cookie/token, reproducing a
  fingerprint surface, or judging the attacker capability needed to automate a protected site.
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
  by *shape*, not name, so you survive rebuilds; see `jsdeobf/` in this repo (DAG of passes to a
  fixpoint, sandboxed V8 eval, cross-file decoder resolution, member-fold, DCE). **(b) custom VM /
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
  `jsbeautifier`; the `jsdeobf/` toolkit here for the string-array family.
- WASM: `wabt` + `wasm-tools` (`brew install wabt wasm-tools`); Node `vm` + native
  `WebAssembly`/`TextDecoder`/`webcrypto` for the offline harness.
- Piloting: `puppeteer`/`playwright` (+ `puppeteer-extra-plugin-stealth`); prefer
  `puppeteer.connect` to a real GPU Chrome, or CDP `Network.getAllCookies` for HttpOnly cookies.
  For replaying an extracted credential from outside a browser, match the TLS fingerprint
  (`curl-impersonate` / `utls` / `tls-client`) and header order.
- Fidelity (offline path): `@napi-rs/canvas` (Skia), `node-web-audio-api` — note these emit the
  *library's* fingerprint, not a browser's.

## What you'll conclude

For most platforms you can produce a working credential by **piloting the genuine client**. Piloting
doesn't require understanding, so the obfuscation and the cryptographic story add little durable cost
to that attack. The real cost is conventional anti-bot economics: a real engine, a plausible
fingerprint, an allowed origin, and (to scale) rotated device profiles plus proxies. State your
result as an **attacker-capability tier** (pure replay, offline replica, single real browser, rotated
browser farm) and what each one costs. That is what a defender needs to model, and it beats a
"bypassed: yes/no" verdict.
