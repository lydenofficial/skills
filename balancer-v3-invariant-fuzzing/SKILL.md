---
name: balancer-v3-invariant-fuzzing
description: Stateful invariant fuzzing of Balancer V3 — single Vault accounting, pool math, hooks, ERC4626 buffers, and BPT — with Echidna/Medusa/Foundry. Use when hunting Balancer V3 scope (Vault, pools, hooks, buffers) or auditing any Balancer V3 fork.
license: MIT
---

# Balancer V3 Invariant Fuzzing

Balancer V3 is built around a **single Vault that holds all pool assets** and enforces
core accounting and safety checks; pool contracts are small and custom pool types are
simpler to build and review. V3 adds a hooks framework, native ERC4626 boosted
liquidity through internal buffers, and pool types including Weighted, Stable,
StableSurge, Gyro (E-CLP and 2-CLP), AutoRange (ReClamm), and Liquidity Bootstrapping
Pools. (Per Immunefi Balancer program resources.)

This skill is the V3-specific invariant list and harness. Generic fuzzing mechanics
live in `property-based-testing` — do not duplicate them here.

## Architecture map (verify against target source first)

| Component | Role |
|---|---|
| `Vault` | Holds ALL tokens. Transient storage tracks per-operation balance deltas; every operation must settle (deltas zero at end of call). Enforces pool registration, pause, recovery mode. |
| `Router` / `BatchRouter` | User entry points; delegate to Vault. |
| Pools (`Weighted`, `Stable`, `GyroECLP`, `Gyro2CLP`, `ReClamm`, `StableSurge`, `LBP`) | Implement `computeInvariant`, `computeBalance` (swap math), `onAddLiquidity`/`onRemoveLiquidity`. Hold no tokens themselves. |
| `Hooks` (`IHooks`) | `onBefore/AfterSwap`, `onBefore/AfterAddLiquidity`, `onBefore/AfterRemoveLiquidity`, dynamic swap fee, `onComputeDynamicSwapFeePercentage`. Hooks can change fees and (some) amounts — but must never break Vault accounting. |
| `Buffers` | ERC4626 wraps: `wrap`/`unwrap` yield-bearing tokens into the Vault; buffer shares track wrapped vs underlying. Yield fees accrue to the protocol. |
| `ProtocolFeeController` | Swap/yield fee percentages, fee recipients. |
| `BPT` | Pool tokens minted/burned against the pool invariant. |

## Vault-level invariants (must hold after EVERY settled call)

1. **Token conservation:** `vault.balance(token) == Σ pool balances(token) + Σ buffer balances(token) + protocol fees owed(token)` for every token. Any drift is a finding.
2. **Transient settlement:** all transient balance deltas are zero at the end of every
   external call — no operation leaves the Vault mid-settlement (reentrancy surface).
3. **No token leaves without accounting:** every token outflow is matched by a BPT
   burn, a swap output, a buffer unwrap, or a fee payment — never by a bare transfer.
4. **Pool balances only change through registered pools:** no direct manipulation of
   a pool's recorded balances except via Vault-mediated operations.
5. **Pause / recovery mode:** when paused, all state-changing pool ops revert; recovery
   mode allows only proportional exits (no swaps, no unbalanced ops).

## Pool invariants (per pool type — fuzz each)

6. **Invariant monotonicity:** swaps never change the pool invariant except by the
   swap fee accrual direction (invariant must not decrease on a swap; must not
   increase except by fees). Adds/removes move the invariant proportionally to the
   BPT minted/burned.
7. **BPT accounting:** `totalSupply(BPT)` tracks the invariant: minting BPT without a
   corresponding invariant increase (donation!) or burning BPT without a decrease is
   a finding. First-deposit / minimum-BPT behavior must prevent share-price inflation.
8. **Swap output bounds:** `amountOut` is exactly the pool-math result minus fee —
   never more (pool drained) and never less than the quoted amount (user loss).
   Exact-in and exact-out paths must agree.
