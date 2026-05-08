# FHEVM Agent Skill

Portable AI-agent skill for building confidential smart contracts with Zama FHEVM.

This repository is a bounty submission for the Zama Developer Program Mainnet Season 2 Bounty Track. The core artifact is [`SKILL.md`](./SKILL.md): a practical operating manual for AI coding agents such as Claude Code, Cursor, Windsurf, and GitHub Copilot.

`__SKILL.md` is kept as the working copy used in this local workspace.

## What Makes It Different

Most FHEVM mistakes are not ordinary Solidity mistakes. Agents often produce code that looks plausible but leaks assumptions from plaintext Solidity: `require` on private conditions, missing ACL grants, direct use of external encrypted inputs, synchronous decryption assumptions, or stale SDK calls.

This skill is built around preventing those failures.

It gives agents:

- A mental model for encrypted handles instead of plaintext values
- A task decision tree before code generation
- Canonical ACL, encrypted input, and decryption patterns
- Working examples for confidential tokens, blind auctions, and voting
- Frontend guidance using the current Zama Relayer SDK flow
- A common-error table based on real FHEVM developer mistakes
- A self-verification checklist agents can run before returning code
- Adversarial prompts to test whether the skill prevents subtle bugs

## Repository Contents

| Path | Purpose |
|---|---|
| `SKILL.md` | Main skill file for agents |
| `__SKILL.md` | Local working copy of the same skill content |
| `prompts/` | Natural-language prompts for evaluating agent behavior |
| `demo-video.md` | Three-minute demo video script and structure |
| `EVALUATION.md` | Manual evaluation protocol for judges and reviewers |
| `test-results/` | Place for compile/test logs generated during demo validation |

## How To Evaluate

1. Give an AI coding agent access to `SKILL.md`.
2. Run one prompt from `prompts/`.
3. Check the generated code against the self-verification checklist.
4. Run the adversarial prompts from `EVALUATION.md`.
5. Confirm the agent avoids common FHEVM errors:
   - No `if` or `require` on encrypted conditions
   - No direct operations on `externalEuintXX` values before `FHE.fromExternal`
   - ACL permissions re-granted after encrypted state writes
   - User decryption returns handles, not plaintext from Solidity
   - Public decryption uses an async request and callback flow

## Demo Recommendation

Use `prompts/01-confidential-voting.md` for the recorded demo. It exercises encrypted input, encrypted branching, ACL, async public decryption, and tests in one compact scenario.

Run:

```bash
npx hardhat compile
npx hardhat test
```

Store outputs in `test-results/compile.log` and `test-results/hardhat-test.log`.

## Known Limits

This skill is not a replacement for auditing. It helps agents generate safer FHEVM code, but generated contracts should still be reviewed, compiled, tested, and audited before production use.

Areas that should be verified for each generated project:

- Exact package versions
- Current Zama network addresses and relayer configuration
- Gateway callback protection for the target environment
- Token standard compatibility if using ERC-7984 or OpenZeppelin Confidential Contracts
- Business-logic privacy leaks from events, revert messages, timestamps, or public metadata

## Maintenance Notes

FHEVM is moving quickly. The skill is intentionally written with version hygiene reminders so agents avoid stale tutorials and prefer official config exports. Before future submissions or reuse, re-check `SOURCES.md` and update any SDK, config, or decryption-flow changes.
