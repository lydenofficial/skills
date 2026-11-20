---
name: gyro-eclp-math-audit
description: Audit Gyro E-CLP (elliptic concentrated liquidity pool) math on Balancer — ellipse invariant, derived parameters, swap quadratic, rounding, and liquidity bounds. Use when auditing Gyro E-CLP pools (Balancer V3 GyroECLPPool / GyroECLPMath) or hunting Balancer Gyro scope.
license: MIT
---

# Gyro E-CLP Math Audit

E-CLPs trade along a section of an **ellipse**. The invariant is the minor radius
(semi-minor axis) of the ellipse, obtained by transforming a circle with stretch,
rotation, and displacement. Two tokens only; liquidity is concentrated asymmetrically
inside the price bounds `[alpha, beta]`.

Grounding: Balancer docs (Gyro E-CLP pool page, pool maths reference), Gyroscope
docs (E-CLPs, `GyroECLPMath.sol` function table), Certora Balancer V3 audit
2026-01-26 (Gyro pool section, incl. finding I-01).

## Parameters (verify every one)

Immutable (set at creation, from off-chain computation at 38-decimal precision):

| Param | Meaning | Check |
|---|---|---|
| `paramsAlpha` (α) | Lower price bound | `0 < α < β`; sqrt values stored as immutables |
| `paramsBeta` (β) | Upper price bound | `β > α`; bounds actually enforced on swaps |
| `paramsC` (c) | `cos(-φ)` | `c² + s² == 1` (within fixed-point tolerance) |
| `paramsS` (s) | `sin(-φ)` | same; sign convention matches rotation direction |
| `paramsLambda` (λ) | Stretch, `λ >= 1` | **Lower bound must be validated** — Certora I-01: missing lower-bound validation for the E-CLP stretch factor. `λ < 1` inverts the stretch and breaks the invariant geometry. `λ == 1` is a perfect circle (price-bounded StableSwap). |
| `tauAlphaX/Y`, `tauBetaX/Y` | Derived tangent points at α/β | Recompute off-chain from (α, β, c, s, λ); must match deployed values exactly (they are trusted inputs — a wrong derived param silently warps the curve). |
| `u, v, w, z, dSq` | Derived auxiliary values | Same: recompute off-chain and compare; any mismatch is a finding. |

## The invariant

Conceptually `r = g(t)` where `t = (x, y)` are real reserves and
`g = Str(1/λ) ∘ Rot(-φ) ∘ v` — shift reserves by virtual offsets `v`, rotate by `-φ`,
stretch by `1/λ`. The result is the ellipse's minor radius `r`.

Checks:
1. `calculateInvariant` recomputed independently (Python/TS reference maths from the
   Balancer docs) matches the on-chain value for random reserves — including at the
   price bounds and at the center.
2. Invariant is **non-decreasing** on liquidity adds and **non-increasing** on removes,
   within the enforced ratio bounds (see below). Any swap must leave `r` unchanged
   (up to the allowed rounding direction) — a swap that increases `r` leaks value to
   the swapper; one that decreases it leaks value to LPs.
3. Invariant never goes negative / underflows at extreme reserves (near-zero balance
   of one token at the price edge).

## Swap math

Swaps are computed by solving a **quadratic in transformed coordinates**
(ellipse → circle via rotation + stretch), then mapping back:
- `calcOutGivenIn` / `calcInGivenOut` — verify against the reference implementation
  for random inputs; pay attention to which root of the quadratic is selected.
- The implementation **intentionally overestimates** the post-swap invariant / output
  in a direction that favors the pool. Verify the overestimation direction is correct
  in every branch — a flipped rounding direction is a direct value-extraction bug.
- `calcXGivenY` — used by both swap directions; errors here affect everything.
- 18-decimal fixed-point throughout; check for precision loss at small reserves and
  at large `λ` (high stretch = steep curve = amplified rounding error).
- Price bounds: swaps that would push the price outside `[α, β]` must revert or be
  clamped per spec — never silently execute outside the range.

## Liquidity operations

- `liquidityInvariantUpdate` — balanced adds/removes update the invariant without a
  full recomputation. Verify the fast path equals the full `calculateInvariant` path.
- **Invariant ratio bounds:** on liquidity operations the invariant cannot decrease
  below 60% or increase beyond 500% (same limits as the Stable Pool,
  `IUnbalancedLiquidityInvariantRatioBounds`). Check the bounds are actually enforced
  on every liquidity entry point, including unbalanced and single-token operations.
- **Swap fee bounds:** minimum swap fee 0.000001% (an order of magnitude below the
  Stable Pool minimum); maximum effectively unconstrained (100%). Verify fee is
  applied to the correct amount and cannot be set to a value that breaks the math.
- Proportional vs unbalanced adds: unbalanced adds must not let a depositor shift the
  price for free (check invariant-ratio enforcement on the unbalanced path).

## Rounding / precision attack surface

- Newton-iteration / quadratic-solver convergence: verify iteration count and
  termination conditions; non-convergence must revert, never return a stale value.
- Derived params are computed off-chain at 38 decimals then stored on-chain — check
  truncation when down-scaling to 18 decimals cannot move the curve enough to matter
  (compare on-chain vs 38-decimal reference at the bounds).
- Small-pool / first-deposit edge: tiny `r` amplifies relative rounding error;
  check minimum-liquidity / first-deposit behavior.

## Known findings to re-check (do not assume fixed)

- Certora 2026-01-26 I-01: missing lower-bound validation for the E-CLP stretch
  factor (`λ`). Confirm the deployed factory/pool validates `λ >= 1`.
- Any pool whose derived params (`tau*`, `u/v/w/z`, `dSq`) do not match an independent
  off-chain recomputation — treat as a critical finding (wrong curve = wrong prices).

## Audit workflow

1. Pull immutable + dynamic data via `getGyroECLPPoolImmutableData` /
   `getGyroECLPPoolDynamicData`; recompute all derived params off-chain.
2. Differential-test `calculateInvariant`, `calcOutGivenIn`, `calcInGivenOut` against
   the reference maths (TS/Python refs linked from Balancer docs) over randomized
   reserves, including edges.
3. Fuzz swap sequences for invariant monotonicity and for price-bound enforcement;
   pair with `balancer-v3-invariant-fuzzing` for Vault-level invariants.
4. Verify fee/ratio bounds on every entry point; check the factory's parameter
   validation (especially `λ` lower bound).

## Scope note

This skill is E-CLP only. For 2-CLP (hyperbola invariant `(x+a)(y+b) = L²`) use the
same differential-testing approach but the 2-CLP formulas, not the ellipse ones.
For Vault-level concerns (accounting, hooks, buffers) use
`balancer-v3-invariant-fuzzing`. For rate-provider manipulation of tokens inside
an E-CLP, use `rate-provider-oracle-audit`.