9. **Rounding favors the pool:** every rounding decision in swap/add/remove math
   rounds in the pool's favor (or is explicitly bounded). A rounding that favors the
   caller, repeated over many calls, is a value-extraction bug.
10. **Unbalanced / single-token ops:** invariant-ratio bounds enforced (e.g. Gyro
    60%/500% limits); unbalanced ops cannot shift the price for free.

## Hook invariants

11. A hook can change the dynamic swap fee and (where allowed) amounts — but the
    Vault-level token conservation invariant must still hold after the hook runs.
12. Hook callbacks cannot reenter the Vault to change the same operation's outcome
    (reentrancy through `onBefore/After*`).
13. Dynamic fee is bounded by the pool's declared fee bounds on every swap.
14. A malicious or buggy hook must not be able to steal funds from *other* pools
    (hook scope is per-pool).

## Buffer invariants (ERC4626 boosted liquidity)

15. Buffer shares: `wrapped balance == shares * rate` at all times; wrapping/unwrapping
    conserves value (minus yield fee).
16. Buffer rate comes from the rate provider — if the rate is manipulable, the buffer
    is manipulable. Pair with `rate-provider-oracle-audit`.
17. Yield fees accrue exactly per the configured percentage; no fee on principal.
18. Buffers cannot be drained below the shares owed (no donation-based share inflation;
    same class as ERC4626 inflation — see `rate-provider-oracle-audit`).

## Harness

- **Tools:** Foundry invariant tests for speed, then Echidna/Medusa for stateful
  coverage. Reuse the harness patterns from `property-based-testing`.
- **Actors:** LP, swapper, arbitrageur, hook admin, fee manager, attacker (donations,
  tiny amounts, rounding games).
- **Handlers (stateful sequences):** `addLiquidity` (proportional / unbalanced /
  single-token), `removeLiquidity` (same three), `swap` (exact-in / exact-out),
  `donate` (direct token transfer to Vault — must be a no-op for accounting),
  `wrap`/`unwrap` buffers, `setSwapFee` (hook / admin), `pause`/`unpause`,
  `setYieldFee`, `collectFees`, `updatePriceRatio` (ReClamm), `createPool`.
- **Ghost state:** off-chain mirror of every pool balance, BPT supply, and buffer
  share count; assert equality with on-chain values after each call.
- **Adversarial sequences:** donate-then-deposit, swap-dust loops, add/remove in the
  same block, hook fee flips mid-operation, pause during an operation, recovery-mode
  exits, buffer wrap/unwrap around a rate change.

## Vulnerability oracle (what a counterexample usually means)

- Donation breaks BPT pricing (first-deposit inflation) → check pool-specific
  minimum-BPT / virtual-share defenses.
- Rounding favors the caller on any path → quantify per-call extraction × plausible
  call count.
- Hook changes amounts without Vault re-validation → hook can mint value.
- Buffer rate manipulation → pool value manipulation (rate provider attack).
- Transient-storage desync → reentrancy or settlement failure.
- Recovery-mode bypass → swaps or unbalanced exits while in recovery.
- Fee accounting drift → protocol fees over/under-collected; LPs diluted.

## Validation gate (before reporting)

1. Reproduce on a local fork against the deployed Vault + pool addresses with concrete
   amounts and block numbers.
2. Quantify extracted value in USD at realistic sizes (dust-only extraction that
   cannot scale is not a finding).
3. Confirm the pool/hook/buffer is in scope for the program (Balancer V3 vs V2,
   deployed factory, hook address).
4. Cross-check the same sequence against the reference pool maths (Balancer docs)
   to rule out harness bugs.

## Scope note

V3 only. For V2 (separate Vault per deployment, different accounting) do not reuse
these invariants blindly. For pool-type-specific math use `gyro-eclp-math-audit`
(E-CLP) and `reclamm-pool-audit` (ReClamm); for rate providers use
`rate-provider-oracle-audit`; for formal proofs of the same properties use
`certora-formal-verification-review`.
