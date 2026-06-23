# Phase 3 reference — Surfaces & behavior: recognize, rate, source

For each device surface and behavior the client reads (from `decompose.md`), decide whether you can
synthesize/replay it or must source it live from a real browser. Tag each one **static or dynamic** in
`recon/surfaces.json` — that single tag drives the credential tier in Phase 4. Per-surface work is
independent; run a subagent per surface.

## Hard-to-emulate device surfaces

**Why a surface is hard.** It is expensive to emulate when its output depends on the real machine and
the server can tell. Rate each surface on:

- **Hardware-derived output** — produced by the GPU, audio DSP, or codecs, varies by driver/renderer/OS
  (canvas, WebGL, audio). Synthetic values rarely match any real population.
- **Synchronous read** — read inline during execution, so there is no lazy hook point to inject a value
  computed elsewhere without intercepting the exact call.
- **Internal consistency** — cross-checked against the rest of the profile. A UA claiming macOS must
  pair with an Apple WebGL renderer, a matching DPR/screen, a plausible font set. One value that does
  not agree with the others is the tell.
- **Server-side scoring against a known-good corpus** — the vendor compares your value to a population
  of real ones; novel or software-renderer values score as bot even if internally valid.
- **Dynamic / per-challenge input** — the server dictates *what* to render or compute this session (a
  string to draw, a shader to run, a nonce to hash), so any previously captured value is stale.

High on several → source it live from a real browser. Low on all → synthesize or replay.

**Per-surface read:**

- **Canvas 2D (text + geometry)** — renderer- and font-stack-dependent anti-aliasing. Moderate to hard,
  and the *content is often dynamic* (server-specified text/shapes), which kills static replay. Render
  live per challenge.
- **WebGL / shaders** — `UNMASKED_RENDERER` and `VENDOR`, plus `readPixels` of a rendered scene.
  GPU/driver/ANGLE-backend specific. Hard. Software fallback (SwiftShader, llvmpipe) is itself a bot
  signal, so you need a real GPU or a real browser on real hardware.
- **Audio (`OfflineAudioContext`)** — oscillator → compressor → `getChannelData` sum. Float-path and
  build dependent but stable per engine build. Moderate; capture once per (browser build, OS) and
  replay if the server does not vary the graph.
- **Fonts / `measureText` / enumeration** — installed fonts plus rasterizer. Moderate; stable per
  machine, so capture-and-reuse works unless cross-checked against the UA/platform.
- **Media codecs (`canPlayType`), WebRTC, `getClientRects`, plugins** — enumerable and mostly static.
  Easy to mirror from one captured real profile.
- **Timing, `performance.now` jitter, GC behavior** — behavioral, usually scored loosely. Easy to
  approximate.

**Approaches to source the hard surfaces from a real browser:**

- **Full pilot (default).** Run the genuine client in a real browser on real hardware. Every surface is
  authentic and self-consistent with no extra work; heaviest per request.
- **Hybrid render-and-inject.** Keep the cheap pipeline offline, and stand up a real browser as a
  *rendering oracle* for the hard surface only: feed it the server's challenge input (text/shader/
  params), let it produce the canvas/WebGL/audio value on real hardware, capture the value, inject it
  at the exact call site in your emulated client. Worth it only when the hard surface is small and well
  isolated and the rest is cheap to run.
- **Capture-and-replay.** For *static* surfaces, record the real value once per (browser build, OS, GPU)
  and reuse it. Confirm with the negative control that the server is not varying the input; if it is,
  the replay is stale and you must render live.
- **Real-GPU configuration.** Headless Chrome defaults to SwiftShader. For authentic GPU output, run
  headed on a GPU box, run headless with GPU enabled and the right ANGLE backend
  (`--use-angle=metal`/`d3d11`/`gl`), or `puppeteer.connect` to a real desktop Chrome. Match the
  GPU/driver to the device profile you are claiming.
- **Remote real browsers / farms.** A pool of real (or real-GPU) browsers behind an emulated front end
  gives throughput. Rotate device profiles and proxies so authentic fingerprints do not cluster.

**The call that decides everything:** is the hard surface's *input* dynamic? If the server fixes what to
render or compute each session (common for canvas-text and shader challenges), captured values are stale
and you need a live real engine per request → this forces credential tier C3. If the input is static,
capture a profile once and replay → C2. Make this call with the negative control.

## Identifying behavioral signal collection

Behavioral scoring watches how a user moves, types, and touches — the other collection branch next to
the device fingerprint.

**Find the listeners (static).** In the deobfuscated loader, look for `addEventListener` (or `on*`
handlers) on `document` / `window` / `body` for: `mousemove`, `mousedown`/`mouseup`, `click`, `wheel`,
`scroll`, `keydown`/`keyup`/`keypress`, `pointer*`, `touchstart`/`touchmove`/`touchend`, `devicemotion`,
`deviceorientation`, `focus`/`blur`, `visibilitychange`. Trace each listener to the buffer it appends
to, and that buffer to the field it serializes into the sensor payload.

**Confirm at runtime (dynamic).** Wrap `EventTarget.prototype.addEventListener` (log type + target)
before the SDK loads, or read `getEventListeners(document)` in DevTools. Then interact and diff the
sensor payload: move the mouse, type, scroll, and watch which bytes grow. The fields that change under
interaction are the behavioral channel.

**Watch the beacon.** Behavioral telemetry is batched and shipped on a timer or on
`visibilitychange`/`beforeunload`, often via `navigator.sendBeacon` or a periodic `fetch`/XHR with an
encoded body. A payload that grows with interaction and flushes periodically is the signature.

**Rate it on two axes.** (1) *What is captured* — raw event streams (x, y, t per move; key dwell and
flight times; touch radius/pressure; scroll cadence; idle gaps) versus coarse aggregates (counts,
entropy, presence flags). Raw streams are harder to fake convincingly. (2) *Does it gate the
credential* — produce the credential with **zero** behavioral input and run the negative control. If it
still passes, behavior is a soft server-side score you can defer; if it blocks, you must supply
plausible behavior (this forces credential tier C4). TrustSig collected pointer/keyboard/touch in the
loader but did not gate the token on it; many vendors only score it server-side.

**Synthesis difficulty.** Convincing behavior is a distribution-matching problem, not a value to copy.
Cheapest first: ignore it if it does not gate; replay recorded real human sessions, rescaled and
retimed to the page; generate physics- or Bézier-based mouse paths with human key/touch timing; or
pilot a real browser and drive synthetic-but-plausible events through it so the rest of the fingerprint
stays authentic. Interactive challenges (slider, click-the-images) are the same signal under a
controlled task, with a perception problem stacked on top of the motion problem.

## The decisive constraint (why a real engine is the cheap path)

Fingerprint reads are commonly synchronous and must *be* a real engine's; software renderers and
headless tells are scored against you. The cost gradient: pure HTTP replay (cheapest, usually blocked) →
offline JS/WASM replica with synthesized fingerprints (hard, fidelity-bound) → drive a real browser
(easy, high-fidelity, heavier per request). Most engagements land on the last rung because it is the
path of least resistance, not because the others are impossible.

## Fidelity tools (offline path)

`@napi-rs/canvas` (Skia), `node-web-audio-api` — note these emit the *library's* fingerprint, not a
browser's, so they help you run the VM but not pass a real-GPU scorer.

## Deliverable

`recon/surfaces.json` — every checked surface and behavior, each tagged `static|dynamic` with the
hardness rating and whether it gates the credential, plus the evidence (diffed against the golden
trace). This file decides the credential tier in Phase 4.
