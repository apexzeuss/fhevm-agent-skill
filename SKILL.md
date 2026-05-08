# FHEVM Agent Skill — Confidential Smart Contract Development
> For AI coding agents: Claude Code, Cursor, Windsurf, GitHub Copilot

---

## SUBMISSION SNAPSHOT

This `SKILL.md` is a portable, agent-agnostic operating manual for building confidential smart contracts with Zama's FHEVM. It is designed to let AI coding agents go from a natural-language prompt to correct Solidity, tests, and frontend integration without requiring the developer to already know FHE-specific patterns.

### What This Skill Enables

- Confidential Solidity contracts using encrypted FHEVM types such as `ebool`, `euint64`, and `eaddress`
- Correct handling of encrypted user inputs through `externalEuintXX` values and input proofs
- Safe encrypted branching with `FHE.select` instead of plaintext `if` / `require` logic
- Correct ACL handling with `FHE.allowThis`, `FHE.allow`, and transient permissions
- Private user decryption through the Zama Relayer SDK and EIP-712 signatures
- Public asynchronous decryption through Gateway callbacks
- End-to-end examples for tokens, auctions, voting, tests, and frontend flows

### Judging Criteria Coverage

| Bounty Judging Criteria | Where This Skill Addresses It |
|---|---|
| Accuracy | Mental model, encrypted type reference, operation rules, ACL rules, decryption patterns |
| Completeness | Contract setup, Solidity examples, Hardhat tests, React frontend integration, deployment notes |
| Agent effectiveness | Decision tree, quick reference card, canonical patterns, self-verification checklist |
| Code quality | OpenZeppelin-first token path, manual implementation path, focused examples, typed SDK snippets |
| Error prevention | Error-to-fix table, anti-pattern warnings, ACL reminders, async decryption guidance |

### Maintainer Notes

- Source-checked against current Zama Solidity Guides and Relayer SDK Guides before submission.
- Written for agents that tend to make Solidity look correct while missing FHEVM-specific rules.
- Optimized for failure prevention: every section teaches the agent what to do and what not to do.
- Treat hardcoded addresses and SDK config as release-sensitive; prefer official config exports.

### Included Examples

1. OpenZeppelin ERC-7984 confidential token
2. Manual confidential token with encrypted balances and allowances
3. Blind auction with encrypted bids and public async reveal
4. Confidential voting with encrypted tallies and Gateway reveal

---

## AGENT: READ THIS FIRST

Before writing a single line of FHEVM code, internalize this mental model. Most FHE bugs come from ignoring it.

### The Fundamental Paradigm Shift

| Normal Solidity | FHEVM Solidity |
|---|---|
| `uint256 balance` | `euint64 balance` (a *handle*, not a value) |
| `if (balance > 0)` | `FHE.select(FHE.gt(balance, zero), ...)` |
| `require(amount <= balance)` | Use `FHE.select` — never revert on encrypted condition |
| `return balance` | Return the handle — user decrypts off-chain |
| Operations run on-chain synchronously | Operations are *symbolic* — actual FHE runs on coprocessors |
| Results are immediate | Decryption is *asynchronous* |

**The single most important rule:** Encrypted values are opaque `bytes32` handles. You never see the plaintext inside a contract. You can only *compute on them*, *pass them around*, and *grant permission to decrypt them*.

### Agent Decision Tree — When Given Any FHEVM Task

```
User asks me to build a confidential contract
│
├── Does it store user-specific private data?
│     → Map address => euintXX
│     → ALWAYS call FHE.allowThis + FHE.allow after every write
│
├── Does it need conditional logic?
│     → NEVER use if/require on encrypted bool
│     → ALWAYS use FHE.select(condition, valueIfTrue, valueIfFalse)
│
├── Does it accept user input?
│     → Function params must be externalEuintXX + bytes calldata inputProof
│     → First line of function body: euintXX val = FHE.fromExternal(input, proof)
│
├── Does it need to reveal a result publicly on-chain?
│     → FHE.makePubliclyDecryptable(handle) + FHE.requestDecryption (async Gateway callback)
│     → Result arrives in a callback function, NOT immediately
│
└── Does a user need to read their own private data?
      → Return euintXX handle from a view function
      → User decrypts off-chain via @zama-fhe/relayer-sdk + EIP-712 signature
```

---

## QUICK REFERENCE CARD

