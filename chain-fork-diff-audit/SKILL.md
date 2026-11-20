# Chain Fork Diff Audit

Audit a blockchain client fork by diffing it against its upstream — the fastest way to find
consensus-breaking, fund-loss, and security-patch-removal bugs in chain forks (Cosmos SDK chains,
go-ethereum forks, Cronos, Sei, BSC-style clients). A fork's most dangerous bugs are rarely in its
new code; they hide in what it changed, removed, or failed to backport from upstream.

## When to Use

- Auditing a Cosmos SDK fork (Sei, Cronos, Osmosis, any Cosmos chain): diff vs `cosmos/cosmos-sdk`
  (and `cosmos/ibc-go`, `cometbft/cometbft`) at the pinned version.
- Auditing a go-ethereum fork (BSC, Cronos EVM layer, any EVM chain client): diff vs
  `ethereum/go-ethereum` at the pinned version.
- Any chain whose bug bounty scope is its node client / forked codebase.
- After a chain upgrade: diff the new release against the previous release to find the delta.

## When NOT to Use

- Pure Solidity / smart-contract audits — use `web3-audit`, `slither-semgrep-solidity`.
- Generic Go code review with no upstream to compare — use `go-security-review`.
- Cosmos module logic with no fork relationship — use `cosmos-vulnerability-scanner`.

## Workflow

### Phase 1: Pin the upstream and the fork point

1. Find the upstream version the fork is based on: `go.mod` (`require github.com/cosmos/cosmos-sdk vX.Y.Z`,
   `require github.com/ethereum/go-ethereum vX.Y.Z`), release notes, or `git log` for the fork point.
2. Clone upstream at exactly that tag/commit. If the fork does not declare a version, bisect: find the
   newest upstream commit whose tree matches the fork's vendored/unchanged files.
3. Record: `FORK_REPO`, `FORK_COMMIT`, `UPSTREAM_REPO`, `UPSTREAM_TAG/COMMIT`. If the fork point
   cannot be pinned, say so explicitly — an unpinned diff is unreliable.

### Phase 2: Produce the diff

```bash
git -C <fork> remote add upstream <upstream-url>   # read-only, local clone only
git -C <fork> fetch upstream <upstream-tag> --depth 1
git -C <fork> diff <upstream-tag>...HEAD --stat    # size of the delta first
git -C <fork> diff <upstream-tag>...HEAD -- <paths> # then per-area
```

- Never fetch or build from an untrusted remote without inspecting it first; cloning public
  upstream repos is read-only source review.
- Exclude generated files, vendored deps, docs, and test-only churn from the security diff
  (note the exclusion, do not silently drop it).

### Phase 3: Triage the delta by risk area

Classify every changed hunk into one of these buckets, highest risk first:

| Bucket | What to look for | Why it matters |
|---|---|---|
| **Consensus-critical** | BeginBlock/EndBlock/FinalizeBlock, block proposal/validation, state machine transitions, ABCI handlers, fork-choice / chain rules | A divergence here = chain halt or consensus split |
| **Cryptography** | Signature verification, key derivation, hashing, address derivation, randomness | Weakened crypto = forgery, fund theft |
| **Removed / weakened checks** | Deleted `require`/`if err != nil`, relaxed validation, dropped allowlists, removed slashing / penalty logic | Classic fork bug: security control silently dropped |
| **Changed constants / params** | Block gas limit, timeouts, fee params, staking/unbonding periods, governance thresholds | Economic + liveness impact |
| **Networking / P2P** | Peer scoring, mempool admission, gossip, DoS guards | Eclipse / mempool DoS / spam |
| **RPC / API surface** | New or changed JSON-RPC / gRPC methods, removed auth, new admin endpoints | Exposed dangerous methods |
| **State / storage** | Changed store keys, pruning, migration logic, encoding changes | State divergence, failed upgrades |
| **New features** | Custom modules, precompiles, EVM extensions | New attack surface; audit as new code |

### Phase 4: Check for missing upstream security patches (the highest-yield step)

1. List upstream security advisories / releases AFTER the fork point:
   - Cosmos SDK: `cosmos/cosmos-sdk` security advisories + CHANGELOG security sections.
   - go-ethereum: `ethereum/go-ethereum` security advisories (GHSA) + release notes.
2. For each upstream fix, check whether the fork contains it: search the fork for the fixed
   function / patched pattern. A fix that is absent = the vulnerability is LIVE in the fork.
3. Pay special attention to fixes in files the fork modified — a merge conflict resolved
   incorrectly can silently revert a security patch.
4. Cross-check known CVE/GHSA lists for the upstream version range the fork spans.

### Phase 5: Verify the fork's claims (deployment-vs-code)

- Confirm the running binary / deployed chain actually matches the fork source you audited:
  use `bytecode-verification` for on-chain contracts, and compare the node's reported version /
  commit hash (`version` command, `/status` RPC) against the audited commit.
- If the deployed version differs from the audited source, the diff audit does not apply to it —
  re-pin and re-diff.

### Phase 6: Validate each candidate finding

- Every candidate must pass the 8-Question Gate (`triage-validation`) before reporting.
- Prove reachability: a changed line in dead code or behind an unreachable flag is not a finding.
- Where possible, write a PoC: unit test on the fork, or a local testnet / devnet reproduction.
  For consensus bugs, a two-node local network showing divergent state roots is decisive proof.
- Severity guide: consensus split / chain halt = High–Critical; live upstream CVE unpatched =
  severity of the original CVE (usually High–Critical); weakened validation with fund impact =
  per funds at risk.

## Cosmos-Specific Notes

- Forks often pin an OLD SDK version while cherry-picking features — check `go.mod` for the SDK,
  `ibc-go`, and CometBFT versions separately; each can lag independently.
- Check `app.go` wiring: modules added/removed, custom AnteHandler, custom `BeginBlocker` ordering.
- IBC: forks that touch `x/ibc` or packet-forward-middleware are high-risk; use
  `cosmos-vulnerability-scanner` IBC patterns (§1-16) on the diff hunks.
- EVM chains on Cosmos (Cronos, Sei EVM): also diff the EVM layer against its geth base — see
  the geth section below — and check EVM/Cosmos state sync (precompiles, bank ↔ ERC-20 mapping).

## go-ethereum-Specific Notes

- Forks usually change: consensus engine (`consensus/` — Clique/Parlia/Aura replacements),
  `core/` (gas schedule, state transition), `eth/` (sync, tx pool admission), `rpc/` (new namespaces).
- Check the fork's consensus engine against upstream's: any change to block validation, difficulty,
  or finality logic is consensus-critical.
- Check `params/` (chain config): chain ID, fork block numbers, gas limits — a wrong fork block can
  disable a security hard-fork.
- Check for removed DoS guards: tx pool limits, `eth_getLogs` range caps, RPC method allowlists —
  pair with `evm-rpc-dos-probing` for live verification.

## Output

For each validated finding: file:line in the fork, the upstream original for comparison, what changed,
attack scenario, severity (CVSS 3.1), and PoC or reproduction steps. Also report:
- Upstream version pinned, fork commit audited, diffstat summary.
- List of upstream security fixes checked and their status (present / ABSENT) — absent ones are
  findings, present ones are evidence of diligence.
- Areas of the diff explicitly excluded (generated code, vendored deps) and why.

## Rules

- Audit only targets in scope for an authorized bug bounty program or code the user owns.
- Read-only on source; PoCs run on local forks / testnets only, never against mainnet.
- Never claim a finding from a diff hunk alone — prove reachability and impact.
