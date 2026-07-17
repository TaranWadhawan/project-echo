# Project ECHO: Blockchain Design & Consensus Algorithm

## Table of Contents

1. [Hybrid Consensus Model](#hybrid-consensus-model)
2. [Block Structure](#block-structure)
3. [Transaction Types](#transaction-types)
4. [Smart Contracts](#smart-contracts)
5. [State Proofs & Merkle Trees](#state-proofs--merkle-trees)
6. [Fork Resolution](#fork-resolution)
7. [Validator Management](#validator-management)

---

## Hybrid Consensus Model

Project ECHO uses a sophisticated hybrid consensus combining four mechanisms:

### Consensus Formula

```
Validator Weight = (0.40 × PoS) + (0.20 × PoA) + (0.20 × Reputation) + (0.20 × Useful Work)

Block Finality: Weight >= 0.667 (66.7% consensus threshold)
```

### Components

#### 1. Proof of Stake (40%)

**Mechanism:** Validators stake tokens for voting power.

```cpp
struct StakeInfo {
    UUID validator_id;
    uint64_t staked_amount;     // Tokens locked
    uint64_t stake_age;         // Seconds since staked
    bool slash_risk;            // At risk for slashing
};

float stake_weight = normalized(staked_amount) * 0.40f;
```

**Slashing Rules:**
- Double-signing: -33% of stake
- Invalid block proposal: -10% of stake
- Offline: -5% per day
- Minimum: 32 tokens to be validator

#### 2. Proof of Authority (20%)

**Mechanism:** Trusted operators (core team, institutional partners) have inherent authority.

```cpp
struct AuthorityInfo {
    UUID authority_id;
    std::string org_name;
    float authority_score;  // 0.0 to 1.0
    uint64_t appointed_at;
};

float authority_weight = authority_score * 0.20f;
```

**Authority Scores:**
- Core team: 0.95
- Institutional validator: 0.75
- Community validator: 0.50

#### 3. Reputation (20%)

**Mechanism:** Validators earn reputation through consistent, non-malicious participation.

```cpp
float reputation_weight = normalized(reputation_score) * 0.20f;

// Reputation increases for:
// - Proposing valid blocks: +0.5
// - Voting correctly: +0.1
// - 1000 consecutive valid blocks: +1.0
//
// Reputation decreases for:
// - Invalid votes: -1.0
// - Missing vote deadline: -0.5
// - Slashing events: -5.0
```

#### 4. Useful Work (20%)

**Mechanism:** Validators contribute to scientific computing earn consensus weight.

```cpp
struct UsefulWorkCredit {
    UUID validator_id;
    std::string workload_type;  // "linear_algebra", "graph_optimization"
    uint64_t compute_hours;     // Verified compute time
    uint64_t last_contribution; // Timestamp
};

float useful_work_weight = normalized(compute_hours) * 0.20f;
```

**Valid Workloads:**
- Mathematical optimization
- Distributed simulations
- ML inference
- Rendering
- Research computations

---

## Block Structure

### Block Header

```cpp
struct BlockHeader {
    // Identification
    uint64_t height;                    // Block number (0-indexed)
    UUID proposer_id;                   // Who proposed this block
    
    // Chain
    Hash previous_block_hash;           // Link to parent
    uint64_t timestamp;                 // Unix seconds
    
    // Content
    Hash merkle_root;                   // Root of transactions
    Hash state_root;                    // Root of world state after block
    
    // Consensus
    uint32_t nonce;                     // Optional PoW nonce
    std::vector<UUID> voters;           // Validators who voted YES
    uint64_t vote_weight;               // Cumulative weight of voters
};

struct Block {
    BlockHeader header;
    std::vector<Transaction> transactions;
    std::vector<Signature> signatures;   // One per voter
    uint64_t size_bytes;
};
```

### Block Size Limits

- **Max transactions per block:** 10,000
- **Max block size:** 5 MB (uncompressed)
- **Max block time:** 6 seconds (after voting)
- **Min block time:** 1 second (to prevent spam)

---

## Transaction Types

### Core Transaction Type Enum

```cpp
enum class TransactionType : uint16_t {
    // === Identity (100-199) ===
    RegisterIdentity = 100,      // Create new avatar
    UpdateProfile = 101,         // Change name, bio, etc.
    LinkAccount = 102,           // Bind external auth
    
    // === Economy (200-299) ===
    Transfer = 200,              // Send credits/resources
    Craft = 201,                 // Start crafting
    TradeOffer = 202,            // Create market listing
    AcceptTrade = 203,           // Buy from market
    
    // === Governance (300-399) ===
    Vote = 300,                  // Cast vote on proposal
    ProposeGovernance = 301,     // Create new proposal
    UpdateTaxPolicy = 302,       // Change tax rates
    
    // === Science (400-499) ===
    SubmitComputeResult = 400,   // Provide proof of work
    PublishResearch = 401,       // Publish findings
    ClaimScientificReward = 402, // Claim discovery bonus
    
    // === Social (500-599) ===
    FormGuild = 500,             // Create organization
    JoinGuild = 501,             // Request membership
    RatePlayer = 502,            // Update reputation
    
    // === World (600-699) ===
    Move = 600,                  // Change position
    SpawnEntity = 601,           // Create object/building
    DespawnEntity = 602,         // Destroy object
    ClaimTerritory = 603,        // Control land
    
    // === Smart Contracts (700-799) ===
    DeployContract = 700,        // Upload WASM code
    CallContract = 701,          // Execute contract function
};
```

---

## Smart Contracts

### Contract VM Architecture

**Engine:** WebAssembly (WASM)

**Rationale:**
- Deterministic execution (same input → same output)
- Efficient verification on all nodes
- Portable and language-agnostic
- Security sandbox (memory-isolated)
- 100% gas-metering possible

---

## State Proofs & Merkle Trees

### Merkle Tree Construction

Every block includes a `state_root` that is the cryptographic hash of all entities in the world after that block's transactions are applied.

**Key Properties:**
- Deterministic: Same state always produces same root
- Verifiable: Any node can verify a state proof
- Compact: Merkle proofs are logarithmic in size
- Incremental: Only changed entities need recomputing

---

## Fork Resolution

### Canonical Chain Selection

**Rule:** The chain with the highest cumulative validator weight is canonical.

This ensures that the most-validated branch is always considered the authoritative history.

---

## Validator Management

### Validator Lifecycle

```
Candidate → Active → Suspended ← (slashing)
              ↓
            Retired (voluntary exit)
```

### Validator Election

**Frequency:** Every 1000 blocks (~100 minutes)

**Process:**
1. Score all validators using hybrid formula
2. Sort by weight
3. Select top N (typically 100-1000)
4. Validators with weight > 0.01 are eligible

---

This blockchain design prioritizes:
✅ **Decentralization** via multiple consensus mechanisms
✅ **Security** via slashing and reputation
✅ **Efficiency** via WASM contracts and merkle proofs
✅ **Fairness** via weighted voting and transparent rules
