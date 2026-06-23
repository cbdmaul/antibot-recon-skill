# Phase 3 reference — Decompose the client

Only needed when you want a lighter / offline / scaled replica instead of piloting a real browser per
request. Pick branches by what Phase 1 found — not every target has WASM, a challenge, or a token. The
branches below are independent; run them as parallel subagents and merge.

## Deobfuscate the client JS, structurally

Two families:

- **(a) Commodity string-array** (obfuscator.io: string-array + property/numeric decoders, per-build
  renaming) — detect decoders by *shape*, not name, so you survive rebuilds. Build a DAG of passes run
  to a fixpoint: sandboxed V8 eval of decoders, cross-file decoder resolution, member-fold, dead-code
  elimination, beautify.
- **(b) Custom VM / control-flow flattening** (Kasada, parts of Akamai) — recover the dispatch table
  and the opcode handlers. Substantially more work; consider dynamic tracing over static rewriting.

Detecting by shape rather than identifier is what makes the work survive the vendor's per-build
renaming; an analysis keyed to names dies on the next deploy.

## If there is WASM

- Extract any runtime string table by **executing the real glue + WASM in a Node `vm`** — don't reverse
  the cipher, run it. Supply `crypto.getRandomValues` via `webcrypto`; Node provides native
  `WebAssembly` and `TextDecoder`.
- Decompile with `wabt` (`wasm2wat`, `wasm-objdump`, `wasm-decompile`) and `wasm-tools`.
- Identify the single obfuscated export; the rest is `wasm-bindgen` plumbing. It dispatches through
  import trampolines (`__hN = (…args, idx) => table[idx].apply(this, args)`). Name a probe by
  correlating the runtime `table#index` call sequence (instrument the import object) with the
  deobfuscated glue.

## Map the credential pipeline & telemetry

The handshake/sensor request, any signing/AEAD, and how the token or cookie is assembled. For
behavioral systems, enumerate the telemetry fields (event schema, timing, entropy) you would need to
synthesize — see `surfaces.md` for the behavioral branch.

## Subagent fan-out

Spawn in one batch and merge:

- **deobfuscator** — run the structural passes; return clean sources + the credential-assembly call
  graph.
- **wasm-analyst** — extract tables, decompile, return the probe inventory (which host APIs the module
  reads) and the dispatcher map.
- **per-surface analysts** — one each for canvas, WebGL/shaders, audio, fonts; see `surfaces.md`.
- **behavioral-analyst** — listeners → buffers → payload fields; see `surfaces.md`.

## Tools

- JS: `tree-sitter` + `tree-sitter-javascript` (parse/locate/splice), a V8 sandbox (`mini-racer`) for
  safe decoder eval, `jsbeautifier`; a DAG-based pass pipeline for the string-array family.
- WASM: `wabt` + `wasm-tools` (`brew install wabt wasm-tools`); Node `vm` + native
  `WebAssembly`/`TextDecoder`/`webcrypto` for the offline harness.

## Deliverables

- Deobfuscated sources under `recon/deobf/`.
- `recon/wasm/` — extracted tables, the `.wat`, the probe inventory.
- A `surfaces.json` skeleton listing every device surface and behavior the client reads (filled in by
  `surfaces.md`, each tagged static|dynamic).

## Gate

Every credential-affecting input is enumerated and traced from the host API it reads to the field it
serializes. Anything you cannot trace is a gap that will surface later as a field-diff mismatch against
the golden trace.
