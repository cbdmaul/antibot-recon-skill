# Phase 1 reference — Triage

Goal: name the credential and its shape **before** writing any analysis code. Triage is cheap and
decides everything downstream, so do not skip it. Output is `recon/triage.json`.

## The triage questions

Load the protected site in a real browser, behave like a user, and watch the **network + cookie jar +
storage**. Answer:

- **What is attached to protected requests?** Diff a successful protected request against a blocked
  one. Is the differentiator a request header (`x-…`), a **cookie**, a query param, or a body field?
  That is the credential — name it exactly.
- **Where does it come from?** Find the response that *sets the cookie* or *returns the token* — the
  sensor/handshake/collect endpoint. First-party path or a third-party vendor domain?
- **What client tech?** Is there a `.wasm`? an iframe? a web worker? Or pure JS? Inspect the loader:
  commodity string-array obfuscation (obfuscator.io) vs a custom **VM / control-flow flattening** vs
  minified-only.
- **Is there a challenge or PoW?** A server-issued program the client must run, a compute/hash loop,
  or an interactive CAPTCHA that appears on suspicion.
- **Is there an attestation challenge?** A `WWW-Authenticate: PrivateToken` response means PAT /
  Privacy Pass — a different shape entirely. See `attestation.md`.
- **How is failure enforced?** `403` + a vendor string, a JS challenge interstitial, a CAPTCHA
  redirect, or silent down-ranking. The block message often names the vendor.

## Parallelize triage with a recon subagent team

The modalities above are independent, so fan them out in **one batch** of subagents and merge their
findings. Each subagent returns a small structured result; you assemble `triage.json`.

- **network-diff** — capture a successful protected request and a blocked one; diff headers, cookies,
  query, and body; return the single differentiator (the credential) and the endpoints involved.
- **storage-inspector** — dump `document.cookie`, `localStorage`, `sessionStorage`, IndexedDB; return
  which keys change across the sensor exchange.
- **loader-classifier** — fetch and skim the loader/sensor scripts; return client tech (wasm? iframe?
  worker?) and the obfuscation family (string-array vs VM/control-flow vs minified).
- **block-prober** — trip the defense (e.g. a headless/blank request) and capture the block response,
  status, and vendor string; return how failure is enforced.
- **attestation-probe** — send a bare protected request and inspect `WWW-Authenticate`; return whether
  PAT is offered and whether a fallback (CAPTCHA/fingerprint) is present.

Merge rule: the credential name and transport come from network-diff; tech from loader-classifier;
enforcement from block-prober; PAT case from attestation-probe. Conflicts are resolved by re-running
the specific modality, not by guessing.

## Recognition guide (common shapes)

Vendors rename and re-shape constantly; use this to recognize, then confirm by triage.

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

## Deliverable: `recon/triage.json`

```json
{
  "vendor": "best guess from tells, or unknown",
  "credential": { "kind": "token|cookie|score|attestation", "name": "X-...|_abck|...", "transport": "header|cookie|query|body|form" },
  "endpoints": { "sensor": "...", "protected": "..." },
  "client_tech": { "wasm": true, "iframe": true, "worker": true, "obfuscation": "string-array|vm|minified" },
  "challenge": { "present": true, "kind": "vm|pow|captcha|none" },
  "attestation": { "pat": false, "case": "none|optional|software|hardware" },
  "enforcement": "403:<vendor string> | interstitial | captcha-redirect | down-rank"
}
```

## Gate

You can name the credential and its transport, you know the client tech, and you have classified the
attestation case. If PAT is **hardware-backed**, stop here and read `attestation.md` — most of the
build pipeline does not apply, and the requirement becomes real attested hardware.
