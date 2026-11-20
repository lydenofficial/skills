---
name: master-vault-strategy-audit
description: Audit Lista DAO MasterVault hub-and-spoke yield aggregator — strategy allocation, debt accounting, fees, async unstaking, and strategy migration. Use when auditing Lista MasterVault (contracts/masterVault/MasterVault.sol), its strategies (SnBnb/Bnbx/StkBnb/Ceros), or similar hub-and-spoke vaults.
license: MIT
---

# MasterVault Strategy Audit

Lista DAO's `MasterVault` is a hub-and-spoke yield aggregator: a central vault pools
BNB/ETH, mints receipt tokens (`vaultToken`, e.g. ceToken) 1:1, and allocates funds
across external yield strategies (SnBnb/Synclub, Bnbx/Stader, StkBnb/pSTAKE,
Ceros) through a common `IBaseStrategy` interface. Strategies handle async
unstaking (unbonding queues) and reward harvesting.

Grounding: Lista DAO `lista-dao-contracts` source, read directly —
`contracts/masterVault/MasterVault.sol` (516 lines), `contracts/strategy/BaseStrategy.sol`,
`contracts/strategy/IBaseStrategy.sol`. All function names, structs, and accounting
below are from that source. Verify against the deployed version before reporting —
Lista is upgradeable and the code may have moved on.

## Architecture (from source)

| Element | From source |
|---|---|
| `StrategyParams { active, allocation, debt }` | Per-strategy: active flag, allocation in parts-per-million (1% = 10000), current debt. |
| `totalDebt` | Assets currently borrowed by all strategies. |
| `totalAssetInVault()` | `totalAssets() - feeEarned`, where `totalAssets()` is `address(this).balance`. Idle liquidity = raw ETH balance minus accrued fees. |
| `depositETH()` | Mints `vaultToken` shares = `amount - fee` (1:1 minus `depositFee`); `feeEarned += amount - shares`. `onlyProvider`, `nonReentrant`, `whenNotPaused`. |
| `withdrawETH(account, amount)` | Burns shares, pays idle balance first, then `withdrawFromActiveStrategies` in array order. |
| `_depositToStrategy` | Checks `strategyParams.active` and `strategy.canDeposit(amount)`; `totalDebt += value` and `strategyParams.debt += value` where `value` is what the strategy reports. |
| `_withdrawFromStrategy` | Decrements debt by the **requested `amount`, not the returned `value`** (source comment: the swap fee deducted in the external pool "needs to be paid by the users only"). `require(value <= amount)`. |
| `allocate()` | For each active strategy: `strategyRatio = debt * 1e6 / (totalAssetInVault + totalDebt)`; if below target `allocation`, deposits the difference. `onlyManager`. |
| `setStrategy(strategy, allocation)` | `onlyOwner`; requires `IBaseStrategy(strategy).vault() == address(this)`; total active allocations `<= 1e6`; `strategies.length < MAX_STRATEGIES`. |
| `migrateStrategy(old, new, newAllocation)` | Withdraws all old debt, replaces the strategy in the array, checks total allocations `<= 1e6`. `onlyManager`. |
| `retireStrat(strategy)` | Withdraws `debt` then deactivates; `_deactivateStrategy` treats `debt <= 10` as zero — a **dust write-off** (up to 10 wei of debt is forgiven). |
| Fees | `depositFee` / `withdrawalFee` in ppm, capped by `maxDepositFee` / `maxWithdrawalFee` (both `<= 1e6` at init). `withdrawFee()` sends `feeEarned` to `feeReceiver`. |
| Roles | `owner` (OwnableUpgradeable), `provider` (`onlyProvider` = owner or provider), `manager` mapping (`onlyManager`). `pause`/`unpause` owner-only. |
| `IBaseStrategy` | `deposit()`, `depositAll()`, `withdraw(recipient, amount)`, `withdrawInToken`, `estimateInToken`, `harvest()`, `panic()`, `pause()`/`unpause()`, `balanceOf()`, `balanceOfToken()`, `balanceOfWant()`, `balanceOfPool()`, `setRewards()`, `canDeposit()`, `assessDepositFee()`, `vault()`. |
| `receive() payable` | Vault accepts plain ETH transfers. |

## Accounting invariants (must hold after every call)

1. **Conservation:** `totalAssetInVault() + totalDebt + feeEarned == address(this).balance + Σ strategy-reported holdings` — any drift is a finding. Note `totalAssets()` is the raw balance, so forced ETH (donations, selfdestruct) inflates `totalAssetInVault()` and therefore `allocate()` deposits.
2. **Debt integrity:** `totalDebt == Σ strategyParams[strategy].debt` for all strategies in the array. Any path that changes one without the other breaks accounting.
3. **Debt vs value:** `_withdrawFromStrategy` decrements debt by requested `amount`
   while the strategy may return `value < amount`. Repeated partial withdrawals with
   `value < amount` overstate the reduction:
   debt falls faster than value returns. Track cumulative `(amount - value)` across
   withdrawals: if it grows unboundedly, the vault's books diverge from reality and
   later withdrawals / `allocate()` decisions are wrong.
4. **Allocation sum:** active allocations always `<= 1e6` (100%). Check every path
   that changes allocations (`setStrategy`, `changeStrategyAllocation`,
   `migrateStrategy`) — and check `changeStrategyAllocation` cannot push the sum over
   100% via a stale-then-active sequence.
