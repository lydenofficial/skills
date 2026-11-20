---
name: evm-audit-governance
description: DAO governance vulnerabilities including flash loan voting, proposal execution ordering, timelock bypass, fake proposals via CREATE2, quorum manipulation, Gnosis Safe module bypasses, and reward distribution attacks. Load when auditing any governance system.
---
# EVM Audit — Governance & DAO Vulnerabilities
Load when auditing DAO voting, timelocks, proposal execution, multi-sig governance, or reward distribution.
## Reference Files
- `references/checklist.md` — Full governance checklist

## Workflow
1. Read the governance code: proposals, voting, execution, timelock, and multisig.
2. Run the checklist in `references/checklist.md` against it, class by class.
3. Validate each suspected finding with a concrete exploit path or PoC — a checklist hit is a lead, not a finding.
4. Report impact-first, with funds at risk quantified.