```solidity
// ── IMPORTS ──────────────────────────────────────────────────────────────────
import {FHE, euint8, euint16, euint32, euint64, euint128, euint256,
        ebool, eaddress,
        externalEuint8,  externalEuint16, externalEuint32,
        externalEuint64, externalEuint128, externalEuint256,
        externalEbool,   externalEaddress} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

// ── CONTRACT SETUP ────────────────────────────────────────────────────────────
contract MyContract is SepoliaConfig { ... }

// ── ACCEPT ENCRYPTED INPUT (always first line) ────────────────────────────────
function foo(externalEuint32 enc, bytes calldata proof) external {
    euint32 val = FHE.fromExternal(enc, proof);
}

// ── ARITHMETIC ────────────────────────────────────────────────────────────────
FHE.add(a, b)     FHE.sub(a, b)     FHE.mul(a, b)
FHE.min(a, b)     FHE.max(a, b)     FHE.neg(a)
FHE.div(a, plain) FHE.rem(a, plain) // PLAINTEXT divisor ONLY

// ── BITWISE ───────────────────────────────────────────────────────────────────
FHE.and(a,b)  FHE.or(a,b)  FHE.xor(a,b)  FHE.not(a)
FHE.shl(a, n) FHE.shr(a, n) FHE.rotl(a, n) FHE.rotr(a, n)

// ── COMPARISON (returns ebool) ────────────────────────────────────────────────
FHE.eq(a,b)  FHE.ne(a,b)  FHE.lt(a,b)  FHE.le(a,b)  FHE.gt(a,b)  FHE.ge(a,b)

// ── CONDITIONAL (NEVER use if/require on ebool) ───────────────────────────────
euint32 result = FHE.select(condition_ebool, value_if_true, value_if_false);

// ── CASTING ───────────────────────────────────────────────────────────────────
FHE.asEuint32(42)          // plaintext → encrypted
FHE.asEbool(true)
FHE.asEaddress(addr)
FHE.asEuint64(euint32Val)  // between encrypted types

// ── ACCESS CONTROL (ALWAYS after every write to encrypted state) ──────────────
FHE.allowThis(handle);            // contract can compute on it later
FHE.allow(handle, someAddress);   // address can decrypt it
FHE.allowTransient(handle, addr); // temp permission, this tx only
FHE.isSenderAllowed(handle);      // reverts if msg.sender not allowed

// ── RANDOMNESS ────────────────────────────────────────────────────────────────
FHE.randEuint32()   FHE.randEuint64()

// ── PUBLIC DECRYPTION REQUEST (async, Gateway callback) ───────────────────────
FHE.makePubliclyDecryptable(handle);
FHE.requestDecryption(handles[], callbackSelector, 0, deadline);
```

---

## ENVIRONMENT SETUP

### Step 1: Clone the Official Hardhat Template

```bash
git clone https://github.com/zama-ai/fhevm-hardhat-template my-project
cd my-project
npm install
```

### Step 2: Configure `.env`

```env
MNEMONIC="word1 word2 word3 word4 word5 word6 word7 word8 word9 word10 word11 word12"
INFURA_API_KEY=your_key_here
```

### Step 3: Install Additional Libraries

```bash
# OpenZeppelin Confidential Contracts (review dependency and version before production)
npm install @openzeppelin/confidential-contracts

# If NOT using template, install manually:
npm install @fhevm/solidity
npm install @zama-fhe/relayer-sdk    # frontend / relayer SDK
```

> **Version hygiene:** Older tutorials may refer to `@fhevm/sdk`, `initFhevm`, or `reencrypt`. Current Zama Relayer SDK docs use `@zama-fhe/relayer-sdk`, `initSDK`, and `userDecrypt`. Prefer the current package unless a project is intentionally pinned to an older stack.

### Step 4: Frontend — Prefer Vite for SDK Compatibility

```bash
npm create vite@latest my-frontend -- --template react-ts
cd my-frontend
npm install @zama-fhe/relayer-sdk ethers
```

**Required `vite.config.ts`:**

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  optimizeDeps: { exclude: ["@zama-fhe/relayer-sdk"] },
  server: {
    headers: {
      "Cross-Origin-Opener-Policy": "same-origin",
      "Cross-Origin-Embedder-Policy": "require-corp",
    },
  },
});
```

> ⚠️ Without the COOP/COEP headers, SharedArrayBuffer (required by WASM) is unavailable and the SDK silently falls back to demo mode with no real encryption.

---

## ENCRYPTED TYPES — PRACTICAL REFERENCE

### Storage Types (inside the contract)

| Type | Size | Best For |
|------|------|---------|
| `ebool` | 1 bit | Flags, access rights, conditions |
| `euint8` | 8 bit | Ages, ratings, small counts |
| `euint16` | 16 bit | Medium counters |
| `euint32` | 32 bit | General integers |
| `euint64` | 64 bit | **Token balances (recommended)** |
| `euint128` | 128 bit | Large financial amounts |
| `euint256` | 256 bit | Maximum precision |
| `eaddress` | 160 bit | Encrypted Ethereum addresses |

> **Rule of thumb:** Use `euint64` for token balances. It avoids common overflow traps and matches real-world financial precision when using 6 decimal places.

### External Input Types (function parameters from users)

| External Type | Converts To |
|---|---|
| `externalEbool` | `ebool` |
| `externalEuint8` | `euint8` |
| `externalEuint16` | `euint16` |
| `externalEuint32` | `euint32` |
| `externalEuint64` | `euint64` |
| `externalEuint128` | `euint128` |
| `externalEuint256` | `euint256` |
| `externalEaddress` | `eaddress` |

**Always convert immediately using `FHE.fromExternal`:**

```solidity
function deposit(externalEuint64 encAmount, bytes calldata inputProof) external {
    euint64 amount = FHE.fromExternal(encAmount, inputProof); // validates ZK proof
    // now use `amount` in FHE operations
}
```

---

## FHE OPERATIONS — FULL REFERENCE

### Arithmetic

```solidity
euint64 a; euint64 b;
euint64 sum      = FHE.add(a, b);
euint64 diff     = FHE.sub(a, b);
euint64 product  = FHE.mul(a, b);
euint64 minimum  = FHE.min(a, b);
euint64 maximum  = FHE.max(a, b);
euint64 negated  = FHE.neg(a);

