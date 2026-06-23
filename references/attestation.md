# Reference — Attestation tokens (Private Access Tokens / Privacy Pass)

Some sites do not fingerprint at all. They accept an *attestation token* minted by a trusted issuer —
the Private Access Token / Privacy Pass family (Apple devices, Cloudflare and Fastly as issuers, the
PACT proposal). This is a different shape from everything else, and it changes the whole strategy, so
catch it in **Phase 1 triage** before building any fingerprint machinery.

## Identify it

- The server answers a protected request with an HTTP auth challenge: `WWW-Authenticate: PrivateToken
  challenge=…, token-key=…`, and the client returns `Authorization: PrivateToken token=…`.
- The token is fetched from an *issuer* (a CDN such as Cloudflare/Fastly, or a platform), not computed
  in the page. There is little or no JS sensor, no canvas/WebGL/audio probing, no behavioral beacon.
- On Apple platforms the OS handles it transparently with no UI. It is usually offered as a way to
  *skip a CAPTCHA*, so it sits next to a CAPTCHA/fingerprint fallback rather than replacing it.

## The issuer is the real gate — three cases, cheapest first

A token is only as strong as the issuer's attestation. Decide which case you are in; it sets the whole
strategy:

- **PAT is optional (an accelerator).** The site still accepts the fallback path: solve the CAPTCHA, or
  pass the fingerprint/JS challenge on the credential ladder. Ignore the token and take the fallback;
  confirm with the negative control that the fallback is accepted. → continue the normal pipeline on
  the fallback.
- **The issuer's attestation is software-grade.** A CDN or identity-provider issuer with no device root
  of trust falls back on fingerprinting, account history, or a CAPTCHA to decide whether to sign. The
  real target is then the issuer's own check — recurse: triage *that* step and defeat it on the
  credential ladder. The PAT wrapper adds nothing new.
- **The issuer requires hardware attestation.** Apple's PAT signs only after the Secure Enclave attests
  the device, with an Apple account in good standing involved; Android's analog is hardware-backed key
  attestation / Play Integrity. There is no value to synthesize and no call site to hook. The token is
  produced by a hardware root of trust you do not control.

## When it is hardware-backed, real hardware is the requirement

This is the one branch where piloting a browser does not help, because the attestation is rooted in the
device, not the page.

- **Use a genuine attested device** in good account standing. One real iPhone/Mac (or attested Android)
  mints tokens; a pool of them is the device-farm analog of a browser farm.
- **Scale is expensive and accountable.** Tokens bind to a device and an account the issuer can
  rate-limit, age-check, and revoke. Rotating identity means rotating real, provisioned, account-bound
  devices, not spinning a process. Cost jumps from "a real browser" to "a real, accountable device" —
  the strongest economic bar in anti-bot.
- **Do not try to forge it.** Faking Secure Enclave / Play Integrity attestation is a platform
  root-of-trust exploit, out of scope for recon. If the device root of trust holds, the token cannot be
  produced off-device.

## Where it sits

Attestation tokens are **not a credential tier**. They are a separate branch. The Phase 1 call —
optional vs software-grade vs hardware-backed — is the difference between "no browser needed," "defeat
the issuer's check on the credential ladder," and "real devices needed." Record the case in
`triage.json` (`attestation.case`) so the profile and the orchestrator route correctly.
