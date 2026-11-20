# Bytecode Verification (EVM + Solana)

Verify that the code actually deployed on-chain matches the source code that was audited — by
rebuilding from source and comparing bytecode. A contract/program whose deployed bytecode differs
from its published source may hide backdoors, different logic, or an older vulnerable version.
This is the "deployment-vs-code diff" step: never trust a repo, an explorer's "verified" badge, or
a team's claim without checking the bytes.

## When to Use

- Before auditing any deployed contract/program: confirm the source you are reading is what is live.
- When a program claims to be verified on Etherscan/Sourcify/Solscan but the match looks suspicious.
- When comparing a fork's deployment against its repo (pairs with `chain-fork-diff-audit`).
- When a finding depends on deployed behavior: the PoC must target the deployed bytecode, not the repo.

## When NOT to Use

- Auditing source code with no deployment yet — use `web3-audit` / `solana-vulnerability-scanner`.
- Decompiling unknown bytecode with no source at all — that is reverse engineering, not verification
  (use a decompiler and treat the result as untrusted).

## EVM: Verify Deployed Bytecode vs Source

### Step 1: Fetch the deployed bytecode

```bash
# Runtime (deployed) bytecode
cast code <contract-address> --rpc-url <rpc-url>
# Creation bytecode is in the deployment transaction input data
cast tx <deploy-tx-hash> --rpc-url <rpc-url> --json | jq -r .input
```

Also fetch the creation transaction, deployer address, and block number. Note the chain and RPC used.

### Step 2: Check the easy paths first

1. **Sourcify** (https://sourcify.dev): query by chain + address. A `perfect` match means
   recompiled bytecode AND metadata hash match. A `partial` match means code matches but metadata
   (e.g. source paths) differs — usually benign, but inspect what differs.
2. **Block explorer verification** (Etherscan / BscScan / etc.): check the compiler version,
   optimizer settings, runs, EVM version, and license shown. Do NOT trust the badge alone —
   recompile yourself (Step 3).
3. If neither exists, the contract is UNVERIFIED: treat all source claims as unproven.

### Step 3: Reproducible rebuild (the decisive check)

1. Clone the exact source at the exact commit/tag the deployment claims.
2. Use the EXACT compiler version and settings from the metadata (solc version, optimizer
   enabled + runs, EVM version, `via_ir` flag). The `metadata.json` from Sourcify/explorer gives
   all of these — never guess them.
3. Compile with the same toolchain (solcx / Foundry `forge build` with the pinned solc).
4. Compare:
   - **Creation bytecode**: must match the deployment tx input, except the constructor arguments
     appended at the end (decode and verify them separately: owner, initial params).
   - **Runtime bytecode**: must match `eth_getCode` output, except the trailing metadata hash
     (IPFS/Swarm hash of the metadata). Strip the metadata hash on both sides before comparing,
     and record the metadata hash itself — a different metadata hash with identical code means
     different source paths/comments, not different logic.
5. Tooling: `forge verify-contract --watch` (Foundry), Sourcify's `sourcify-server` verification
   API, or a manual `solc --bin --bin-runtime` comparison with `diff <(xxd ...) <(xxd ...)`.

### Step 4: Interpret the result

| Result | Meaning | Action |
|---|---|---|
| Byte-for-byte match (modulo metadata hash + ctor args) | Source == deployed | Audit the source; it is the live code |
| Runtime matches, creation differs | Proxy / different constructor args | Check ctor args (owner, impl address); if proxy, verify the IMPLEMENTATION address separately |
| Partial match / metadata differs only | Same logic, different build metadata | Acceptable; note the metadata delta |
| Bytecode differs in logic sections | Deployed code ≠ published source | STOP — the audit target is the deployed bytecode, not the repo. Report the mismatch as a finding (hidden code / unpatched version) |
| Unverifiable (no source, no metadata) | Unknown code | Decompile for triage only; do not claim source-level findings |

### Step 5: Proxies and upgradeability

- If the address is a proxy (EIP-1967 / UUPS / Beacon / Diamond), verify EACH implementation
  contract separately, and check who can upgrade (admin / timelock / multisig).
- A verified proxy pointing at an UNVERIFIED implementation is effectively unverified.

## Solana: Verify On-Chain Program vs Source

### Step 1: Fetch the deployed program

```bash
solana program show <program-id> --url <rpc-url>   # program data address, authority, slot
solana program dump <program-id> <output.so> --url <rpc-url>
sha256sum <output.so>
```

Note the upgrade authority: an upgradeable program can change after verification — record the
authority and the slot of the dump.

### Step 2: Check the easy paths first

1. **Solana Verify / Anchor verify** (`anchor verify` / `solana-verify`): checks the program
   against a claimed repo + commit using a reproducible Docker build.
2. **Solscan / Solana Explorer**: shows whether the program is marked verified and the claimed
   source. Do NOT trust the badge alone — rebuild yourself (Step 3).

### Step 3: Reproducible rebuild (the decisive check)

1. Clone the exact source at the exact commit/tag claimed.
2. Rebuild with the SAME toolchain: Solana toolchain version (`solana --version`), Anchor version
   (`anchor --version`), Rust version. Anchor programs must be built with `anchor build` using the
   same Anchor version — different Anchor versions produce different binaries.
3. Prefer the deterministic build: `solana-verify build` (Docker, pinned toolchain) or
   `anchor build --verifiable` where available.
4. Compare `sha256sum` of the rebuilt `.so` against the dumped on-chain `.so`.
   - Exact match → source == deployed.
   - Mismatch → inspect: common benign causes are different toolchain versions or build
     environment (timestamps, paths). Rebuild in the deterministic Docker environment before
     concluding a real difference.
   - Persistent mismatch after deterministic rebuild → deployed program ≠ published source.
     STOP and report the mismatch.

### Step 4: Interpret the result

| Result | Meaning | Action |
|---|---|---|
| Hash match | Source == deployed | Audit the source; it is the live code |
| Hash differs, deterministic rebuild | Different code deployed | Report mismatch; audit the dumped binary's behavior (disassembly / LiteSVM execution) instead of the repo |
| Upgrade authority is a single key / unknown multisig | Program can change at any time | Note as a trust risk; re-verify at audit time (record slot) |
| Program is closed / immutable (no upgrade authority) | Code cannot change | Hash match is durable evidence |

## Rules

- Audit only targets in scope for an authorized bug bounty program or code the user owns.
- Read-only on chain data; rebuilds run locally in Docker / pinned toolchains.
- Never present a repo audit as an audit of the deployment without a bytecode/hash match.
- Record in every report: address/program ID, chain, block/slot of the check, compiler/toolchain
  versions used, and match result (perfect / partial / mismatch / unverifiable).
- A mismatch is itself a reportable finding only if the program's scope covers deployment
  integrity; otherwise it is a blocker note that changes what can be claimed.
