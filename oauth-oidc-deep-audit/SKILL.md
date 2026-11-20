---
name: oauth-oidc-deep-audit
description: >-
  Deep audit of OAuth 2.0 / OpenID Connect integrations: federated assertion verification
  (signature wrapping/stripping, audience/issuer/expiry, exact redirect_uri, state/nonce,
  identity mapping), PKCE and device-code grant proof enforcement (verifier checks, method
  downgrades, brute-forceable user codes), and OAuth token audience/issuer/scope trust
  (cross-audience token confusion, issuer mix-up, per-operation scope enforcement).
  Use when reviewing a login that trusts an external IdP assertion, a token endpoint that
  redeems authorization codes, or a resource server that accepts bearer tokens.
  Source: unboundcompute/security-agent-skills (MIT) — three sub-skills bundled here.
---

# OAuth / OIDC Deep Audit

A federated login moves the trust boundary to a signed assertion from an identity
provider, and the entire security rests on the relying party verifying that assertion
exactly. A bearer token is only good for what it was issued for. And an authorization
code is only safe if the token endpoint demands the proof that binds it to the client
that started the flow. This skill bundles three deep-audit sub-skills — read the one
that matches the surface under review, and cross-load the others when the flow spans
them (it usually does).

## Route first

| Surface under review | Load |
|---|---|
| Login trusts an external assertion/token (SAML or OIDC id token): signature coverage, audience/issuer/expiry, `redirect_uri` exact match, `state`/`nonce` anti-replay, identity mapping (provider-scoped subject, never bare email) | `saml-and-oidc-flows/SKILL.md` |
| Token endpoint redeeming an authorization code, or a device-authorization grant: PKCE verifier enforced, `plain` method / challenge-absent downgrade rejected, verifier bound to the specific code, device/user codes throttled and bound to the requesting client and approver | `device-code-and-pkce-flows/SKILL.md` |
| Resource server accepting bearer access tokens: audience must be itself, issuer must be the trusted authorization server, exact scope per operation (not "any valid token") | `oauth-token-audience-and-scope-trust/SKILL.md` |

## Shared rules (all three sub-skills)

- **The verification step is the audit target.** Everything downstream trusts whatever
  the check accepts. Read the exact verification code, not the docs.
- **Test only systems you own or are authorized to assess**, with test accounts and a
  test identity provider. A confirmed issuance/forgery bypass is account-takeover-grade.
- **Confirm or kill each lead** with a tampered assertion/code/token that still succeeds
  (confirm) or is rejected at verification (kill), and record it per
  `FINDING-SCHEMA.md` (source = attacker-shaped assertion/redirect/token, sink = the
  login or operation it reaches).
- **SIWE (Sign-In with Ethereum) is NOT covered by these three sub-skills.** SIWE
  message verification (domain, nonce, expiry, signature over the exact message) is a
  signature/domain-binding check: treat a SIWE message like an assertion — the
  verification step is the target. Audit it with the same confirm-or-kill discipline
  (tampered domain/nonce/expiry that still verifies = confirm).

## Sources

- `saml-and-oidc-flows/SKILL.md` — `unboundcompute/security-agent-skills@auditing-saml-and-oidc-flows`, MIT.
- `device-code-and-pkce-flows/SKILL.md` — `unboundcompute/security-agent-skills@auditing-device-code-and-pkce-flows`, MIT.
- `oauth-token-audience-and-scope-trust/SKILL.md` — `unboundcompute/security-agent-skills@auditing-oauth-token-audience-and-scope-trust`, MIT.
- `FINDING-SCHEMA.md` — finding record schema shared by the source repo.
- Only change made to the sources: the `FINDING-SCHEMA.md` relative link was fixed
  (`../../FINDING-SCHEMA.md` → `../FINDING-SCHEMA.md`) so it resolves inside this bundle.