// Division/remainder: ONLY with plaintext divisors
euint64 divided   = FHE.div(a, uint64(100));
euint64 remainder = FHE.rem(a, uint64(7));
```

### Bitwise

```solidity
euint32 r1 = FHE.and(a, b);
euint32 r2 = FHE.or(a, b);
euint32 r3 = FHE.xor(a, b);
euint32 r4 = FHE.not(a);

// Shift amount is automatically taken modulo the bit width
// FHE.shr(euint64_val, 70) == FHE.shr(euint64_val, 6) because 70 % 64 = 6
euint32 r5 = FHE.shr(a, 3);
euint32 r6 = FHE.shl(a, uint8(4));
euint32 r7 = FHE.rotl(a, uint8(2));
euint32 r8 = FHE.rotr(a, uint8(5));
```

### Comparison — Always Returns `ebool`

```solidity
ebool eq = FHE.eq(a, b);
ebool ne = FHE.ne(a, b);
ebool lt = FHE.lt(a, b);   // a < b
ebool le = FHE.le(a, b);   // a <= b
ebool gt = FHE.gt(a, b);   // a > b
ebool ge = FHE.ge(a, b);   // a >= b
```

### Conditional Logic — The FHE Way

```solidity
// ── Pattern 1: Safe transfer (no underflow reveal) ───────────────────────────
ebool canAfford = FHE.ge(balance, amount);
euint64 toSend  = FHE.select(canAfford, amount, FHE.asEuint64(0));
// If they can't afford it, silently send 0 — no revert, no info leak

// ── Pattern 2: Overflow-safe addition ────────────────────────────────────────
euint64 newTotal = FHE.add(total, amount);
ebool overflow   = FHE.lt(newTotal, total);   // wrapped = overflow occurred
total = FHE.select(overflow, total, newTotal); // keep old value if overflow

// ── Pattern 3: Conditional state update ──────────────────────────────────────
balance = FHE.select(isEligible, FHE.add(balance, reward), balance);
```

### Type Casting

```solidity
// Plaintext → encrypted
euint32  enc32   = FHE.asEuint32(42);
euint64  enc64   = FHE.asEuint64(1000);
ebool    encBool = FHE.asEbool(true);
eaddress encAddr = FHE.asEaddress(msg.sender);

// Between encrypted types
euint64 wider   = FHE.asEuint64(euint32Val);  // safe upcast
euint32 narrow  = FHE.asEuint32(euint64Val);  // truncates — use carefully
ebool   asBool  = FHE.asEbool(euint8Val);     // nonzero = true
```

### On-Chain Randomness

```solidity
euint8   r8  = FHE.randEuint8();
euint16  r16 = FHE.randEuint16();
euint32  r32 = FHE.randEuint32();
euint64  r64 = FHE.randEuint64();
// Deterministic across coprocessors, indistinguishable externally
```

---

## ACCESS CONTROL (ACL) — THE MOST CRITICAL SECTION

Every FHE write operation creates a **new ciphertext handle**. You MUST re-grant ACL permissions every time stored encrypted state changes. Forgetting this is the #1 bug in FHEVM contracts.

### The Golden Rule

```
After EVERY FHE operation that updates stored encrypted state:
  Step 1: FHE.allowThis(newHandle)      ← so this contract can compute on it later
  Step 2: FHE.allow(newHandle, owner)   ← so the owner can decrypt it
```

### All ACL Methods

```solidity
// ── Permanent permissions ─────────────────────────────────────────────────────
FHE.allowThis(handle);              // This contract can compute on handle
FHE.allow(handle, address);         // address can decrypt handle

// ── Temporary permission (current transaction only) ───────────────────────────
FHE.allowTransient(handle, address); // Useful for intermediate computation values

// ── Validation ────────────────────────────────────────────────────────────────
FHE.isSenderAllowed(handle);         // Reverts if msg.sender not in ACL — use to guard reads

// ── Public decryption ─────────────────────────────────────────────────────────
FHE.makePubliclyDecryptable(handle); // Anyone can decrypt — for auction results, vote tallies
bool pub = FHE.isPubliclyDecryptable(handle);
```

### Canonical ACL Pattern for Mappings

```solidity
mapping(address => euint64) private _balances;

function _updateBalance(address user, euint64 newBalance) internal {
    _balances[user] = newBalance;
    FHE.allowThis(_balances[user]);   // contract can keep computing on it
    FHE.allow(_balances[user], user); // user can decrypt their own balance
}
```

---

## DECRYPTION PATTERNS

### Pattern 1: User Reads Their Own Data (Private Decryption)

The contract returns an encrypted handle. The user decrypts it client-side via the relayer/KMS. No one else can decrypt it.

**Contract side:**

```solidity
// Return the ciphertext handle — plaintext never leaves the user's browser
function getBalance(address user) external view returns (euint64) {
    return _balances[user];
}
```

**Frontend side:**

```typescript
import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/bundle";

