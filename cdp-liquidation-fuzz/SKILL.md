---
name: cdp-liquidation-fuzz
description: Fuzz MakerDAO-style CDP liquidation (Vat/Clip/Dog/Spot, Dutch auctions) with stateful invariant testing (Echidna/Medusa/Foundry). Use when auditing or hunting CDP/lending protocols (Maker, Spark, Lista, forks) where liquidations, auctions, or collateral accounting are in scope.
license: MIT
---

# CDP Liquidation Fuzzing

Stateful invariant fuzzing of a MakerDAO-style CDP system. Goal: prove the liquidation
pipeline can never create bad debt, strand collateral, or pay liquidators incorrectly —
or find the exact input sequence that does.

## System map (verify against target source first)

| Contract | Role |
|---|---|
| `Vat` | Core accounting. `urns(ilk, urn)` = `(ink, art)`; `gem`/`dai` balances; `frob`/`fork`/`grab`/`heal`/`suck`. All internal math in `wad` (1e18), `ray` (1e27), `rad` (1e45). |
| `Jug` | Stability fee. Per-ilk `rate` accumulator; debt = `art * rate`. `drip(ilk)` must be called before accurate debt math. |
| `Spot` | Price feed. `par` (target price, 1 ray), `mat` (liquidation ratio), `poke(ilk)` refreshes `spot` from the oracle. |
| `Dog` | Liquidation trigger. `bark(ilk, urn, keeper)` seizes unsafe urns up to `Hole` (global cap) and per-ilk `hole`; takes `chop` (liquidation penalty). |
| `Clip` | Dutch auction. `take(id, amt, max, who, data)`; price decays from `buf`-padded starting price over `tail` duration; `redo` restarts stale auctions; `cusp`/`tip` control auction start. |
| `DaiJoin` / `GemJoin` | Adapters between Vat-internal balances and ERC-20 tokens. |
| `CdpManager` | Vault ids map to `(ilk, urn)`; user-facing `frob`/`open`/`give`/`quit`. |

Grounding: MakerDAO protocol mechanics (Vat units, Jug drip, Dog → Clipper Dutch
auctions, `clip.take`) per the `maker` protocol reference; liquidation vulnerability
patterns per Decurity CDP / Dacian (see "Vulnerability oracle" below).

## Invariants to fuzz

### Vat accounting (must hold after EVERY call)
1. `sum over urns of art[ilk] * rate[ilk] == total DAI debt` — no debt created or destroyed except via `frob`/`heal`/`suck`/`fold`.
2. `vat.dai` conservation: DAI minted via `frob` equals DAI in `dai` balances + `sin` (system debt) accounting; `suck` increases `sin` exactly by the minted amount.
3. Urn safety: after `frob`, `ink * spot >= art * rate * mat` (or the urn is barkable — never "unsafe but unbarkable").
4. `gem` balances only move via `join`/`exit`/`frob`/`grab` — no silent drains.

### Jug / Spot
5. `rate` is monotonically non-decreasing between `drip` calls; `drip` updates `rate` by exactly the accrued fee.
6. `spot` only changes via `poke`; a stale oracle (no `poke` for N blocks) must not let an unsafe urn look safe — fuzz with oracle freeze / price crash scenarios.
7. `mat` and `par` changes via `file` take effect only for subsequent safety checks.

### Dog (bark)
8. `bark` succeeds exactly when the urn is unsafe (`ink * spot < art * rate * mat`) and `Hole`/`hole` caps are not exceeded.
9. `bark` seizes exactly `art` debt and `ink` collateral per the `chop` penalty; seized collateral + `lot` accounting matches the auction lot.
10. `bark` on a safe urn always reverts; `bark` on an already-barked urn always reverts.

### Clip (Dutch auction)
11. Auction price strictly decreases over time from `buf * spot` and never goes below the `cusp` floor; `take` at time T pays exactly `price(T) * lot`.
12. `take` settles exactly: bidder pays DAI, receives collateral, `owe`/`lot` decrease consistently; partial `take` leaves a consistent remainder (no dust that can never be taken).
13. Outbid/interrupted auctions refund prior bidders in full on every path (`redo`, cancel, `deal`).
14. Auction expiry (`tail`): after `tail`, `take` reverts and `redo` restarts with a fresh price — never a stuck auction holding collateral forever.
15. `deal` (auction settlement) releases exactly the unsold collateral to the right party.

### End-to-end solvency
16. System-wide: `total collateral value (at oracle price) >= total DAI debt + accrued fees` after any sequence that includes liquidations — i.e. liquidations never leave bad debt the system cannot absorb (bound via `Hole`/`chop`/`buf`).
17. No sequence of `frob`/`drip`/`poke`/`bark`/`take` can mint unbacked DAI or withdraw collateral while debt remains.

## Harness

- **Tools:** Foundry invariant tests first (fast iteration), then Echidna or Medusa for
  stateful fuzzing. Generic harness patterns live in `property-based-testing` — this skill
  only adds the CDP-specific state machine and invariants.
- **Actors:** borrower, keeper/liquidator, attacker, governance (`file`/`rely` on params).
- **Stateful sequences (handlers):** `frob` (lock/mint, draw/repay), `drip`, `poke`
  (price up/down, freeze), `bark`, `take` (partial/full, at various times), `redo`,
  `deal`, `file` (mat/chop/hole/buf/tail changes), `join`/`exit`, `heal`/`suck`.
- **Price scenarios:** crash 50%+ in one update (Maker Black Thursday shape), slow bleed,
  oracle freeze then catch-up, depeg of a stable collateral.
- **Ghost state:** track expected `art`/`ink`/`dai`/`sin` per urn off-chain in the harness
  and assert equality after each call — accounting desync is the highest-value finding class.

## Vulnerability oracle (what a counterexample usually means)

Sourced from Decurity CDP and Dacian lending/liquidation writeups (via the
`evm-audit-defi-lending` checklist):
- Self-liquidation profitable (bonus > gas + price impact).
- Partial-auction rounding leaves dust or under-collateralizes the remainder.
- Interrupted-bid funds not returned; auction cancel paths that strand bids.
- Interest accrued after liquidation (stale `rate`) or before close (wrong ordering).
- Paused collateral token blocks repay but not liquidation (unfair liquidation).
- Liquidation before grace period after unpause; no grace period after repay resumption.
- `take`/`bark` callable with wrong collateral token (unchecked `collateralToken` param).
- Single-borrower edge: liquidation loop skips when `count == 1`.
- Fixed liquidation bonus causing revert below the bonus threshold (position stuck).
- Zero-value transfers reverting and blocking liquidation; deny-listed token blocking push-based settlement (use pull).

## Validation gate (before reporting)

1. Reproduce the counterexample on a local fork with concrete numbers (collateral, debt,
   price path, blocks elapsed).
2. Quantify: bad debt in USD, collateral stranded, liquidator over/under-payment.
3. Confirm the finding is reachable from the deployed config (check live `mat`/`chop`/
   `hole`/`buf`/`tail` values, not just defaults).
4. Run the same sequence against the deployed bytecode (not just the repo) when the
   target is a fork with modified params.

## Scope note

This skill is for Maker-style CDPs (Vat/Jug/Spot/Dog/Clip). For generic lending
(Aave/Compound-style health-factor liquidation) use `web3-audit` + the lending
checklist; for liquidation cascade/contagion patterns use the liquidation-cascade
detection patterns. Do not use this skill to re-do generic invariant fuzzing —
`property-based-testing` already covers that.