5. **Share accounting:** shares minted = `amount - fee`; shares burned on withdraw =
   `amount`. Total shares outstanding must track `totalAssetInVault + totalDebt`
   (1:1 minus fees). A path that mints shares without receiving ETH (or burns without
   paying) is a finding.
6. **`availableToWithdraw()`** sums `strategies[i].balance` (native balance of each
   strategy contract) + `totalAssetInVault()` — verify a strategy's native balance is
   actually withdrawable and not e.g. locked in an external protocol (async unstaking).

## Attack classes

### Strategy accounting drift (HIGH)
- The `amount`-vs-`value` debt decrement (invariant 3): a strategy that consistently
  returns less than requested (fees, slippage, rounding, external-protocol haircut)
  makes recorded debt fall faster than real value. Quantify the drift per withdrawal
  and over many withdrawals; if the vault later `allocate()`s based on understated
  debt, it over-deposits into a loss-making strategy.
- `estimateInToken` / `balanceOfToken` / `balanceOf` disagreements: if the strategy's
  view functions disagree with what `withdraw` actually returns, the vault's
  decisions (and users' expectations) are wrong.

### Async unstaking / withdrawal queues
- SnBnb (1h batch), Bnbx (24h batch), StkBnb strategies use async unstaking:
  `withdraw` starts a queue; funds arrive later. Check: can `withdrawETH` complete
  while the underlying unstake is still pending (it pays from idle first, then
  strategies in order — what if a strategy cannot pay yet? does it revert the whole
  withdrawal, or silently underpay? `require(withdrawn <= shares - ethBalance)` then
  `shares = ethBalance + withdrawn` — the user receives less than requested with no
  revert).
- `panic()`: full withdrawal from the external protocol. Check it is callable when
  needed, cannot be griefed, and actually recovers funds (not just marks debt zero).

### Strategy management
- `setStrategy` checks `strategy.vault() == address(this)` — verify a malicious
  strategy cannot fake this (it is a view call on the strategy itself; a malicious
  strategy can return anything). Check what else validates a strategy (audits,
  timelock, allowlist?).
- `migrateStrategy`: `require(withdrawn > 0)` when old debt > 0 — a strategy that
  returns 0 blocks migration (funds stuck in the old strategy). Check the failure mode.
- `retireStrat` dust write-off (`debt <= 10` → debt set to 0): small, but verify it
  cannot be repeated to write off real debt (it can only trigger when debt is already
  ≤ 10 wei).
- `manager` role: `onlyManager` functions (`allocate`, `depositToStrategy`,
  `withdrawFromStrategy`, `migrateStrategy`, `retireStrat`, `withdrawFee`) — a
  compromised manager can drain via `depositToStrategy` into a malicious strategy or
  mis-allocate. Check manager key management and whether a timelock exists.

### Fee manipulation
- `depositFee`/`withdrawalFee` changeable by owner up to `maxDepositFee` /
  `maxWithdrawalFee`. Check the max caps are sane (init requires them `> 0` and
  `<= 1e6` — a max of 100% is allowed by the check!). Front-running a fee increase
  traps deposits; verify timelock / notice.
- `feeEarned` accounting: `totalAssetInVault()` subtracts `feeEarned` from the raw
  balance. If `feeEarned` exceeds the balance (e.g. after a large withdrawal),
  `totalAssetInVault()` returns 0 — check the vault does not brick deposits in that
  state. `withdrawFee()` requires `totalAssets() >= feeEarned` and sets `feeEarned = 0`
  **after** the external call — check reentrancy (it is `nonReentrant`, but verify
  the state update ordering: `feeEarned = 0` happens after the `call` — a failed
  `require(sent)` reverts, so the ordering is safe only because of the revert).

### External protocol risk (per strategy)
- Each strategy wraps a liquid-staking protocol (Synclub, Stader, pSTAKE, Ceros).
  Audit the strategy's `deposit`/`withdraw`/`harvest` against that protocol's
  current behavior: slashing, depeg of the LST (snBNB/BNBx/stkBNB/ceToken), paused
  staking, unbonding-period changes.
- `canDeposit()` gating: if it returns false, `_depositToStrategy` silently does
  nothing (returns false, no revert) — `allocate()` then silently skips the strategy.
  Check silent-skip cannot strand deposits.

### Upgradeability
- Upgradeable (OwnableUpgradeable + Pausable + ReentrancyGuard). Check proxy admin,
  implementation address, and whether upgrades are timelocked — an upgrade can
  replace all accounting.

## Audit workflow

1. Read the deployed `MasterVault` + each strategy from source; diff against the
   repo version (see `bytecode-verification`).
2. Verify invariants 1–6 with unit tests; fuzz deposit/allocate/withdraw/migrate/
   retire sequences with strategy mocks that return `value < amount`.
3. Fork-test each strategy against its live external protocol (stake/unstake/harvest).
4. Check live params on-chain: allocations, debts, fees, caps, roles, paused state.
5. Quantify every finding in BNB/ETH and USD.

## Scope note

Lista MasterVault only — not generic ERC4626 vaults (for those, the ERC4626
inflation pattern in `rate-provider-oracle-audit` and the `evm-audit-erc4626`
checklist apply). For Lista's CDP/lending side use `cdp-liquidation-fuzz`; for
Lista's PSM-like mechanics use `psm-stability-module-audit`.