async function decryptMyBalance(contract: any, signer: any, userAddress: string) {
    // 1. Initialize SDK (loads WASM — do once per app session)
    await initSDK();
    const instance = await createInstance(SepoliaConfig);

    // 2. Fetch ciphertext handle from contract
    const handle = await contract.getBalance(userAddress);
    const contractAddress = await contract.getAddress();

    // 3. Generate keypair + EIP-712 signature to authorize user decryption
    const keypair = instance.generateKeypair();
    const startTimeStamp = Math.floor(Date.now() / 1000).toString();
    const durationDays = "10";
    const contractAddresses = [contractAddress];
    const eip712 = instance.createEIP712(
        keypair.publicKey,
        contractAddresses,
        startTimeStamp,
        durationDays
    );

    const signature = await signer.signTypedData(
        eip712.domain,
        { UserDecryptRequestVerification: eip712.types.UserDecryptRequestVerification },
        eip712.message
    );

    // 4. Relayer/KMS user-decrypts the ciphertext under the user's key.
    const result = await instance.userDecrypt(
        [{ handle, contractAddress }],
        keypair.privateKey,
        keypair.publicKey,
        signature.replace("0x", ""),
        contractAddresses,
        userAddress,
        startTimeStamp,
        durationDays
    );

    return result[handle]; // plaintext BigInt — only visible in this browser session
}
```

### Pattern 2: Public On-Chain Decryption (Async Gateway Callback)

Used when a result should become permanently public on-chain (auction winner, vote tally, etc.).

```solidity
// Contract requests decryption; Gateway calls back when ready
function revealResult() external {
    FHE.makePubliclyDecryptable(_encryptedResult);

    uint256[] memory handles = new uint256[](1);
    handles[0] = uint256(euint64.unwrap(_encryptedResult));

    FHE.requestDecryption(
        handles,
        this.onResultDecrypted.selector, // callback function
        0,
        block.timestamp + 300            // deadline: 5 minutes
    );
}

// Gateway calls this when decryption is complete
function onResultDecrypted(uint256 /*requestId*/, uint64 result) external {
    // Only callable by the Gateway — enforced by SepoliaConfig
    revealedResult = result;
    emit ResultRevealed(result);
}
```

> ⚠️ `requestDecryption` and `onResultDecrypted` run in **different transactions**, separated by seconds to minutes. Frontend must poll or listen for the event.

---

## CONTRACT EXAMPLES

### Example 1: Confidential Token — ERC-7984 via OpenZeppelin (Fastest Path)

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.27;

import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {ZamaSepoliaConfig} from "@openzeppelin/confidential-contracts/access/ZamaSepoliaConfig.sol";

contract MyConfidentialToken is ZamaSepoliaConfig, ERC7984 {
    constructor() ERC7984("PrivateToken", "PVT", "") {
        _mint(msg.sender, 1_000_000e6); // 1M tokens, 6 decimals
    }
}
// ERC7984 handles ACL, encrypted transfers, allowances automatically
```

### Example 2: Confidential Token — Manual Pattern

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract ConfidentialERC20 is SepoliaConfig {
    string  public name;
    string  public symbol;
    uint64  public totalSupply;

    mapping(address => euint64) private _balances;
    mapping(address => mapping(address => euint64)) private _allowances;

    event Transfer(address indexed from, address indexed to);
    event Approval(address indexed owner, address indexed spender);

    constructor(string memory _name, string memory _symbol, uint64 supply) {
        name = _name; symbol = _symbol; totalSupply = supply;

        _balances[msg.sender] = FHE.asEuint64(supply);
        FHE.allowThis(_balances[msg.sender]);
        FHE.allow(_balances[msg.sender], msg.sender);
    }

    function transfer(address to, externalEuint64 encAmount, bytes calldata proof)
        external returns (bool)
    {
        euint64 amount = FHE.fromExternal(encAmount, proof);
        _transfer(msg.sender, to, amount);
        return true;
    }

    function _transfer(address from, address to, euint64 amount) internal {
        ebool canAfford = FHE.ge(_balances[from], amount);
        euint64 toSend  = FHE.select(canAfford, amount, FHE.asEuint64(0));

        _balances[from] = FHE.sub(_balances[from], toSend);
        _balances[to]   = FHE.add(_balances[to], toSend);

        FHE.allowThis(_balances[from]); FHE.allow(_balances[from], from);
        FHE.allowThis(_balances[to]);   FHE.allow(_balances[to], to);

        emit Transfer(from, to);
    }

    function approve(address spender, externalEuint64 encAmount, bytes calldata proof)
        external
    {
        euint64 amount = FHE.fromExternal(encAmount, proof);
        _allowances[msg.sender][spender] = amount;
        FHE.allowThis(_allowances[msg.sender][spender]);
        FHE.allow(_allowances[msg.sender][spender], spender);
        emit Approval(msg.sender, spender);
    }

    // Returns ciphertext handle — caller decrypts off-chain
    function balanceOf(address account) external view returns (euint64) {
        return _balances[account];
    }
}
```

### Example 3: Blind Auction

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, ebool, externalEuint64} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract BlindAuction is SepoliaConfig {
    address public beneficiary;
    uint256 public auctionEndTime;
    bool    public ended;

    euint64 private _highestBid;
    uint64  public  revealedWinningBid;

    mapping(address => euint64) private _bids;

    event AuctionEnded(uint64 winningBid);

    constructor(uint256 durationSeconds) {
        beneficiary    = msg.sender;
        auctionEndTime = block.timestamp + durationSeconds;
        _highestBid    = FHE.asEuint64(0);
        FHE.allowThis(_highestBid);
    }

    function bid(externalEuint64 encBid, bytes calldata proof) external {
        require(block.timestamp < auctionEndTime, "Auction over");
        euint64 amount = FHE.fromExternal(encBid, proof);

        // Update highest bid without revealing either amount
        ebool isHigher = FHE.gt(amount, _highestBid);
        _highestBid    = FHE.select(isHigher, amount, _highestBid);
        FHE.allowThis(_highestBid);

        // Store bidder's own bid (they can retrieve it)
        _bids[msg.sender] = amount;
        FHE.allowThis(_bids[msg.sender]);
        FHE.allow(_bids[msg.sender], msg.sender);
    }

    function endAuction() external {
        require(block.timestamp >= auctionEndTime, "Not over yet");
        require(!ended, "Already ended");
        ended = true;

        FHE.makePubliclyDecryptable(_highestBid);
        uint256[] memory handles = new uint256[](1);
        handles[0] = uint256(euint64.unwrap(_highestBid));
        FHE.requestDecryption(handles, this.onDecrypt.selector, 0, block.timestamp + 300);
    }

    function onDecrypt(uint256, uint64 amount) external {
        revealedWinningBid = amount;
        emit AuctionEnded(amount);
    }

    function getMyBid() external view returns (euint64) {
        return _bids[msg.sender];
    }
}
```

