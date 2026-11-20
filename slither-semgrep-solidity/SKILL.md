---
name: slither-semgrep-solidity
description: Static analysis sweep for Solidity — run Slither's 90+ built-in detectors, then Semgrep custom-pattern rules (Decurity ruleset + project-specific rules), then triage. Use when sweeping thousands of lines of Solidity for known vulnerability patterns before manual audit, or when a target codebase is too large to read line-by-line first.
---

# Slither + Semgrep for Solidity

Two complementary static analyzers, run in order. Full detail lives in the
bundled sub-skills — read them before running the tools:

- `slither/SKILL.md` — Slither (Trail of Bits): installation, 90+ detectors,
  triage workflow, custom detectors, printers, CI integration, configuration.
- `semgrep-solidity/SKILL.md` — Semgrep: custom YAML rules, taint tracking,
  Decurity community ruleset, autofix, CI/CD integration.

## The two tools, and when each wins

| Tool | Strength | Weakness |
| --- | --- | --- |
| **Slither** | 90+ built-in detectors with deep Solidity understanding (inheritance, state-variable tracking, CFG). Fast (seconds). Machine-readable JSON. Python API for custom detectors. | Cannot catch runtime logic errors, economic exploits, or oracle manipulation. 30–60% false-positive rate on real projects — triage is mandatory. |
| **Semgrep** | Custom pattern rules you write for YOUR codebase (protocol-specific invariants, project anti-patterns). Taint tracking (`mode: taint`) traces data flow source → sink. | Matches patterns only — no program-state reasoning. Solidity support is community-driven (tree-sitter-solidity). Does not replace Slither. |

Use **both**: Slither for the broad sweep, Semgrep for the targeted rules
Slither's detectors don't cover.

## Installation

```bash
# Slither (package name is slither-analyzer, NOT slither)
uv tool install slither-analyzer
# or: pip install slither-analyzer

# Semgrep
pip install semgrep

# Verify
slither --version
semgrep --version
```

## Workflow

### 1. Slither sweep first

```bash
# Foundry project (uses forge build output)
slither . --filter-paths "lib/|test/|script/" --json slither-report.json

# Or target specific detectors only
slither . --detect reentrancy-eth,arbitrary-send-erc20,uninitialized-state

# List all detectors for your version (IDs change between versions)
slither --list-detectors
```

- Always `--filter-paths` to exclude dependencies, tests, and scripts —
  without it Slither reports on every file including `lib/`.
- Do **not** run `slither .` unconfigured and treat the output as an audit.
- Read `slither/SKILL.md` for detector categories, custom detectors,
  printers (`--print`), and `.slither.conf.json` configuration.

### 2. Semgrep custom-rule sweep

```bash
# Community ruleset for smart contracts (start here before writing rules)
git clone https://github.com/Decurity/semgrep-smart-contracts.git /tmp/semgrep-rules
semgrep --config /tmp/semgrep-rules /path/to/contracts --json -o semgrep-report.json

# Taint tracking (data flow source -> sink) for a specific pattern
semgrep --config <ruleset-with-taint> --json /path/to/contracts
```

- `--config auto` does **not** include Solidity rules — point at a specific
  ruleset or write your own.
- Write custom rules for the target's invariants (the bugs Slither's generic
  detectors miss). See `semgrep-solidity/SKILL.md` for rule-writing,
  taint mode, and the operator reference.
- Semgrep finds patterns, not bugs — every hit needs manual confirmation.

### 3. Triage (mandatory — never skip)

1. **Deduplicate** Slither and Semgrep hits that describe the same issue.
2. **Discard** informational / optimization findings — they are not security
   issues. Expect 30–60% of raw Slither output to be false positives.
3. **Confirm** each remaining hit by reading the code: trace the data flow,
   check for upstream validation, check whether the path is reachable by an
   unprivileged actor. A detector hit is a lead, not a finding.
4. **Promote** only confirmed, reachable, impactful issues to the
   triage-validation gate (8-Question Gate) before writing any report.
   Raw analyzer output is never submitted as a finding.

## What this skill does NOT do

- It does not catch runtime logic errors, economic exploits, or oracle
  manipulation — use fuzzing (Echidna/Medusa, see property-based-testing)
  and manual audit (see web3-audit) for those.
- It does not replace manual review. Static analysis finds known patterns;
  novel logic bugs need a human reading the code.
- It does not prove exploitability. Every hit needs a PoC (Foundry test on
  a local fork) before it counts as a validated finding.

## Sources

- Slither skill: `andresdefi/cryptoskills@slither` v1.0 (Apache-2.0, author 0xinit).
- Semgrep skill: `andresdefi/cryptoskills@semgrep-solidity` v1.0 (Apache-2.0, author 0xinit).
- Upstream tools: Slither by Trail of Bits; Semgrep; Decurity
  semgrep-smart-contracts ruleset.
