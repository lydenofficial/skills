# skills

A collection of agent skills: bug bounty hunting, smart contract and
protocol security auditing, UI/design, human-voice writing, and
on-chain wallet safety. Each folder is one self-contained skill with
a `SKILL.md` playbook (plus `skill.yaml`, references and scripts
where applicable).

## How to use

Copy a skill folder into your agent's skills directory, for example:

```bash
cp -r bug-bounty ~/.agents/skills/
```

Then use it like any other skill in your agent setup. Skills are
Markdown playbooks, so they work with any agent that supports the
skill format (Claude Code, Muse, and similar).

## Skills (87)

### Bug Bounty

- **bounty_program_eligibility** — Verify a bug bounty program is free to submit and open to a new account before hunting it.
- **bug-bounty** — Complete bug bounty workflow — recon, pre-hunt learning, vulnerability hunting (IDOR, SSRF, XSS, auth bypass, CSRF, race conditions, SQLi, XXE, file upload, business logic, GraphQL, HTTP smuggling, cache poisoning,…
- **bug-chaining** — Standalone bug chaining skill — synthesized from omni-killchain (including references/chaining-impact.md), triage-validation, web2-vuln-classes, bug-bounty, and bb-methodology.
- **bugcrowd-reporting** — Bugcrowd-specific reporting tactics complementing report-writing: VRT category search-and-fallback strategy when no exact match exists, manual severity override when VRT defaults underrate impact, severity-request…

### Security Audit & Hunting

- **address-clustering-attribution** — Explains blockchain address clustering heuristics (UTXO common-input ownership, account-based deposit sweeps), entity and label layers, attribution confidence, peel chains and taint-style tracing concepts, and…
- **agentic-app-audit** — Black-box security audit of a DEPLOYED AI agent (not the MCP server behind it) — tool-call hijacking, cross-session memory poisoning, confused-deputy via connected tools, agent-to-agent IDOR, excessive agency /…
- **argus** — Argus — the all-seeing scanner suite. Six automated scanners for high-value web + LLM bug classes — CORS misconfiguration (origin reflection / null / credentialed read), CRLF & host-header injection, NoSQL injection…
- **auditing-cross-chain-bridge-and-message-trust** — Audit a cross-chain bridge or messaging protocol for misplaced trust in cross-chain messages: forged or unverified source-event proofs, spoofable or too-small signer sets and validator quorums, replayable messages on…
- **balancer-v3-invariant-fuzzing** — Stateful invariant fuzzing of Balancer V3 — single Vault accounting, pool math, hooks, ERC4626 buffers, and BPT — with Echidna/Medusa/Foundry.
- **bb-methodology** — Use at the START of any bug bounty hunting session, when switching targets, or when feeling lost about what to do next.
- **bond-auction-mechanism-audit** — Audit Olympus-style bond auctions — price discovery, discount/control variable, capacity, vesting, and front-running.
- **bsc-contract-audit** — Evidence-based due-diligence audit for BNB Smart Chain (chain 56) token contracts, NFT mints, and deposit-refund "lock-to-allocate" escrows — verification status, live privilege inventory (owner/pause/blacklist/hidden…
- **bytecode-verification** — Verify deployed on-chain code matches its published source via reproducible rebuild — EVM (eth_getCode / creation bytecode vs solc recompilation with exact version and settings, Sourcify perfect/partial match,…
- **cdp-liquidation-fuzz** — Fuzz MakerDAO-style CDP liquidation (Vat/Clip/Dog/Spot, Dutch auctions) with stateful invariant testing (Echidna/Medusa/Foundry).
- **certora-formal-verification-review** — Certora formal verification with CVL — write mathematical proofs for smart contract correctness: rules, invariants, ghost variables, hooks, parametric rules, multi-contract verification, and counter-example debugging.
- **chain-fork-diff-audit** — Audit a blockchain client fork by diffing it against upstream — Cosmos SDK forks (Sei, Cronos, Osmosis) vs cosmos/cosmos-sdk, and go-ethereum forks (BSC, Cronos EVM, any EVM chain client) vs ethereum/go-ethereum.
- **cicd-security** — CI/CD pipeline security hunting — GitHub Actions workflow injection, secret exfiltration, self-hosted runner poisoning, dependency confusion, OIDC token theft, and supply chain attacks.
- **client-reverse** — Client-side request-signing and anti-bot token reversal for bug bounty — when a request carries a sign/sig/hmac/token/nonce/timestamp/X-Sensor header that Burp Repeater cannot replay, recover the signer just enough to…
- **cloud-pentest** — Post-access cloud exploitation for AWS, GCP, and Azure — what to do AFTER you obtain credentials or reach a metadata endpoint.
- **container-audit** — Audit container images, Dockerfiles, and Kubernetes manifests for misconfigurations, excessive privileges, exposed secrets, and runtime risks.
- **cosmos-vulnerability-scanner** — Scans Cosmos SDK blockchain modules and CosmWasm contracts for consensus-critical vulnerabilities — chain halts, fund loss, state divergence.
- **credential-attack** — Password spray methodology for bug bounty — when to do it vs web-vuln hunting, the wordlist-gen + breach-check + osint-employees + spray pipeline, mode selection (http-form / oauth / o365 / okta), rate-limit + lockout…
- **cross-chain-clustering-techniques-agent** — Guides multi-chain wallet and entity clustering using public bridge traces, wrapped-asset flows, temporal and behavioral heuristics, unified graphs with chain-prefixed addresses, and confidence scoring.
- **erc4337-bundler-entrypoint-audit** — Audit an ERC-4337 account-abstraction deployment: smart-account UserOperation validation (signature/nonce acceptance), paymaster sponsorship scope and spend caps (deposit drain), bundler/entry-point replay and…
- **evm-rpc-dos-probing** — Probe an EVM JSON-RPC endpoint for denial-of-service weaknesses — missing or weak rate limits, expensive methods with no caps (eth_getLogs wide ranges, eth_call, debug_traceTransaction, trace_filter, eth_feeHistory),…
- **fizz** — Generate stateful Solidity invariant fuzz suites from Foundry or Hardhat projects for Echidna and Medusa: extract ABIs, pick entry points, generate handlers and properties, run coverage cycles and fuzz campaigns, sync…
- **go-security-review** — Security best practices and vulnerability prevention for Golang — injection (SQL, command, XSS), cryptography, path traversal, SSRF and HTTP security headers, cookies, secrets management, memory safety, PII in logs,…
- **governance-timelock-audit** — Audit DAO governance and timelock systems for security vulnerabilities: flash-loan voting and proposal creation, proposal execution front-running and ordering, CREATE/CREATE2 contract substitution, quorum manipulation…
- **graphql-audit** — GraphQL security hunting — introspection abuse, field suggestion enumeration (clairvoyance), batching DoS, IDOR via aliasing, auth bypass, injection via arguments, subscription abuse, depth/complexity bombs, and WAF…
- **gyro-eclp-math-audit** — Audit Gyro E-CLP (elliptic concentrated liquidity pool) math on Balancer — ellipse invariant, derived parameters, swap quadratic, rounding, and liquidity bounds.
- **johnwick** — Use when the user says "run johnwick" / "/johnwick", hands you a bug-bounty target or scope and wants the full manual+recon+automated hunt run end-to-end, or asks to hunt a target with total coverage (every in-scope…
- **jsmax** — High-impact JavaScript analysis and exploitation engine.
- **launch-curve-audit** — Method and tooling to audit a token launch curve on BSC — bonding-curve or tranche IEO, Agent-ID or allowlist gating, escrowed bids, graduation into a pool, coupled off-chain emissions.
- **layerzero-oft-review** — Security review checklist for LayerZero v2 OFT / OFT Adapter and OApp configuration: DVN stack and quorum, send/receive library, Executor, enforced options, peers, rate limit, blocklist, escrow supply reconciliation,…
- **live-browser-js-harvesting** — Harvest JavaScript files from a target using the live browser when curl/httpx/katana get blocked (403, WAF, bot protection), then feed the JS to jsmax for secret/endpoint extraction.
- **llm-redteam** — LLM application red-teaming — prompt injection (direct + indirect), jailbreak, system-prompt leak, data exfiltration, guardrail bypass, multi-turn crescendo, cross-lingual + cipher + invisible-unicode token smuggling,…
- **master-vault-strategy-audit** — Audit Lista DAO MasterVault hub-and-spoke yield aggregator — strategy allocation, debt accounting, fees, async unstaking, and strategy migration.
- **mcp-client-probing** — Red-team the tool layer of an AI agent from the client side: tool discovery (read every tool definition as the model sees it), prompt injection via tool descriptions (tool poisoning, line jumping), tool shadowing and…
- **mcp-server-audit** — Security audit of Model Context Protocol (MCP) servers — tool poisoning, prompt injection via tool descriptions and results, unscoped/over-privileged tools, path traversal in file tools, command injection in shell/exec…
- **meme-coin-audit** — Meme coin and token security audit — rug pull detection (honeypot, hidden mint, fee manipulation, LP lock bypass), Solana SPL token analysis (freeze authority, mint authority, metadata mutability), Token-2022 extension…
- **mobile-app-audit** — Audit iOS and Android mobile applications against OWASP MASVS / MASTG — insecure storage (keychain/keystore classes, allowBackup, clipboard, FLAG_SECURE), weak crypto (AES-ECB, hardcoded keys, IV reuse, PBKDF2…
- **mobile-pentest** — Mobile app pentest for bug bounty (Android APK + iOS IPA) — runtime-first workflow: install app, proxy through Burp/mitmproxy, drive the UI, capture packets, then test the API exactly like a web target; escalate to…
- **move-audit** — Guides authorized security auditing of Move smart contracts on Sui and Aptos for bug-bounty and client-audit engagements.
- **nuclei-offline-templates** — Write custom Nuclei templates to codify a finding into a repeatable, mass-scannable check and run them offline from a local template directory (no template update / no network): capture the minimal repro, write…
- **oauth-oidc-deep-audit** — Deep audit of OAuth 2.0 / OpenID Connect integrations: federated assertion verification (signature wrapping/stripping, audience/issuer/expiry, exact redirect_uri, state/nonce, identity mapping), PKCE and device-code…
- **omni-killchain** — Elite end-to-end offensive bug-bounty hunting orchestrator.
- **on-chain-investigator-agent** — Operates as an on-chain forensics investigator using only public chain data and OSINT—tracing flows across chains, clustering addresses, reviewing contracts for risk patterns, detecting scam vectors, and producing…
- **optimism** — OP Stack protocol reference for auditing Base and Optimism bedrock: derivation, bridges and messaging, deposits and withdrawals, predeploys and precompiles, fault proofs and dispute games, batcher and SystemConfig,…
- **oracle-price-feed-audit** — Audits Solidity oracle integrations for vulnerabilities including missing stale price checks against heartbeat intervals, missing L2 sequencer uptime validation, same heartbeat for multiple feeds, assuming oracle…
- **pii-hunter** — Elite PII (Personally Identifiable Information) leak hunting skill for authorized bug bounty and pentests.
- **property-based-testing** — Writes, reviews, and debugs property-based tests — Hypothesis, fast-check, proptest, jqwik, rapid, and Echidna or Medusa for Solidity invariants.
- **psm-stability-module-audit** — Audit MakerDAO-style Peg Stability Modules (PSM / PSM3) — 1:1 mint/redeem, tin/tout fees, debt ceiling, gem joins, depeg exposure, and admin roles.
- **rate-provider-oracle-audit** — Audit Balancer rate providers and yield-bearing-token exchange rates — getRate() manipulation, donation/inflation attacks, staleness, upgradeability, and registry review.
- **reclamm-pool-audit** — Audit Balancer ReClamm (AutoRange / reclaiming AMM) pools — virtual balances, centeredness, daily price-shift exponent, price-ratio updates, and proportional-only liquidity.
- **rust-review** — Performs comprehensive Rust security review for safe/unsafe boundary issues, memory safety in unsafe blocks, concurrency hazards, panic-induced DoS, FFI safety, and async runtime mistakes.
- **security-arsenal** — Security payloads, bypass tables, wordlists, gf pattern names, always-rejected bug list, and conditionally-valid-with-chain table.
- **slither-semgrep-solidity** — Static analysis sweep for Solidity — run Slither's 90+ built-in detectors, then Semgrep custom-pattern rules (Decurity ruleset + project-specific rules), then triage.
- **solana-anchor-test-harness** — Solana Anchor program testing for security PoC — local testing with Surfpool, LiteSVM, and Mollusk, fuzz testing (Trident, cargo-fuzz), Anchor program structure, and security review.
- **solana-vulnerability-scanner** — Scans Solana programs for 6 critical vulnerabilities including arbitrary CPI, improper PDA validation, missing signer/ownership checks, and sysvar spoofing. Use when auditing Solana/Anchor programs.
- **soroban-common-mistakes** — Review Stellar Soroban (Rust) smart contracts for common security mistakes: missing require_auth, reinitialization, wrong storage type and missing TTL extension, unchecked arithmetic, rounding errors, missing state…
- **supply-chain-security** — Review new or changed dependencies for supply-chain compromise before they enter a project — malicious install scripts (preinstall/postinstall), binding.gyp/node-gyp install-time execution (June 2026 worm),…
- **ton-auditor** — Parallelized security audit of TON/FunC/Tact smart contracts: 8 specialist agents covering attack vectors, math precision, access control, economic security, execution trace, invariants, periphery, and first principles,…
- **triage-validation** — Finding validation before writing any report — 8-Question Gate (all 8 questions), 4 pre-submission gates, always-rejected list, conditionally valid with chain table, CVSS 3.1 quick reference, severity decision guide,…
- **uniswap-v4-hook-audit** — Security-first Uniswap v4 hook audit — hook lifecycle and permissions risk matrix (all 14 hook permissions), NoOp rug-pull attack via beforeSwapReturnDelta, delta accounting invariants (sum of deltas == 0),…
- **web-extension-auditor** — Audit Chrome or Firefox WebExtensions for manifest compatibility, permission scope, CSP, content-script isolation, service-worker lifecycle, storage, bundle size, and store readiness.
- **web2-recon** — Web2 recon pipeline — subdomain enumeration (subfinder, Chaos API, assetfinder), live host discovery (dnsx, httpx), URL crawling (katana, waybackurls, gau), directory fuzzing (ffuf), JS analysis (LinkFinder,…
- **web2-vuln-classes** — Complete reference for 26 web2 bug classes with root causes, detection patterns, bypass tables, exploit techniques, and real paid examples.
- **web3-audit** — Smart contract security audit — 10 DeFi bug classes (accounting desync, access control, incomplete path, off-by-one, oracle, ERC4626, reentrancy, flash loan, signature replay, proxy), pre-dive kill signals (TVL < $500K…
- **webauthn-passkey-deep-audit** — Audit the server side of WebAuthn/passkey ceremonies for verification bugs that let an attacker-shaped response become an authenticated session: challenge binding (server-issued, single-use), exact origin and…

### UI & Design

- **anti-ui-slop** — Stop coding agents from shipping generic UI.
- **design-taste-frontend** — tasteskill 2.5.1, the anti-slop frontend skill.
- **emil-design-eng** — Emil Kowalski's design-engineering philosophy for UI polish, component design, animation decisions, and the invisible details that make software feel great.
- **frontend-design** — Guidance for distinctive, intentional visual design when building new UI or reshaping an existing one. Helps with aesthetic direction, typography, and making choices that don't read as templated defaults.
- **hallmark** — Design skill for websites and new pages that look designed, not generated.
- **make-interfaces-feel-better** — Design engineering principles for making interfaces feel polished.
- **stageshow** — Build faux product UI that looks like a real product, as a still illustration or an animated demo with a cursor.
- **ui-ux-pro-max** — UI/UX design intelligence for web, mobile, and desktop.
- **web-design-guidelines** — Review UI code for Web Interface Guidelines compliance. Use when asked to "review my UI", "check accessibility", "audit design", "review UX", or "check my site against best practices".

### Writing & Human Voice

- **anti-ai-slop-writing** — Produces human-sounding text that avoids detectable AI writing patterns.
- **ghostwriter** — Writes, rewrites, and critiques messages, posts, and docs as the user, in their own voice with machine tells stripped.
- **humanizer** — Rewrite AI-sounding text so it reads like the writer without changing what it says.
- **report-writing** — Bug bounty report writing for H1/Bugcrowd/Intigriti/Immunefi — report templates, human tone guidelines, impact-first writing, CVSS 3.1 scoring, title formula, impact statement formula, severity decision guide, downgrade…

### Wallet & On-Chain Safety

- **fast-free-mint-claim** — Free-mint an NFT and claim a token/coin together with a fast process: parallel checks, previews and submits, one upfront preflight, and auto speed-up if a transaction stalls — but speed alone does not guarantee a slot.
- **fastmint-fcfs** — Fire a free or paid FCFS mint the moment it opens from the Binance Agentic Wallet on BSC: auto recon, 120ms polling across 6 fast RPCs, one broadcast only — but speed alone does not guarantee a slot.
- **on-chain-safety-rails** — Apply always-on safety rails for on-chain wallet operations, including secrets, simulation, confirmation, approvals, and fail-closed handling.
- **single-confirm-auto-revoke** — Require one mandatory confirmation per wallet activity (Yes=execute, No=cancel). After an activity that creates an approval, automatically revoke that approval. Clean up the wallet when dangerous approvals are found.
- **total-cost-disclosure** — For any activity, always show the total cost, visible and hidden, in three currencies at once: crypto (e.g. BNB), Rupiah (IDR), and USD, including gas, fees, and slippage. Never guess; use a live rate.

## Notes

- Binary assets (photos, fonts, icons) are not included; a few
  design skills reference stock photos that ship with the original
  skill sources, not here.
- `omni-killchain/cheatsheets/encoding-tricks.md` is omitted: it
  contains raw non-UTF-8 bytes by design.
- A few personal skills (personal workflows and account-specific
  safety rules) are intentionally not published.
- Skills that originated in other repositories keep their original
  `LICENSE` files and attribution inside their folders.

## License

MIT — see [LICENSE](LICENSE). Third-party skills keep their own
licenses where present.