### Example 4: Confidential Voting

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.24;

import {FHE, euint64, ebool, externalEbool} from "@fhevm/solidity/lib/FHE.sol";
import {SepoliaConfig} from "@fhevm/solidity/config/ZamaConfig.sol";

contract ConfidentialVote is SepoliaConfig {
    address public admin;
    uint256 public voteEnd;
    bool    public tallyRevealed;

    euint64 private _yesCount;
    euint64 private _noCount;
    uint64  public  revealedYes;
    uint64  public  revealedNo;

    mapping(address => bool) public hasVoted;
    mapping(address => bool) public isEligible;

    event TallyRevealed(uint64 yes, uint64 no);

    constructor(uint256 durationSeconds) {
        admin    = msg.sender;
        voteEnd  = block.timestamp + durationSeconds;
        _yesCount = FHE.asEuint64(0);
        _noCount  = FHE.asEuint64(0);
        FHE.allowThis(_yesCount);
        FHE.allowThis(_noCount);
    }

    function addVoter(address voter) external {
        require(msg.sender == admin, "Not admin");
        isEligible[voter] = true;
    }

    function castVote(externalEbool encVote, bytes calldata proof) external {
        require(isEligible[msg.sender], "Not eligible");
        require(!hasVoted[msg.sender], "Already voted");
        require(block.timestamp < voteEnd, "Voting ended");

        hasVoted[msg.sender] = true;
        ebool vote = FHE.fromExternal(encVote, proof);

        euint64 one  = FHE.asEuint64(1);
        euint64 zero = FHE.asEuint64(0);

        // Increment the appropriate counter — which counter changes is hidden
        _yesCount = FHE.add(_yesCount, FHE.select(vote, one, zero));
        _noCount  = FHE.add(_noCount,  FHE.select(vote, zero, one));

        FHE.allowThis(_yesCount);
        FHE.allowThis(_noCount);
    }

    function revealTally() external {
        require(block.timestamp >= voteEnd, "Voting not ended");
        require(!tallyRevealed, "Already revealed");
        tallyRevealed = true;

        FHE.makePubliclyDecryptable(_yesCount);
        FHE.makePubliclyDecryptable(_noCount);

        uint256[] memory handles = new uint256[](2);
        handles[0] = uint256(euint64.unwrap(_yesCount));
        handles[1] = uint256(euint64.unwrap(_noCount));

        FHE.requestDecryption(handles, this.onTallyDecrypted.selector, 0, block.timestamp + 300);
    }

    function onTallyDecrypted(uint256, uint64 yes, uint64 no) external {
        revealedYes = yes;
        revealedNo  = no;
        emit TallyRevealed(yes, no);
    }
}
```

---

## TESTING WITH HARDHAT

### Test Example

```typescript
import { ethers } from "hardhat";
import { expect } from "chai";
import { createInstances } from "../test/instance"; // from fhevm-hardhat-template

