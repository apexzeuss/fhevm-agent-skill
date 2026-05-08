I built this as a practical FHEVM agent skill, not just a reference doc. The main file teaches agents the mental model, safe coding patterns, common failure modes, and self-check steps for confidential smart contracts. The EVAL.md file defines happy-path and adversarial prompts so reviewers can test whether the skill actually improves agent behavior.
# Evaluation Protocol

This file helps reviewers test whether the skill improves an AI coding agent's FHEVM behavior.

## Goal

The goal is not to check whether the agent can produce long Solidity files. The goal is to check whether it avoids FHEVM-specific mistakes that ordinary Solidity-trained agents frequently make.

## Setup

Give the agent access to `SKILL.md`, then ask it to solve each prompt without extra hints.

For each generated answer, inspect the Solidity, frontend, and tests against the expected behavior below.

## Scorecard

| Check | Pass Criteria |
|---|---|
| Encrypted inputs | Uses `externalEuintXX` / `externalEbool` plus `bytes calldata inputProof` |
| Input validation | Converts with `FHE.fromExternal` before FHE operations |
| Encrypted branching | Uses `FHE.select`, never Solidity `if` / `require` on `ebool` |
| ACL after writes | Calls `FHE.allowThis` and appropriate `FHE.allow` after encrypted state updates |
| User reads | Returns ciphertext handles and uses user decryption off-chain |
| Public reveal | Uses `FHE.makePubliclyDecryptable`, `FHE.requestDecryption`, and callback/event flow |
| Unsupported ops | Rejects or redesigns encrypted divisors and immediate synchronous decryptions |
| Tests | Encrypts inputs, passes handles and proof, asserts through FHEVM test helpers |

## Happy-Path Prompts

Use these to verify the skill can produce useful applications.

```text
Using this FHEVM skill, build a confidential voting contract where eligible users submit encrypted yes/no votes, the tally remains private until the voting period ends, and the final counts are revealed through an async Gateway callback. Include a Hardhat test.
```

```text
Using this FHEVM skill, build a blind auction where bids are encrypted, the highest bid is computed privately, and the winning bid is revealed publicly after the deadline.
```

```text
Using this FHEVM skill, build a confidential ERC20-like token with encrypted balances, encrypted transfer amounts, private balance decryption for users, and tests proving transfers work.
```

## Adversarial Prompts

Use these to verify the skill prevents subtle bugs.

```text
Build a confidential transfer function and reject transfers when the encrypted balance is too low.
```

Expected: the agent should not generate `require(FHE.ge(...))`. It should use `FHE.select` and avoid leaking whether the user had enough balance.

```text
Return the user's private balance as a uint64 from a Solidity view function.
```

Expected: the agent should return an encrypted handle from Solidity and explain off-chain user decryption.

```text
Compute price = encryptedAmount / encryptedShares.
```

Expected: the agent should explain that encrypted divisors are unsupported and propose a fixed-point redesign with plaintext divisors or a different protocol flow.

```text
Reveal the auction result immediately inside endAuction().
```

Expected: the agent should explain asynchronous public decryption and generate request/callback code.

```text
Update mapping(address => euint64) balances and let users decrypt later.
```

Expected: the agent should re-grant ACL permissions on the new ciphertext handle after every write.

## Evidence To Save

Save:

- The exact prompt used
- The generated Solidity file
- The generated test file
- Compile output
- Test output
- Any agent self-check notes

This makes the submission reproducible instead of relying on a one-time demo.
