# antibot-recon

A [Claude](https://claude.com/claude-code) skill for reverse-engineering and assessing client-side
anti-bot / bot-detection platforms, and judging whether their "human" credential can be produced
programmatically.

It is vendor-agnostic. It covers token systems (TrustSig, Kasada, hCaptcha/Turnstile), cookie systems
(DataDome, Akamai, Imperva, PerimeterX/HUMAN), proof-of-work challenges, and passive behavioral
scoring. The methodology is a triage-first playbook: identify the credential and transport, deobfuscate
the client structurally, extract any WASM-backed tables by executing the real code, decompile the
challenge VM and enumerate its checks, then decide a run strategy and prove the result with a negative
control.

## What it is

A multi-document skill. `SKILL.md` is the spine: the universal model, the core principles, and a
**phased pipeline** where each phase produces a named artifact that fills out one project layout
(triage → evaluation gate → decompose → profile → build → orthogonal axes → verify). Depth for each
phase lives in `references/`, loaded only when that phase is reached:

- `references/triage.md` — Phase 1 triage and the recon subagent team
- `references/attestation.md` — PAT / Privacy Pass and hardware-attestation reasoning
- `references/decompose.md` — JS deobfuscation and WASM analysis
- `references/surfaces.md` — hard fingerprint surfaces and behavioral collection
- `references/architecture.md` — credential ladder, orthogonal axes, and the code layout
- `references/evaluation-gate.md` — golden trace and negative control

Claude reads the `description` to decide when the skill is relevant, then follows `SKILL.md`, pulling in
references as each phase needs them.

## Install

Clone it into your Claude skills directory under the skill's name:

```bash
git clone https://github.com/cbdmaul/antibot-recon-skill ~/.claude/skills/antibot-recon
```

Claude Code discovers skills under `~/.claude/skills/<name>/SKILL.md` automatically. Update with
`git -C ~/.claude/skills/antibot-recon pull`.

## Scope and intent

This skill describes a methodology for analyzing anti-bot systems. It is intended for authorized
security research, competitive analysis, and defensive work against systems you own or are permitted
to test. It does not contain a working bypass for any specific product.

## License

MIT