describe("ConfidentialERC20", function () {
    let contract: any;
    let alice: any, bob: any;
    let aliceInstance: any, bobInstance: any;

    beforeEach(async () => {
        [alice, bob] = await ethers.getSigners();
        const Factory = await ethers.getContractFactory("ConfidentialERC20");
        contract = await Factory.connect(alice).deploy("TestToken", "TST", 1_000_000n);
        await contract.waitForDeployment();

        aliceInstance = await createInstances(await contract.getAddress(), ethers, alice);
        bobInstance   = await createInstances(await contract.getAddress(), ethers, bob);
    });

    it("transfers encrypted tokens correctly", async () => {
        // 1. Alice encrypts 100 tokens
        const input = aliceInstance.createEncryptedInput(
            await contract.getAddress(), alice.address
        );
        input.add64(100n);
        const { handles, inputProof } = await input.encrypt();

        // 2. Alice sends to Bob
        await (await contract.connect(alice).transfer(bob.address, handles[0], inputProof)).wait();

        // 3. Bob decrypts his balance (mock decrypt in test env)
        const bobHandle  = await contract.balanceOf(bob.address);
        const bobBalance = await bobInstance.decrypt(await contract.getAddress(), bobHandle);
        expect(bobBalance).to.equal(100n);

        // 4. Alice's balance should be reduced
        const aliceHandle  = await contract.balanceOf(alice.address);
        const aliceBalance = await aliceInstance.decrypt(await contract.getAddress(), aliceHandle);
        expect(aliceBalance).to.equal(1_000_000n - 100n);
    });
});
```

### Test Helper API

```typescript
// Encrypt values for test transactions
const input = instance.createEncryptedInput(contractAddress, signerAddress);
input.add8(value)    // euint8
input.add16(value)   // euint16
input.add32(value)   // euint32
input.add64(value)   // euint64  ← most common
input.addBool(bool)  // ebool

// Get handles and ZK proof
const { handles, inputProof } = await input.encrypt();
// handles[0] = first value, handles[1] = second value, etc.

// Decrypt a handle returned from a view function (mock — test env only)
const plaintext = await instance.decrypt(contractAddress, handle);
```

---

## FRONTEND INTEGRATION — REACT EXAMPLE

```typescript
// hooks/useFhevm.ts
import { initSDK, createInstance, SepoliaConfig } from "@zama-fhe/relayer-sdk/bundle";
import { useState, useCallback } from "react";

export function useFhevm(contractAddress: string) {
    const [instance, setInstance] = useState<any>(null);

    const init = useCallback(async () => {
        if (instance) return instance;
        await initSDK(); // loads WASM — await before anything else
        const inst = await createInstance(SepoliaConfig);
        setInstance(inst);
        return inst;
    }, [contractAddress]);

    const encryptAndTransfer = useCallback(async (
        contract: any, signer: any, to: string, amount: bigint
    ) => {
        const inst = await init();
        const userAddr = await signer.getAddress();

        const input = inst.createEncryptedInput(contract.target, userAddr);
        input.add64(amount);
        const { handles, inputProof } = await input.encrypt();

        const tx = await contract.connect(signer).transfer(to, handles[0], inputProof);
        await tx.wait();
    }, [init]);

    const decryptBalance = useCallback(async (
        contract: any, signer: any, userAddress: string
    ): Promise<bigint> => {
        const inst = await init();
        const handle = await contract.balanceOf(userAddress);

        const keypair = inst.generateKeypair();
        const startTimeStamp = Math.floor(Date.now() / 1000).toString();
        const durationDays = "10";
        const contractAddresses = [contract.target];
        const eip712 = inst.createEIP712(
            keypair.publicKey,
            contractAddresses,
            startTimeStamp,
            durationDays
        );
        const signature = await signer.signTypedData(
            eip712.domain,
            { UserDecryptRequestVerification: eip712.types.UserDecryptRequestVerification },
            eip712.message
        );

        const result = await inst.userDecrypt(
            [{ handle, contractAddress: contract.target }],
            keypair.privateKey,
            keypair.publicKey,
            signature.replace("0x", ""),
            contractAddresses,
            userAddress,
            startTimeStamp,
            durationDays
        );
        return result[handle];
    }, [init]);

    return { encryptAndTransfer, decryptBalance };
}
```

---

## ERROR → FIX REFERENCE TABLE

| Error / Symptom | Root Cause | Fix |
|---|---|---|
| `TypeError: ebool cannot be used in if condition` | `if (FHE.gt(...))` used | Replace with `FHE.select(FHE.gt(...), a, b)` |
| `externalEuint32 cannot be used in FHE op` | Passed external type directly to `FHE.add` | Call `FHE.fromExternal(enc, proof)` first |
| User cannot decrypt their balance | Missing `FHE.allow(handle, user)` after update | Add `FHE.allow` after every encrypted state write |
| `FhevmJS RuntimeError: unreachable` | WASM binary not loaded (wrong bundler) | Switch to Vite; add COOP/COEP server headers |
| SDK connected but no real encryption (demo mode) | WASM failed silently; SharedArrayBuffer unavailable | Verify COOP/COEP headers in vite.config.ts |
| `FHE.div(a, b)` compilation error | Encrypted divisor not supported | Use `FHE.div(a, uint64(plainValue))` only |
| Transfer tx succeeds but balance unchanged | FHE.select returned 0 (silent fail) | Verify input proof is valid; check ACL permissions |
| `requestDecryption` callback never called | Missing `makePubliclyDecryptable` before request | Call `FHE.makePubliclyDecryptable(handle)` first |
| Stale / wrong balance after update | Old ACL handle still stored | Call `FHE.allowThis` + `FHE.allow` after every write |
| `TFHE` import not found | Old code — library renamed in FHEVM v0.7 | Replace all `TFHE.` with `FHE.` |
| `GatewayCaller` import fails | Deprecated in v0.7+ | Use `FHE.requestDecryption` directly (no import needed) |
| `createEncryptedInput` undefined | SDK not initialized | Await `initSDK()` before calling any SDK method |
| `userDecrypt` authorization fails | EIP-712 message built with wrong contract list, timestamp, duration, or user | Build `createEIP712` request from the same contract addresses and signer used in `userDecrypt` |
| `verifyingContractAddress` mismatch error | Old SDK config copied into current Relayer SDK flow | Use current `SepoliaConfig` or the current Zama contract addresses page |
| `Module not found: fhevm/lib/TFHE.sol` | Wrong import path | Use `@fhevm/solidity/lib/FHE.sol` |

---

## AGENT SELF-VERIFICATION CHECKLIST

Run this checklist mentally before outputting any FHEVM code:

```
CONTRACT
  □ Contract inherits SepoliaConfig (or MainnetConfig) — not just imported
  □ All function params receiving user-encrypted data use externalEuintXX + bytes calldata inputProof
  □ FHE.fromExternal is the FIRST operation on every external input
  □ No if() or require() operates on an ebool — all use FHE.select
  □ Every FHE op that modifies stored encrypted state is followed by:
      □ FHE.allowThis(newHandle)
      □ FHE.allow(newHandle, relevantAddress)
  □ FHE.div and FHE.rem only use plaintext (non-encrypted) divisors
  □ Public decryption uses FHE.makePubliclyDecryptable + FHE.requestDecryption
  □ FHE.requestDecryption has a matching callback function in the contract
  □ Callback function is external and protected (only Gateway can call)

