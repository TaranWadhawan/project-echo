# Project ECHO: Cryptography & Security Protocols

## Table of Contents

1. [Cryptographic Primitives](#cryptographic-primitives)
2. [Key Management](#key-management)
3. [Digital Signatures](#digital-signatures)
4. [Encryption](#encryption)
5. [Hashing](#hashing)
6. [Challenge-Response Protocol](#challenge-response-protocol)
7. [Post-Quantum Readiness](#post-quantum-readiness)
8. [Threat Resistance](#threat-resistance)

---

## Cryptographic Primitives

### Algorithm Selection

| Purpose | Algorithm | Key Size | Rationale |
|---------|-----------|----------|----------|
| Signatures | Ed25519 | 256 bits | Deterministic, fast, safe defaults |
| Key Exchange | Curve25519 | 256 bits | Elliptic curve, high security/performance |
| Encryption | ChaCha20-Poly1305 | 256 bits | AEAD, resistant to timing attacks |
| Hash (state) | SHA3-256 | 256 bits | Keccak-based, standard, secure |
| Hash (fast) | BLAKE3 | 256 bits | Modern, parallelizable, faster than SHA3 |
| TLS | TLS 1.3 | Variable | Latest standard, removes weak ciphers |
| Transport Security | AES-256-GCM | 256 bits | AEAD, hardware-accelerated |

### Implementation Libraries

```cpp
#include <openssl/evp.h>        // OpenSSL (TLS, AES, SHA3)
#include <sodium.h>             // libsodium (Ed25519, ChaCha20, Blake2b)
#include <blake3.h>             // BLAKE3 hashing
```

---

## Key Management

### Key Derivation

**Master Key Generation:** All keys are derived from a single master seed using HKDF-SHA256.

```cpp
struct KeyDerivation {
    // Master seed (never stored, generated on startup)
    uint8_t master_seed[32];
    
    // Derive specific key material
    // domain = "echo.player.signing", "echo.validator.stake", etc.
    std::vector<uint8_t> derive_key(
        const std::string& domain,
        uint64_t index = 0
    ) {
        uint8_t info[64];
        memcpy(info, domain.data(), domain.size());
        memcpy(info + domain.size(), &index, sizeof(index));
        
        uint8_t derived_key[32];
        HKDF_SHA256(
            derived_key, 32,           // Output
            master_seed, 32,           // Input key material
            nullptr, 0,                // Salt (none)
            info, sizeof(info)         // Info
        );
        
        return std::vector<uint8_t>(derived_key, derived_key + 32);
    }
};
```

### Key Hierarchies

```
Master Seed
├── Player Keys
│   ├── Signing Key (for transactions)
│   ├── Encryption Key (for private messages)
│   └── Backup Key (recovery)
├── Validator Keys
│   ├── Block Proposal Key
│   ├── Voting Key
│   └── Slashing Key (for penalties)
└── Service Keys
    ├── TLS Certificate Key
    ├── Session Keys
    └── Webhook Signing Key
```

### Key Storage

**Client-Side:**
- Keys stored in **encrypted localStorage** (browser) or **keychain** (native)
- Master seed never persisted (regenerated from password + salt)
- Optional hardware wallet support (hardware signing device)

**Server-Side:**
- Keys in **encrypted at-rest** using AES-256-GCM
- **Key rotation** every 90 days
- **Hardware security modules (HSM)** for validator keys
- Separate keys per region/datacenter

---

## Digital Signatures

### Ed25519 Signing

**Properties:**
- Deterministic (same message always produces same signature)
- Fast verification (suitable for blockchain)
- Small keys (32 bytes pubkey, 64 bytes signature)
- No randomness source needed

```cpp
class SigningKeyPair {
    uint8_t private_key[32];  // Secret
    uint8_t public_key[32];   // Public
    
 public:
    // Create signature
    Signature sign(const Bytes& message) const {
        uint8_t sig[64];
        crypto_sign_detached(
            sig, nullptr,
            message.data(), message.size(),
            private_key
        );
        return Signature(sig, 64);
    }
    
    // Verify signature (static method)
    static bool verify(
        const PublicKey& pubkey,
        const Signature& sig,
        const Bytes& message
    ) {
        return crypto_sign_open_detached(
            sig.data(),
            message.data(), message.size(),
            pubkey.data()
        ) == 0;
    }
};
```

### Transaction Signing Flow

```
1. Player creates transaction
   tx = {
       actor: player_id,
       type: "Transfer",
       data: { to_id, amount },
       timestamp: now(),
       nonce: next_nonce()
   }

2. Compute transaction hash
   tx_hash = SHA3_256(serialize(tx))

3. Sign with Ed25519
   signature = Ed25519_sign(tx_hash, player_private_key)

4. Finalize transaction
   signed_tx = { tx, signature }

5. Send to server
   network.submit_transaction(signed_tx)
```

### Nonce Management (Anti-Replay)

```cpp
struct NonceTracker {
    std::unordered_map<UUID, uint64_t> player_nonces;
    
    bool validate_and_consume_nonce(
        const UUID& player_id,
        uint64_t nonce
    ) {
        auto it = player_nonces.find(player_id);
        
        if (it == player_nonces.end()) {
            // First transaction from this player
            player_nonces[player_id] = nonce + 1;
            return true;
        }
        
        if (nonce != it->second) {
            // Nonce doesn't match expected sequence
            return false;
        }
        
        it->second++;  // Increment for next transaction
        return true;
    }
};
```

---

## Encryption

### ChaCha20-Poly1305 (Client-to-Client)

For private messages between players:

```cpp
class EncryptedMessage {
    uint8_t nonce[12];
    uint8_t ciphertext[MAX_MSG_SIZE];
    uint8_t tag[16];  // Authentication tag
    
 public:
    EncryptedMessage encrypt(
        const Bytes& plaintext,
        const SharedSecret& key
    ) {
        // Generate random nonce
        RAND_bytes(nonce, 12);
        
        // Encrypt and authenticate
        chacha20_poly1305_encrypt(
            ciphertext, tag,
            plaintext.data(), plaintext.size(),
            key.data(), 32,
            nonce
        );
        
        return *this;
    }
};
```

### AES-256-GCM (Transport Security)

For QUIC record encryption at network layer (handled by OpenSSL/TLS 1.3):

```
Client Connection
    |
    v
TLS 1.3 Handshake (ephemeral keys)
    |
    v
Session Key Derivation
    |
    v
AES-256-GCM Encryption (per record)
    |
    v
QUIC Transport
```

---

## Hashing

### SHA3-256 (Blockchain State)

Used for blockchain hashes (block headers, merkle roots, transaction IDs):

```cpp
Hash compute_transaction_id(const Transaction& tx) {
    // Hash is deterministic and canonical
    Bytes serialized = serialize(tx);
    uint8_t hash[32];
    
    SHA3_256(
        serialized.data(), serialized.size(),
        hash
    );
    
    return Hash(hash, 32);
}

Hash compute_state_root(const World& world) {
    // Build merkle tree of all entities
    MerkleTree tree;
    
    for (const auto& entity : world.entities) {
        tree.insert(entity.id, serialize(entity));
    }
    
    return tree.root();
}
```

### BLAKE3 (Fast Hashing)

Used for non-consensus hashes (caching, deduplication):

```cpp
Hash fast_hash(const Bytes& data) {
    blake3_hasher hasher;
    blake3_hasher_init(&hasher);
    blake3_hasher_update(&hasher, data.data(), data.size());
    
    uint8_t out[32];
    blake3_hasher_finalize(&hasher, out);
    
    return Hash(out, 32);
}
```

---

## Challenge-Response Protocol

### Purpose

Provide optional **useful computational challenges** instead of meaningless hash puzzles. Validators can prove they performed legitimate work.

### Challenge Structure

```cpp
struct ComputationalChallenge {
    UUID challenge_id;
    
    // Workload type
    enum class Type {
        LinearSystemSolve,      // Solve Ax=b for random matrix A
        GraphPathfinding,       // Find shortest path in random graph
        PolynomialFactorization, // Factor polynomial over finite field
        MatrixMultiplication,   // Compute C = A*B for large matrices
    } type;
    
    // Challenge data
    Bytes input_data;           // Problem instance
    uint64_t difficulty;        // Computation budget (CPU cycles)
    
    // Verification
    Hash expected_output_hash;  // Commitment to correct answer
    uint64_t timestamp;         // When challenge was issued
    
    // Anti-replay
    UUID validator_id;          // Who is solving this
    uint64_t nonce;             // One-time use
};

struct ChallengeResponse {
    UUID challenge_id;
    Bytes solution;              // The computed result
    uint64_t compute_time_ms;   // How long it took
    Signature proof_signature;  // Signed by validator
};
```

### Verification

```cpp
bool verify_challenge_response(
    const ComputationalChallenge& challenge,
    const ChallengeResponse& response
) {
    // 1. Verify signature
    if (!verify_signature(challenge.validator_id, response.proof_signature)) {
        return false;
    }
    
    // 2. Verify solution correctness
    Hash computed_hash = compute_challenge_hash(
        challenge.type,
        challenge.input_data,
        response.solution
    );
    
    if (computed_hash != challenge.expected_output_hash) {
        return false;  // Wrong answer
    }
    
    // 3. Verify reasonable compute time
    uint64_t expected_min_time = challenge.difficulty / CPU_SPEED_ESTIMATE;
    if (response.compute_time_ms < expected_min_time * 0.9) {
        return false;  // Too fast, probably cheating
    }
    
    return true;
}
```

### Resistance to Attacks

**Precomputation:** Challenge uses random instance each time
**Parallelization:** Workload is inherently sequential or difficult to parallelize
**GPU/ASIC:** Workload uses general computation (not specialized hardware)
**Replay:** Nonce prevents same challenge being used twice
**Impersonation:** Signature proves who solved it

---

## Post-Quantum Readiness

### Migration Path

**Phase 1 (Current):** Ed25519 + Curve25519
**Phase 2 (Year 2):** Hybrid signatures (Ed25519 + Dilithium)
**Phase 3 (Year 3):** Full migration to post-quantum

### Candidate Algorithms

```cpp
// NIST Post-Quantum Finalists

// Signatures
struct PostQuantumSignature {
    // Dilithium-3 (recommended for mid-range security)
    enum class Algorithm {
        Dilithium2,   // NIST security level 2
        Dilithium3,   // NIST security level 3
        Dilithium5,   // NIST security level 5
        Falcon,       // Lattice-based alternative
    };
};

// Key Encapsulation (for session setup)
struct PostQuantumKEM {
    enum class Algorithm {
        Kyber512,     // NIST security level 1
        Kyber768,     // NIST security level 3
        Kyber1024,    // NIST security level 5
    };
};
```

### Hybrid Approach

```cpp
struct HybridSignature {
    Signature classical;           // Ed25519 signature
    Signature post_quantum;        // Dilithium signature
};

bool verify_hybrid_signature(
    const PublicKey& classical_key,
    const PublicKey& pq_key,
    const HybridSignature& sig,
    const Bytes& message
) {
    // Both must verify
    return verify_ed25519(classical_key, sig.classical, message) &&
           verify_dilithium(pq_key, sig.post_quantum, message);
}
```

---

## Threat Resistance

### Attack Vectors & Mitigations

| Attack | Threat | Mitigation |
|--------|--------|------------|
| **Replay** | Attacker reuses signed transaction | Nonce tracking per actor |
| **Man-in-the-Middle** | Intercept & modify message | TLS 1.3 + QUIC |
| **Signature Forgery** | Create fake transaction | Ed25519 is deterministic, hard to forge |
| **Key Compromise** | Attacker steals private key | Slashing, reputation damage, stake loss |
| **Double-Spending** | Submit two conflicting txs | Server mempool, consensus finality |
| **Sybil Attack** | Create many fake identities | Stake requirement (32 tokens min) |
| **Eclipse Attack** | Isolate node from network | Multiple peer connections, gossip |
| **DDoS** | Flood network with garbage | Rate limiting, proof-of-work for new peers |
| **Timing Attack** | Extract key from timing differences | Constant-time crypto (libsodium guarantees) |

### Cryptographic Guarantees

```
✅ Authentication: Ed25519 signatures prove identity
✅ Integrity: Hashing detects tampering
✅ Confidentiality: AES-256-GCM protects private data
✅ Non-repudiation: Signatures are cryptographically binding
✅ Replay Protection: Nonce + timestamp + signature
✅ Perfect Forward Secrecy: TLS 1.3 ephemeral keys
✅ Quantum Resistance: Path to post-quantum migration
```

---

## Cryptographic Implementation Checklist

- ✅ Use libsodium for all cryptographic operations
- ✅ Never implement crypto from scratch
- ✅ Constant-time comparisons for sensitive data
- ✅ Secure random number generation (getrandom/CryptGenRandom)
- ✅ Key material never logged or displayed
- ✅ Regular security audits
- ✅ Dependency scanning for vulnerabilities
- ✅ Hardware security module support for validators
- ✅ Key rotation policies
- ✅ Post-quantum migration roadmap

---

All cryptographic operations prioritize **correctness, auditability, and future-proofing**.
