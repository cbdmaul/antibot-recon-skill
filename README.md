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

The skill is a single `SKILL.md` with YAML frontmatter (`name`, `description`) and a Markdown body.
Claude reads the `description` to decide when the skill is relevant, then follows the body.

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