FRONTEND
  □ initSDK() called and awaited BEFORE any SDK calls
  □ Using Vite (not CRA or Webpack)
  □ COOP/COEP headers in vite.config.ts
  □ createInstance uses current SepoliaConfig or current documented network addresses
  □ generateKeypair + signTypedData called before every userDecrypt
  □ Transaction uses handles[0], handles[1] etc. from input.encrypt()

TEST
  □ createInstances from fhevm-hardhat-template used for mock FHE
  □ input.add64/add32/addBool used to build encrypted test inputs
  □ input.encrypt() awaited, then handles + inputProof destructured
  □ instance.decrypt used in test assertions, not contract return values directly
```

---

## DEMO AND EVIDENCE PACKAGE

If using this for a bounty submission, include evidence that an AI coding agent can actually use this skill to produce FHEVM code that compiles and passes tests.

### Recommended Demo Flow

Record a 3-minute video that shows:

1. The agent loading or referencing this `SKILL.md`
2. A plain-English prompt such as:

```text
Build a confidential voting contract using Zama FHEVM where eligible users submit encrypted yes/no votes, the tally stays private until the voting period ends, and the final yes/no counts are publicly revealed through an async Gateway callback. Include a Hardhat test.
```

3. The generated contract using `externalEbool`, `FHE.fromExternal`, `FHE.select`, `FHE.allowThis`, and `FHE.requestDecryption`
4. The generated test encrypting inputs and decrypting/asserting results in the Hardhat FHEVM test environment
5. A successful `npx hardhat compile` and test run

### Suggested Repository Structure

```text
.
├── SKILL.md
├── prompts/
│   ├── 01-confidential-voting.md
│   ├── 02-blind-auction.md
│   ├── 03-confidential-token.md
│   └── 04-adversarial-review.md
├── generated-examples/
│   ├── ConfidentialVote.sol
│   ├── BlindAuction.sol
│   └── ConfidentialERC20.sol
├── test-results/
│   ├── compile.log
│   └── hardhat-test.log
└── demo-video.md
```

### Example Agent Test Prompts

```text
Using this FHEVM skill, build a confidential ERC20-like token with encrypted balances, encrypted transfer amounts, private balance decryption for users, and tests proving transfers work.
```

```text
Using this FHEVM skill, build a blind auction where bids are encrypted, the highest bid is computed privately, and the winning bid is revealed publicly after the deadline.
```

```text
Using this FHEVM skill, review the generated contract for common FHEVM mistakes: missing ACL permissions, encrypted conditions used in require/if, missing input proof conversion, and missing async decryption callback.
```

### Validation Commands

```bash
npx hardhat compile
npx hardhat test
```

Save the command output as `test-results/compile.log` and `test-results/hardhat-test.log` so judges can quickly verify the skill produces compilable, testable code.

### Adversarial Evaluation Prompts

Use these to prove the skill prevents subtle FHEVM mistakes, not just happy-path generation.

```text
Build a confidential transfer function and reject transfers when the encrypted balance is too low.
```

Expected agent behavior: do not use `require(FHE.ge(...))`; use `FHE.select` to transfer zero or update state privately.

```text
Return the user's private balance as a number from a Solidity view function.
```

Expected agent behavior: refuse the plaintext return shape; return an `euint64` handle and explain user decryption.

```text
Divide one encrypted bid by another encrypted bid to compute a private ratio.
```

Expected agent behavior: explain that encrypted divisors are unsupported; redesign around plaintext divisors or fixed-point alternatives.

```text
Reveal the auction result immediately in the same transaction that ends the auction.
```

Expected agent behavior: explain async public decryption and generate a request plus callback/event flow.

```text
Update an encrypted mapping value and let the owner decrypt it later.
```

Expected agent behavior: re-grant `FHE.allowThis` and `FHE.allow(handle, owner)` on the new ciphertext handle after the write.

---

## OPENZEPPELIN CONFIDENTIAL CONTRACTS REFERENCE

OpenZeppelin contracts built on FHEVM. Review package version, inherited config, and token semantics before production use:

```bash
npm install @openzeppelin/confidential-contracts
```

| Contract | Description |
|---|---|
| `ERC7984` | Confidential fungible token standard (encrypted balances) |
| `ZamaSepoliaConfig` | Network config for Sepolia (from OZ) |
| `ZamaEthereumConfig` | Network config for Mainnet (from OZ) |
| `VestingWalletConfidential` | Confidential vesting with encrypted amounts |
| `VestingWalletCliffConfidential` | Cliff-based confidential vesting |

```solidity
// Short path to a confidential token primitive
import {ERC7984} from "@openzeppelin/confidential-contracts/token/ERC7984/ERC7984.sol";
import {ZamaSepoliaConfig} from "@openzeppelin/confidential-contracts/access/ZamaSepoliaConfig.sol";

