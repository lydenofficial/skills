---
name: psm-stability-module-audit
description: Audit MakerDAO-style Peg Stability Modules (PSM / PSM3) — 1:1 mint/redeem, tin/tout fees, debt ceiling, gem joins, depeg exposure, and admin roles. Use when auditing PSM contracts (Maker, Spark PSM3, forks) or hunting stablecoin-peg scope.
license: MIT
---

# PSM / Stability Module Audit

A Peg Stability Module lets anyone mint the stablecoin 1:1 against a reserve asset
(usually USDC) and redeem it back 1:1, minus fees. It is the peg's last line of
defense — and a large, directly-drainable pool of the protocol's most liquid asset.
Spark's PSM3 is the same design deployed on L2s (Base, Arbitrum, Optimism, Unichain).

## Mechanism (verify against target source first)

| Element | Role |
|---|---|
| `buyGem` / `sellGem` (naming varies by deployment) | Mint stablecoin for gem (reserve asset); redeem stablecoin for gem. Fixed 1:1 rate, plus fees. |
| `tin` / `tout` | Fee on mint (tin) and redeem (tout), in `wad`/`ray`. Fees accrue to the protocol (vow / treasury). |
| `line` | Debt ceiling: maximum stablecoin the PSM can mint. Enforced per PSM. |
| `GemJoin` / `DaiJoin` | Adapters moving the gem and the stablecoin in/out of the Vat (or equivalent core accounting). |
| `wards` / `rely` / `deny`, `file` | Admin roles and parameter governance. |
| `cage` / pause | Emergency shutdown. |

No oracle is involved in the 1:1 rate — that is the point (and the risk): the PSM
trusts the gem's peg absolutely.

## Invariants to verify

1. **Conservation:** `gem held by PSM + stablecoin minted via PSM == gem deposited`
   (minus fees, plus fees accrued). Every mint increases PSM debt by exactly the
   minted amount; every redeem decreases it by exactly the redeemed amount.
2. **Debt ceiling:** `line` is enforced on every mint path, including batched and
   indirect paths (e.g. via a router or a wrapper). Minting above `line` must revert.
3. **Fee accounting:** `tin`/`tout` are deducted exactly once, in the right direction,
   and accrue to the right recipient. Fee = 0 must be a valid config, not a bypass.
4. **1:1 rate:** no path lets a user mint more stablecoin per gem than 1 − `tin`, or
   redeem more gem per stablecoin than 1 − `tout`.
5. **Rounding:** mint/redeem rounding favors the protocol, never the caller, and
   cannot be repeated to drain dust.
6. **No double-counting:** the same gem cannot back two mints (e.g. via a stale
   `GemJoin` balance or a reentrant mint).

## Attack classes

### Depeg of the reserve asset (HIGH)
- The PSM holds the gem at 1:1. If the gem depegs (USDC → $0.90), the PSM is
  instantly insolvent by the depeg amount × PSM gem balance — and arbitrageurs will
  redeem the depegged gem for full-value stablecoin until the PSM is empty.
- **Check:** is there a circuit breaker, a `line` reduction, or a pause that triggers
  on depeg? Who triggers it, and how fast? Is there a debt ceiling per gem type?
- **Check:** does the PSM accept multiple gem types at 1:1 without independent
  pricing? Depositing a depegged stable as if it were worth $1 is a direct drain
  (same class as the CDP stablecoin-arbitrage pattern in `cdp-liquidation-fuzz`).

### Debt-ceiling exhaustion (DoS)
- `line` reached = no more mints. An attacker can fill the ceiling with their own
  mints to deny the peg-defense function to everyone else, then profit from the
  resulting depeg elsewhere.
- **Check:** is `line` high enough to matter, can governance raise it quickly, is
  there a per-user cap?

### Fee manipulation
- `tin`/`tout` set by governance. A sudden fee increase front-runs pending redeems;
  a fee decrease (or zeroing) can be exploited by minting at 0 fee and redeeming
  after a fee is introduced — or vice versa.
- **Check:** timelock on fee changes, fee bounds (can `tin` be set to 100%?), and
  whether fee changes apply to in-flight operations.

### Sandwich / mint-redeem arbitrage bounds
- Minting and redeeming in the same block (or across a fee change) must not be
  profitable beyond the intended arbitrage that defends the peg. Fuzz mint/redeem
  sequences around fee and `line` changes.
- Prior work (Spark PSM3 follow-up hunt): sandwich PoC profit was exactly 0 on the
  deployed PSM3 — re-verify per deployment, do not assume.

### Admin / access control
- `wards`/`file`/`cage`: who can change `tin`/`tout`/`line`, pause the module, or
  move the gem? A single admin who can set `line` to 0 (DoS) or `tin` to 100%
  (value extraction) is a centralization finding — check timelock and multisig.
- `GemJoin`/`DaiJoin` wiring: wrong join address = funds sent to the wrong adapter.
  Verify the deployed wiring, not just the repo.

### Pause / cage behavior
- When paused/caged, can users still redeem (exit) or are they locked in? A pause
  that blocks redemption traps users; a pause that blocks minting but not redemption
  is usually the safer design. Check both directions.
- After `cage`, is there a settlement path for remaining gem/stablecoin?

### L2 / PSM3 specifics
- PSM3 on L2s: check bridge messaging for `line`/fee updates (stale params on L2),
  L2 sequencer downtime behavior, and whether each L2 deployment has its own `line`
  (a ceiling exhausted on one L2 must not affect others — or must, if the design says so).
- Verify deployed PSM3 bytecode matches the audited source (see
  `bytecode-verification`) — the Spark follow-up hunt found PSM3 bytecode
  byte-identical to repo HEAD; re-verify for the target deployment.

## Audit workflow

1. Read the PSM + joins + Vat wiring from deployed source; map every mint/redeem path.
2. Verify invariants 1–6 with unit tests, then fuzz mint/redeem/file/pause sequences
   (pair with `property-based-testing` for the harness).
3. Model the depeg scenario: gem at $0.90 / $0.50 — who loses, how much, what stops it.
4. Check live params (`tin`, `tout`, `line`, `wards`) on-chain, not just in the repo.
5. Quantify: maximum drain = min(PSM gem balance, `line` remaining) under each scenario.

## Validation gate (before reporting)

1. Reproduce on a fork with the deployed PSM address and live params.
2. Quantify the drain / loss in USD under a concrete depeg or fee scenario.
3. Confirm the PSM is in scope and the scenario is reachable (e.g. depeg of the
   specific gem the PSM holds, not a hypothetical unrelated asset).
4. A depeg with no circuit breaker is a real finding only if the loss is material
   (PSM balance × depeg) — a dust-size PSM is not a finding.

## Scope note

PSM only — not generic stablecoin minting (see `web3-audit` / `bsc-contract-audit`)
and not CDP liquidation (see `cdp-liquidation-fuzz`). The PSM's defining feature is
the fixed 1:1 rate with no oracle; if the target's stability module uses an oracle
or a floating rate, use `oracle-price-feed-audit` instead.
