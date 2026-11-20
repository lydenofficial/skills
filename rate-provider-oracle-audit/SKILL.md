---
name: rate-provider-oracle-audit
description: Audit Balancer rate providers and yield-bearing-token exchange rates — getRate() manipulation, donation/inflation attacks, staleness, upgradeability, and registry review. Use when auditing Balancer pools with rate providers, ERC4626-based rate feeds, or any getRate()-style exchange-rate oracle.
license: MIT
---

# Rate Provider / Oracle Audit

Balancer pools price yield-bearing tokens through **rate providers**: contracts whose
`getRate()` returns the exchange rate between the yield-bearing token and its
underlying (e.g. wstETH → ETH, an ERC4626 vault share → asset). The pool trusts that
rate completely — if `getRate()` can be manipulated, the pool's pricing is manipulated.
Balancer maintains a rate-provider review process and registry; only reviewed providers
should be trusted, and every provider must be re-checked at audit time.

This skill is rate-provider-specific. For generic price feeds (Chainlink/TWAP/Pyth,
heartbeat, sequencer uptime) use `oracle-price-feed-audit`; for ERC4626 inflation in
general use the donation-attack pattern below.

## How rate providers are used

- Stable/Weighted pools with yield-bearing tokens call `getRate()` to convert balances
  to underlying value before applying the invariant.
- Buffers (ERC4626 boosted liquidity) use the rate to convert wrapped ↔ underlying.
- A wrong rate by X% misprices every swap in the pool by ~X% — the attacker's profit
  is bounded by pool TVL × rate error, minus slippage/fees.

## Attack class 1 — Donation / share-price inflation (CRITICAL)

The classic ERC4626 inflation attack, applied to the rate:
- Vault with naive `convertToShares` math: first depositor mints 1 share for 1 asset,
  attacker donates a large amount directly to the vault (bypassing deposit),
  `totalAssets` balloons, share price inflates massively, next depositor's shares round
  to 0 — or, for a rate provider, `getRate()` spikes and the pool overvalues the token.
- **Check:** does `getRate()` (or the underlying `convertToAssets`) depend on
  `balanceOf(vault)` (donatable) or on internally tracked assets (safe)?
- **Defenses to verify:** virtual shares / `_decimalsOffset()` (OZ ERC4626 ≥ 4.9, offset
  typically 6), dead shares minted to `address(0)` on first deposit, minimum first
  deposit, admin-seeded vault, internal accounting instead of raw balance.
- **PoC:** on a fork, donate to the underlying vault and read `getRate()` before/after;
  then simulate a swap against the pool at the inflated rate and quantify profit.

## Attack class 2 — Stale / never-updated rate

- Rate provider that caches a rate and only updates on someone calling `update()`:
  if nobody updates, the pool trades on a stale rate while the true rate moved.
- **Check:** update cadence, who can update, is there a heartbeat / max-age check in
  the pool (there usually is not — the pool trusts the provider), what happens if the
  underlying yield accrues faster than updates.
- A rate that only ever increases (yield accrual) but is read stale lets an attacker
  buy the token cheap in the pool and redeem at the true (higher) rate elsewhere.

## Attack class 3 — Rate decrease / negative rebase

- Some yield tokens can lose value (slashing, negative rebase, bad debt). If the
  provider's rate can decrease, check the pool handles it: invariant math, BPT pricing,
  and buffer accounting must not assume monotonic increase.
- **Check:** is a rate decrease bounded, paused, or does it silently reprice all LPs down?

## Attack class 4 — Upgradeable / admin-controlled provider

- If the rate provider is upgradeable or its rate is admin-settable, the admin (or a
  compromised admin key) can set an arbitrary rate and drain the pool.
- **Check:** proxy admin, `owner`/`wards`, timelock on rate changes, whether the
  provider is on Balancer's reviewed registry, and whether the pool has any bound on
  rate movement per update.

## Attack class 5 — Rate computed from manipulable inputs

- Rate derived from a DEX spot price, a single oracle, or an instantaneous balance
  ratio is manipulable (flash loan, sandwich, oracle wick).
- **Check:** every input to `getRate()` — balances, oracle feeds, TWAP windows,
  decimals handling. A rate that mixes 6- and 18-decimal tokens without scaling is a
  finding. Cross-check with `oracle-price-feed-audit` patterns for the feed parts.

## Attack class 6 — Rounding and precision

- `getRate()` rounding direction: must round in the pool's favor (or be bounded).
  Repeated small extractions via rounding add up.
- Rate precision: a rate stored with too few decimals (e.g. 6) loses precision for
  large supplies; check the rate's decimal handling against the token's decimals.

## Review checklist

1. Read `getRate()` end to end. List every state variable and external call it touches.
2. Identify the underlying: ERC4626 vault? staking contract? rebasing token? Is the
   underlying's accounting donation-proof?
3. Simulate a donation to the underlying on a fork; measure `getRate()` movement.
4. Check registry status: is this provider Balancer-reviewed? (Unreviewed provider =
   higher scrutiny, not an automatic finding — prove manipulability.)
5. Check update mechanics: who updates, how often, max staleness, pause behavior.
6. Check admin/upgradeability: who can change the rate, is there a timelock, is there
   a per-update movement bound.
7. Check decimals and rounding in the rate and in every pool that consumes it.
8. Check the pool's behavior when the rate moves sharply (up and down) — swaps, adds,
   removes, buffer wraps.
9. Quantify: max extractable value = TVL × achievable rate error, minus fees/slippage.
   A manipulable rate that cannot move enough to cover fees is not a finding.

## Validation gate (before reporting)

1. Demonstrate the rate movement on a fork with a concrete transaction (donation amount,
   `getRate()` before/after).
2. Demonstrate the profit: a swap (or add/remove) against the real pool at the
   manipulated rate that extracts value.
3. Confirm the provider and pool are in scope (deployed addresses, registry status).
4. Rule out defenses: virtual shares, dead shares, min deposit, internal accounting,
   rate bounds — verify each claimed defense actually holds in the deployed code.

## Scope note

Rate providers only — not generic oracles (see `oracle-price-feed-audit`) and not
pool invariant math (see `gyro-eclp-math-audit`, `reclamm-pool-audit`,
`balancer-v3-invariant-fuzzing`). The donation/inflation pattern here is the
rate-provider application of the ERC4626 inflation class; if the target is a plain
ERC4626 vault with no rate-provider role, the same pattern applies but report it as
a vault inflation finding, not a rate-provider finding.
