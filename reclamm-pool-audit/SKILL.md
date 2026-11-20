---
name: reclamm-pool-audit
description: Audit Balancer ReClamm (AutoRange / reclaiming AMM) pools — virtual balances, centeredness, daily price-shift exponent, price-ratio updates, and proportional-only liquidity. Use when auditing ReClamm pools (Balancer V3, Bush) or hunting Balancer AutoRange scope.
license: MIT
---

# ReClamm Pool Audit

ReClamm (Readjusting / Reclaiming AMM, also called AutoRange) is a two-token Balancer
V3 pool that concentrates liquidity in a price range that **moves by itself**: when
the market price drifts toward the edge of the range, the pool gradually shifts the
range back around the current price. Every LP holds the same fungible BPT.

Grounding: Balancer docs (pool maths reference, ReClamm pool page), Bush docs
(ReClamm pool page), Balancer `reclamm` repo (`ReClammPool.sol`,
`ReClammPoolFactory.sol`), Immunefi Balancer resources (AutoRange/ReClamm in scope).

## Mechanism

- **Virtual balances:** the pool tracks a virtual balance per token — an offset added
  to the real balance. Swaps are priced on the constant-product invariant applied to
  the combined (real + virtual) balances. The spot price of token A in terms of B is
  derived from the virtual balances; the virtual balances define the edges of the
  price range. When the real balance of one token reaches zero, the pool has hit an
  edge of its range.
- **Centeredness:** measures how close the pool is to the middle of its range — 1
  (100%) at the center, falling toward 0 as the price approaches either edge.
- **Target range:** the `centerednessMargin` marks out a target range within the full
  price range. While centeredness is at or above the margin, the pool is **in range**
  and virtual balances stay fixed.
- **Out of range:** when centeredness falls below the margin, the virtual balance of
  the overvalued token starts to decay every second, shifting the range toward the
  market price. Speed is set by `dailyPriceShiftExponent`.
- **Price ratio:** the width of the range (`currentPriceRatio`) can be changed
  gradually via `startPriceRatioUpdate` (target ratio + start/end time, interpolated),
  keeping the range positioned around the current price at the same centeredness.
- **Liquidity:** proportional-only. Unbalanced adds/removes and donations are disabled.
- The pool contract acts as **its own hook** (no separate hook is configured).

## Parameters (verify every one)

| Param | Meaning / range | Check |
|---|---|---|
| `initialMinPrice` | Lower edge of initial range | `> 0`; `min < max`; range width sane for the pair's volatility |
| `initialMaxPrice` | Upper edge of initial range | `> initialMinPrice` |
| `initialTargetPrice` | Starting price | Within the target range defined by the margin; matches market at deployment |
| `dailyPriceShiftExponent` | Speed of range adjustment, 0%–500% | Max ~50% moves the range ~1.41×/day. Admin-changeable — check who can change it and whether a sudden max-speed change can be abused. |
| `centerednessMargin` | Width of target range, 0%–50% | Margin 0 = pool always in range = range never moves (equivalent to a 2-CLP with the same range). Higher = more sensitive to imbalance. Admin-changeable. |
| `tokenAPriceIncludesRate` / `tokenBPriceIncludesRate` | Whether token prices include rate-provider rates | Wrong flag = mispriced pool. Cross-check with `rate-provider-oracle-audit`. |

## Audit checklist

### Virtual balances & pricing
1. Recompute spot price from virtual + real balances independently; verify against the
   pool's quoted price, including at the range edges.
2. Virtual-balance decay: verify the per-second decay math and that it only applies
   while out of range and only to the overvalued token's virtual balance.
3. At the edge (one real balance → 0): swaps in the direction that would push past
   the edge must revert or return zero — never execute at a nonsense price.
4. Donations: direct token transfers to the pool must not move the price (donations
   are disabled by design — verify the pool actually ignores/blocks them rather than
   counting them as liquidity).

### Centeredness
5. Centeredness formula: verify it is 1 at the center and → 0 at the edges, and that
   the margin comparison uses the right side (in-range vs out-of-range).
6. Boundary behavior: centeredness exactly at the margin — in or out? A fence-post
   error here makes the range shift when it should not (or vice versa).
7. A swap that moves the pool toward a more unbalanced state must be priced with the
   shift the margin implies — check that large swaps cannot jump the pool across the
   margin in one step without the decay applying.

### Price-shift manipulation
8. `dailyPriceShiftExponent` at max (500%): simulate a sustained one-sided market —
   how fast can an attacker drag the range? Is there a per-block or per-second cap
   beyond the daily exponent?
9. Admin changes to exponent/margin: front-running a margin decrease (range stops
   moving) or exponent increase (range moves fast) — check timelock / access control
   on `setDailyPriceShiftExponent` / `setCenterednessMargin`.
10. Price-ratio updates (`startPriceRatioUpdate`): verify gradual interpolation,
    start/end time validation (end > start, duration sane), and that narrowing the
    range cannot strand LPs or let an admin squeeze the price. Who can call it —
    swap fee manager or governance?

### Liquidity operations
11. Proportional-only enforcement: unbalanced add/remove and single-token operations
    must revert on every entry point (including via the Vault/router and via hooks).
12. BPT mint/burn math against virtual balances: adding liquidity at a shifted range
    must mint the correct BPT; removing must return the correct pro-rata amounts.
13. Invariant-ratio bounds on liquidity ops (same family as other V3 pools) — verify
    enforced for ReClamm.

### Hook behavior (pool is its own hook)
14. `onBefore/AfterSwap` and liquidity hooks: verify they enforce the ReClamm rules
    (centeredness checks, range updates) and cannot be bypassed by calling the Vault
    directly.
15. Hook reentrancy: the pool-as-hook must not allow reentrant state changes mid-swap.

### Rate providers
16. If either token is yield-bearing, `tokenXPriceIncludesRate` must match reality;
    a wrong flag misprices every swap. Pair with `rate-provider-oracle-audit`.

## Attack scenarios to test on a fork

- Sustained one-sided selling to drag the range, then a reversal — does the range
  chase the market in a way that lets an attacker buy back cheaper (range-chasing
  arbitrage against LPs)?
- Margin/exponent admin change front-run: change params, then immediately trade.
- Edge-of-range swap: push the pool to an edge and check pricing/slippage behavior.
- Price-ratio narrowing during high volatility: can the range be narrowed onto a
  manipulated price?
- Donation to the pool: confirm no price or BPT effect.

## Validation gate (before reporting)

1. Reproduce on a fork against the deployed `ReClammPoolFactory` pool with concrete
   amounts and elapsed time (range shifts are time-dependent — simulate blocks).
2. Quantify LP loss / attacker profit in USD.
3. Confirm the pool and its admin (fee manager / governance) are in scope.
4. Distinguish intended behavior (the range is *supposed* to move) from a bug: a
   finding must show the movement is exploitable beyond the designed mechanism.

## Scope note

ReClamm only. For the underlying Vault accounting use `balancer-v3-invariant-fuzzing`;
for Gyro pools use `gyro-eclp-math-audit`; for rate providers use
`rate-provider-oracle-audit`. Do not confuse ReClamm with Gyro 2-CLP: a zero
centeredness margin makes ReClamm behave like a 2-CLP, but the contracts, params,
and admin surfaces are different.