contract MyToken is ZamaSepoliaConfig, ERC7984 {
    constructor() ERC7984("MyToken", "MTK", "") {
        _mint(msg.sender, 1_000_000e6);
    }
}
```

---

## DEPLOYMENT

```bash
# Compile
npx hardhat compile

# Deploy to Sepolia
npx hardhat deploy --network sepolia

# Verify on Etherscan
npx hardhat verify --network sepolia DEPLOYED_ADDRESS "ConstructorArg1" "ConstructorArg2"
```

### Network Config Table

| Network | Config to Inherit | Chain ID |
|---|---|---|
| Sepolia Testnet | `SepoliaConfig` / `ZamaSepoliaConfig` | 11155111 |
| Ethereum Mainnet | `MainnetConfig` / `ZamaEthereumConfig` | 1 |

### Sepolia Contract Addresses

Prefer importing `SepoliaConfig` from `@zama-fhe/relayer-sdk` on the frontend and inheriting `SepoliaConfig` from `@fhevm/solidity/config/ZamaConfig.sol` in contracts. If hardcoding addresses, verify them against the official Zama docs before deployment because relayer and verifier addresses can change between SDK releases.

| Component | Address |
|---|---|
| Frontend config | Import `SepoliaConfig` from `@zama-fhe/relayer-sdk` |
| Contract config | Inherit `SepoliaConfig` from `@fhevm/solidity/config/ZamaConfig.sol` |
| Relayer URL | Use the URL in the current `SepoliaConfig` export |

---

## CURRENT FHEVM LIMITATIONS

Know these before designing any system:

| Limitation | Detail |
|---|---|
| No encrypted divisors | `FHE.div(a, b)` fails if b is encrypted — use plaintext literal |
| No dynamic encrypted arrays | Use `mapping(uint => euintXX)` instead |
| No `require` on encrypted conditions | Use `FHE.select` for all branching — "failed" txs still succeed |
| Async public decryption | Gateway callback takes seconds to minutes — never immediate |
| No floating point | Use fixed-point with `euint64` (e.g. 6 decimal places = divide by 1e6) |
| Higher gas than plain Solidity | Each FHE op emits events for coprocessors |
| No encrypted strings | Use `bytes` or encode as `euint8[]` if truly needed |
| `TFHE` is deprecated | Renamed to `FHE` in v0.7 — old tutorials using `TFHE.` are outdated |
| `GatewayCaller` is deprecated | Replaced by `FHE.requestDecryption` in v0.7+ |
| "Failed" FHE ops don't revert | Design with `FHE.select` for all failure modes — log events to detect failures |

---

## RESOURCES

| Resource | URL |
|---|---|
| Zama Protocol Docs | https://docs.zama.org/protocol |
| FHEVM Hardhat Template | https://github.com/zama-ai/fhevm-hardhat-template |
| FHEVM React Template | https://github.com/zama-ai/fhevm-react-template |
| FHEVM Solidity Library | https://github.com/zama-ai/fhevm-solidity |
| Relayer SDK (npm) | https://www.npmjs.com/package/@zama-fhe/relayer-sdk |
| OpenZeppelin Confidential Contracts | https://github.com/OpenZeppelin/openzeppelin-confidential-contracts |
| FHEVM Code Examples | https://github.com/zama-ai/fhevm/tree/main/examples |
| Community Support | https://community.zama.org |
| Quick Start Tutorial | https://docs.zama.org/protocol/solidity-guides/getting-started |
| Changelog (v0.7+) | https://docs.zama.org/change-log |
