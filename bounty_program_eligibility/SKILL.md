# Bounty Program Eligibility Check

## Verify the fee and gates on the live page

- Check the fee and the entry gates on the live program page on the day you hunt, before you spend any time hunting.

- Read the live platform payload or the live program page for the submission fee, the reputation requirement, the KYC requirement, and any invite only or proof of concept requirement.

- Treat a program as ineligible when it charges any submission fee, requires reputation points a new account lacks, requires KYC the user has not completed, or is invite only.

- Record the fee, the reputation gate, the KYC gate, and the source you checked before you start hunting.

- Hunt only programs you recorded as free and open, and stop at once when a later check contradicts that record.

## Use raw HTML for HackenProof

- Fetch the raw HTML of a HackenProof program page instead of relying on fetched text, because fetched text drops the Program info block that holds the fee.

- Extract the requirements block from the raw HTML and read the fee and the reputation requirement from that block.

- Treat a program as unverified when you cannot find the requirements block, and do not call it free.

- Treat a reputation requirement of 100 points as common on HackenProof, and check the exact number for each program instead of assuming it is absent.

## Trust only live evidence from the hunt day

- Trust only the live payload or the live page from the day of the hunt, and never trust a stale snapshot or an aggregator list.

- Reverify a program when the last check is from an earlier day, because fees and gates change.

- Check the live Immunefi payload for the free flag and the fee in USDC before hunting an Immunefi program.

- Drop a program from the free list as soon as live evidence shows a fee or a gate, and correct any earlier free claim in the same report.

## Available connectors

- Use `muse.connector_search` to find other connectors and `muse.skill_search` to find related skills.

- If nothing here fits, look on the web for a maintained API or MCP server and set it up with [Custom Connectors and MCPs](/opt/muse/skills/custom-connector-setup/SKILL.md), or build a custom connector.

## Alternatives for better results

- Warm up reputation through a program with no reputation gate before attempting a gated program.

- Prefer a program with a free email submission route when its platform route charges a fee.
