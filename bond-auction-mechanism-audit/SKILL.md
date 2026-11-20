---
name: bond-auction-mechanism-audit
description: Audit Olympus-style bond auctions — price discovery, discount/control variable, capacity, vesting, and front-running. Use when auditing bond/bonding mechanisms (Olympus BondDepository/Auctioneer, forks) or hunting Olympus scope.
license: MIT
---

# Bond Auction Mechanism Audit

Olympus-style bonds sell the protocol's token at a **discount** in exchange for
reserve assets, with the purchased tokens **vesting** over time. The discount is
discovered by an auction mechanism: a control variable adjusts the price based on
demand, and each bond market has a capacity and a debt limit. (Context: Olympus
$3.33M Immunefi program, no KYC.)

This skill is mechanism-level: market creation, price discovery, purchase, vesting,
and settlement. Verify exact function and event names against the target's source —
Olympus has shipped several bond systems (v2 BondDepository, v3 Auctioneer +
FixedExpiryTeller + Aggregator) with different interfaces.

## Mechanism

| Element | Role |
|---|---|
| Bond market | A market for one (quote token → payout token) pair: price, capacity, vesting term, min/max purchase. |
| Price discovery | Bond price starts near the market price and is adjusted by a control variable (BCV) in response to demand: heavy buying lowers the discount (price rises toward market), no buying increases the discount. Some designs use a Dutch-auction-style decaying price instead. |
| Discount | `bond price = market price × (1 − discount)`. The discount is the economic core — it must be bounded and must track the market price. |
| Capacity / debt | Each market has a max payout (capacity) and the protocol tracks outstanding bond debt. Capacity exhaustion must close the market cleanly. |
| Vesting | Purchased bonds vest linearly (fixed term or fixed expiry). Payout tokens are claimable over time; unvested balances must be accounted exactly. |
| Teller / Aggregator | Receives quote tokens, issues bond positions, handles claims. May route through an auctioneer for price updates. |

## Invariants to verify

1. **Price integrity:** bond price is always derived from a fresh market price and a
   bounded discount. `0 < discount < maxDiscount`; bond price never exceeds market
   price (no premium bond) and never goes to zero/negative.
2. **Capacity:** total sold in a market never exceeds its capacity; outstanding debt
   never exceeds the protocol's debt ceiling. Capacity checks happen on every purchase
   path, including batched purchases.
3. **Vesting accounting:** `total payout owed == Σ unvested balances`; claimed amounts
   never exceed vested amounts at the claim time; vesting is linear in time (or per
   the documented schedule) with no early-release path.
4. **Quote-token accounting:** quote tokens received == bonds sold × bond price
   (minus fees, if any). No quote token can be lost or double-counted between the
   teller, the auctioneer, and the treasury.
5. **No free bonds:** every bond position corresponds to a paid purchase; no function
   mints bond positions without payment.

## Attack classes

### Market-price manipulation → discount manipulation (HIGH)
- The bond price depends on the market price (oracle, TWAP, or DEX spot). If the
  market price feed is manipulable (flash loan, thin pool, stale TWAP), the attacker
  pushes the market price up, buys bonds at a large discount to the *true* price, and
  profits when the price normalizes.
- **Check:** price source, TWAP window, staleness checks, deviation bounds, and
  whether the bond price uses a spot price that can be moved in one block. Pair with
  `oracle-price-feed-audit`.

### Control-variable / auction gaming
- If the control variable adjusts per purchase, an attacker can buy in many small
  transactions (or one large one, depending on the adjustment rule) to keep the
  discount high, or front-run other buyers to capture the discount before it decays.
- **Check:** adjustment formula, per-purchase vs per-block updates, minimum time
  between adjustments, and whether the adjustment can be gamed by splitting purchases.
- If the price decays over time (Dutch style), check the decay cannot be reset or
  extended by the seller/admin to trap buyers.

### Capacity / debt exhaustion
- An attacker can fill a market's capacity to deny bonds to others, or (worse) a
  market with no debt ceiling can mint unlimited discounted tokens.
- **Check:** capacity enforcement on every path, debt ceiling, what happens when
  capacity is reached mid-transaction (revert vs partial fill — partial fills must
  refund the excess quote token exactly).

### Vesting exploits
- Claim before vesting (timestamp manipulation, wrong cliff math), double-claim
  (claim does not decrement correctly), vesting schedule that can be shortened by
  transferring the position, or a position that can be sold/transferred to bypass
  vesting restrictions.
- **Check:** claim math against block timestamps, transfer restrictions on bond
  positions, and whether vesting is per-position or global (global vesting can let
  a late buyer claim early).

### Front-running
- Bond purchases are public: an attacker seeing a large buy can buy first (capturing
  the better price before the control variable adjusts) or sandwich the victim.
- **Check:** slippage/min-payout parameters on purchase, and whether the protocol
  documents this as accepted MEV or mitigates it.

### Admin / market creation
- Who can create markets, set the initial price/discount/capacity/vesting term?
  A market created with a wrong initial price (e.g. stale market price) can be drained
  in the first block. Check market-creation validation (price sanity vs oracle,
  capacity bounds, vesting bounds) and timelock.
- Market closure: can admin close a market with outstanding bonds? What happens to
  unvested positions and unclaimed payouts?

### Quote-token risks
- Quote token with fee-on-transfer, rebasing, pausing, or blacklist behavior breaks
  the `quote received == price × amount` accounting. Check the token's properties
  (see `web3-audit` token-compatibility patterns).
- Treasury receiving quote tokens: verify the recipient and that funds cannot be
  redirected by a market parameter.

## Audit workflow

1. Map the bond system: markets, price discovery, purchase, vesting, claim,
   treasury flows. Read the deployed source.
2. Verify invariants 1–5 with unit tests; fuzz purchase/claim sequences around
   price changes, capacity limits, and vesting boundaries.
3. Model the market-price-manipulation scenario: cost to move the price source vs
   profit from discounted bonds (fork test with the real price feed).
4. Check live markets on-chain: prices, discounts, capacities, debts, vesting terms.
5. Quantify: maximum extractable discount value per market.

## Validation gate (before reporting)

1. Reproduce on a fork with the deployed bond contracts and a concrete purchase.
2. Quantify profit in USD (discount × purchasable capacity, minus manipulation cost).
3. Confirm the market is live and in scope; a closed/empty market is not a finding.
4. Distinguish intended discount (the protocol *wants* to sell at a discount) from
   manipulation: the finding must show the discount is extractable beyond the
   designed mechanism (e.g. via price-feed manipulation or a control-variable flaw).

## Scope note

Bond auctions only. For launch bonding curves / IEO-style priced sales on BSC use
`launch-curve-audit` (different mechanism: tranche/graduation, not Olympus-style
discount + vesting). For the payout token's own economics (supply, minting) use
`bsc-contract-audit` / `web3-audit`.
