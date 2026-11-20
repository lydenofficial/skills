# Single Confirm Auto Revoke

## One confirmation, always

- Require one mandatory confirmation for every wallet activity and never execute without an explicit Ya.

- Treat Ya as execute and Tidak as cancel, and treat silence, a vague reply, or an emoji as no confirmation.

- Show a one-line summary before asking, with chain, from, to, asset, amount, spender if approve, and estimated fee.

- Ask once per activity and do not ask again for the same activity after the user already answered.

## What the single confirmation covers

- Treat one Ya as approval for the activity plus the automatic revoke that follows it when the activity creates an approval, and say so in the summary.

- Cover every wallet activity. The confirmation covers the activity itself.

- The automatic revoke applies only to the activities that create an approval: bridge, swap, buy/sell, paid mint, and listing.

- Stop and do nothing if the user answers Tidak, and report that the activity was cancelled.

- Re-confirm if the chain, recipient, amount, approval scope, or contract address changes, or more than about 30 minutes passed since the last confirm.

## After the activity, revoke automatically

- Execute the activity only after Ya and after simulation passes, following the On-Chain Safety Rails skill.

- List approvals right after the activity finishes with `baw approvals list --json`.

- Revoke every approval tied to that activity, using the same chain, token, and spender, without asking for a second confirmation.

- Revoke with `baw approvals revoke --binanceChainId <id> --tokenContract <token> --spender <spender> --type <approve or permit2> --json`.

- Revoke even if the activity failed, when an approval was created before the failure.

## Check the whole wallet

- Check the whole wallet after every activity, not only the approvals from that activity.

- Flag unlimited amounts, high riskyLevel, unverified spenders, permit2 approvals, and old approvals from bridge, swap, buy-sell, paid mint, or listing.

- Revoke every flagged approval in the same confirmed batch, unless the user explicitly named one to keep.

- Use `baw approvals detail --binanceChainId <id> --tokenContract <token> --spender <spender> --type <type> --json` when a flagged approval needs a closer look before revoking it.

## Standalone danger cleanup

- Run revoke as a standalone task when the wallet looks in danger, even with no new activity.

- Treat a wallet as in danger when it holds an unlimited approval to an unverified spender, a high riskyLevel approval, an approval to a known malicious spender, or any approval that can drain the wallet.

- Ask for one confirmation first for the standalone cleanup, listing every approval to revoke, and run the whole list on one Ya.

- Never run a standalone revoke without that single confirmation.

## Report revoke status

- Treat a revoke txHash as broadcast only, not as confirmed, and say that the approval stays active until on-chain confirmation.

- Tell the user the revoke is pending, that they can track it with `baw wallet tx-history`, and that they should wait before treating the wallet as clean.

- Report each revoked approval by token, chain, and spender, plus its txHash, and report any revoke that failed.

- Stop and wait after a tool or RPC error mid-flow and do not retry a broadcast automatically.

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.
