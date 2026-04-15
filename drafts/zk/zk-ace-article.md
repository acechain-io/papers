# The Identity Revolution in the Post-Quantum Era: How ZK-ACE and AR-ACE Redesign Blockchain Authorization

> When quantum computing threatens to break existing signature schemes at the root, what the blockchain world needs is not merely larger keys—but a fundamental paradigm shift.

---

## I. The Root of the Problem: The Scale Cost of Post-Quantum Signatures

In 2024, NIST finalized its post-quantum cryptography standards. Lattice-based signature schemes such as ML-DSA (formerly Dilithium) are positioned to replace classical public-key signatures as the primary defense against quantum adversaries. Yet when engineers began seriously considering deploying these schemes on blockchains, a structural contradiction emerged:

**The size of post-quantum signatures is inherently at odds with a blockchain's sensitivity to data.**

The numbers make the tension concrete:

| Scheme | NIST Security Level | Signature Size |
|---|---|---|
| secp256k1 ECDSA (classical) | — | ≈ 71 bytes |
| Ed25519 (classical) | — | 64 bytes |
| ML-DSA-44 | L2 | **2,420 bytes** |
| ML-DSA-65 | L3 | **3,309 bytes** |
| SLH-DSA-128f | L1 | **17,088 bytes** |

This is not a 10% increase. It is a 30–40× jump in magnitude. In a blockchain environment where every transaction must be received, verified, and permanently recorded by every consensus participant, this translates directly: per-transaction authorization data grows from under 100 bytes to several kilobytes.

The problem compounds in rollup architectures. EIP-4844 introduced blob space that is explicitly constrained and economically priced. Large post-quantum signatures compress effective throughput and raise the marginal cost of every transaction included on the base layer.

---

## II. The Intuitive Fix That Doesn't Hold: Stuffing Verification into ZK Circuits

The cryptography community's most natural response was: **move post-quantum signature verification off-chain and into a zero-knowledge proof circuit**.

The logic sounds reasonable. Rather than posting a 2,420-byte signature on-chain, verify it off-chain and submit only a succinct ZK proof—which under Groth16 occupies just 128 bytes.

But this path carries a fundamental problem that is easy to underestimate.

Verifying a lattice-based signature inside a ZK circuit requires implementing high-dimensional algebraic arithmetic as circuit constraints:

- **Number Theoretic Transforms (NTTs)**: over degree-256 polynomial rings in Z_q with q = 2²³ − 2¹³ + 1, each requiring O(n log n) multiplications;
- **Non-native modular arithmetic**: emulating a 23-bit modulus over a ≈254-bit proof-system field, requiring range-check gadgets per limb;
- **Rejection sampling and hint reconstruction**: logic with data-dependent branching.

The structural lower bound for ML-DSA L2 verification inside a ZK circuit is on the order of **millions of R1CS constraints**.

In other words, this approach merely relocates the scalability burden—from on-chain data to off-chain prover computation. The bottleneck persists; it has simply changed address.

---

## III. The Paradigm Shift: Authorization Is Not Signature Verification

This is precisely the core insight behind **ZK-ACE** (Zero-Knowledge Authorization for Cryptographic Entities).

> At the consensus layer, blockchains do not inherently require verification of a specific cryptographic signature object. What consensus requires is assurance that a given transaction was authorized by the correct entity under the system's rules.

A signature is one implementation mechanism for expressing authorization—not the authorization semantics themselves. Traditional designs conflate "verifying a signature" with "verifying authorization," but these two concerns are separable.

ZK-ACE chooses to prove the semantic property of authorization directly: **that an identity consistent with an on-chain commitment has authorized this specific transaction**. Signature objects never appear in the circuit, and never appear on-chain.

---

## IV. How ZK-ACE Works

### The Identity Layer: The Deterministic Identity Derivation Primitive (DIDP)

ZK-ACE treats a **DIDP (Deterministic Identity Derivation Primitive)** as a black-box building block. The canonical instantiation is **ACE-GF**—a seed-storage-free cryptographic identity framework.

ACE-GF's central idea is a clean separation between identity and authorization:

- The **Root Entropy Value (REV)** is a 256-bit random value that serves as the sole source of entropy for all derived keys. It is **never stored persistently**—it exists only ephemerally in memory during execution;
- The REV is encrypted using AES-GCM-SIV (a nonce-misuse-resistant authenticated encryption scheme), producing a **Sealed Artifact**—the only object that needs to be stored;
- An authorization credential (such as a user passphrase) is processed through Argon2id to derive the unsealing key, which temporarily reconstructs the REV when needed;
- All cryptographic keys are derived via HKDF combined with an explicit context tuple `Ctx = (AlgID, Domain, Index)`, ensuring that keys under different contexts are computationally independent.

This design delivers:
- **Stateless credential rotation**: changing credentials does not change the identity root or any derived keys;
- **Immediate revocation**: removing a credential component permanently blocks access, with no server-side state required;
- **Non-disruptive PQC migration**: adding a post-quantum algorithm requires only assigning a new `AlgID`—existing classical keys and identity commitments are unaffected.

### The On-Chain Anchor: The Identity Commitment

Each identity participating in ZK-ACE maintains a compact **identity commitment** on-chain:

```
IDcom = H(REV ∥ salt ∥ domain)
```

This 32-byte hash value is the sole persistent on-chain identity anchor. The REV itself, all derived keys, and any post-quantum signature objects are absent from the chain entirely.

### The ZK Circuit: Five Core Constraints

The ZK-ACE circuit requires only five constraints to be satisfied:

**C1: Commitment Consistency**
```
H(REV ∥ salt ∥ domain) = IDcom
```
The prover controls an identity root consistent with the on-chain anchor.

**C2: Deterministic Derivation Correctness**
```
target = H(Derive(REV, Ctx))
```
The target-binding hash is consistent with derivation from the identity root under the specified context. The derived key itself is never exposed—only a hash commitment to it appears on-chain.

**C3: Authorization Binding to TxHash**
```
Auth = H(REV ∥ Ctx ∥ TxHash ∥ domain ∥ nonce)
```
The authorization is bound to a specific transaction, preventing substitution attacks.

**C4: Anti-Replay**
Two canonical modes are supported:
- **Nonce registry mode**: `rp_com = H(IDcom ∥ nonce)`, with the chain enforcing monotonicity;
- **Nullifier set mode**: `rp_com = H(Auth ∥ domain)`, with the chain maintaining a spent-nullifier set.

**C5: Domain Separation and Context Consistency**
The `domain` value is enforced consistently across all commitment and binding computations. A proof generated for one chain cannot satisfy the verifier's checks on a different chain.

### Proof System Architecture: Pluggable Backends

ZK-ACE implements a pluggable backend architecture, allowing the same circuit semantics to be proven under fundamentally different proof systems. The reference implementation provides two backends selectable via a compile-time feature flag:

**Circle STARK backend (default, post-quantum secure).** Built on [Stwo](https://github.com/starkware-libs/stwo), this backend operates over the Mersenne-31 field using Poseidon2 hash (width=16, x⁵ S-box) and the FRI protocol. Verification relies only on hash functions—no elliptic curve assumptions—making it fully post-quantum secure. The setup is transparent: no trusted ceremony is required.

**Groth16/BN254 backend (compact classical proofs).** Built on [arkworks](https://github.com/arkworks-rs), this backend uses Poseidon hash over the BN254 scalar field with R1CS constraints and pairing-based verification. It produces the smallest possible proofs (128 bytes) but relies on elliptic curve assumptions that are not quantum-resistant.

The two backends serve complementary roles:

| Aspect | Circle STARK (Stwo) | Groth16/BN254 |
|---|---|---|
| Field | Mersenne-31 (M31) | BN254 Fr (~254-bit) |
| Hash | Poseidon2 (width=16, x⁵) | Poseidon (width=3, x⁵) |
| Proof system | FRI + AIR | R1CS + pairing |
| Post-quantum secure | **Yes** | No |
| Trusted setup | **None** | Required (MPC ceremony) |
| Proof size | ~105–112 KB | **128 bytes** |
| On-chain model | Aggregated (per-block) | Per-transaction |

This architecture allows deployments to choose: use the STARK backend today for post-quantum security with mandatory aggregation, or the Groth16 backend for EVM-native compatibility with minimal on-chain footprint. A single feature flag switches between them—application code and circuit semantics remain identical.

### Constraint Scale: An Order-of-Magnitude Gap

The ZK-ACE circuit is intentionally minimal. Measured constraint counts for each backend:

| Backend | Constraint system | Total constraints |
|---|---|---|
| Circle STARK (Stwo) | AIR (algebraic intermediate representation) | **~240** |
| Groth16/BN254 | R1CS (rank-1 constraint system) | **~1,200** |

These counts are not directly comparable—an AIR constraint over a 16-column trace encodes more logic per row than a single R1CS multiplication gate—but both are extraordinarily compact by ZK circuit standards.

For reference, the structural lower bound for in-circuit ML-DSA L2 verification is on the order of **millions of R1CS constraints**. ZK-ACE's circuit is roughly **three orders of magnitude smaller**. This is not the result of parameter tuning. It is a consequence of architectural choice: proving authorization semantics directly, rather than emulating signature verification inside a circuit.

### Performance Benchmarks

Measured on Apple Silicon (Criterion.rs medians, single-threaded):

**Circle STARK backend (Stwo):**

| Operation | NonceRegistry | NullifierSet |
|---|---|---|
| Prove | **21 ms** | **21 ms** |
| Verify | **1.14 ms** | **1.19 ms** |
| Proof size | 105.2 KB | 111.5 KB |

**Groth16/BN254 backend:**

| Operation | NonceRegistry | NullifierSet |
|---|---|---|
| Prove | 44 ms | 44 ms |
| Verify | 1.51 ms | 1.52 ms |
| Proof size | **128 bytes** | **128 bytes** |

Several observations warrant attention:

**The STARK backend is faster than Groth16 in both proving and verification.** This may seem counterintuitive—STARKs are often associated with larger proofs and heavier verification—but the result is a direct consequence of the underlying field arithmetic. Mersenne-31 field operations (31-bit) are an order of magnitude cheaper than BN254 scalar field operations (254-bit), and STARK proving requires no elliptic curve multi-scalar multiplications. For small circuits like ZK-ACE, this arithmetic advantage dominates.

**Verification latencies of 1–1.5 ms are well within budget for both L1 and rollup settings.** For comparison, an secp256k1 ECDSA verification on the same hardware takes approximately 50–100 µs; the ZK-ACE overhead is roughly 10–15× that of a classical signature check, but replaces what would otherwise be a multi-kilobyte post-quantum signature.

**The STARK proof size (~105 KB) is not an on-chain cost.** Under the mandatory aggregation architecture (Section IV below), individual STARK proofs are verified off-chain by the block builder. The builder produces a single aggregated proof per block. Per-transaction on-chain data consists only of the five public input fields—approximately **160 bytes**—regardless of the number of transactions in the block.

### On-Chain Data Comparison

With the STARK backend and mandatory per-block aggregation:

| Component | PQC Signature Model | ZK-ACE (STARK, aggregated) | ZK-ACE (Groth16) |
|---|---|---|---|
| Signature / Proof | 2,420–4,627 bytes | ≈0 bytes (amortized) | 128 bytes |
| Public key (amortized) | 1,312–2,592 bytes | 0 bytes | 0 bytes |
| Identity commitment | — | 32 bytes (reused) | 32 bytes (reused) |
| Public inputs | — | ≈160 bytes | ≈160 bytes |
| **Per-transaction total** | **3,732–7,219 bytes** | **≈160 bytes** | **≈288 bytes** |

The STARK aggregation model achieves a **23–45× reduction** in per-transaction on-chain authorization data compared to direct post-quantum signatures. Even the Groth16 model achieves a **10–20× reduction**. Both figures are derived entirely from protocol structure—not from implementation-level optimization.

---

## V. Security: Four Formal Guarantees

ZK-ACE provides game-based, reduction-based security proofs under standard cryptographic assumptions:

**Authorization Soundness**: Any adversary without knowledge of REV cannot forge an accepting authorization proof. The reduction proceeds through knowledge soundness of the proof system, collision resistance of H, and DIDP identity-root recovery hardness.

**Replay Resistance**: In nonce mode, a replayed proof fails the monotonicity check. In nullifier mode, the same authorization parameters produce the same nullifier, which is already present in the spent set. In both cases, the only path to causing a second acceptance reduces to the authorization soundness game.

**Substitution Resistance**: Proofs are cryptographically bound to their public inputs. An adversary cannot rebind a valid proof to a different transaction hash or target. This reduces directly to the public-input binding property of the proof system.

**Cross-Domain Separation**: The `domain` parameter is embedded in every commitment and binding computation. A proof generated for one chain or application cannot pass verification in another. This reduces to collision resistance of H and public-input binding.

The Circle STARK backend additionally provides **post-quantum verification security**: the FRI-based verifier relies only on hash function collision resistance (Blake2s), with no dependence on elliptic curve discrete logarithm or pairing assumptions. Under the STARK backend, the entire authorization path—from identity commitment through proof generation to on-chain verification—is post-quantum secure.

---

## VI. From On-Chain Authorization to Mempool Propagation: AR-ACE

ZK-ACE addresses the on-chain authorization problem. In the mempool propagation layer, a symmetric problem is taking shape.

### The Bandwidth Crisis in Post-Quantum Mempools

In distributed block construction, large numbers of objects—blob roots, consensus-layer signature aggregates, execution-layer transactions—must be broadcast so that builders can discover and include them. In a post-quantum setting, proving the validity of these objects calls for STARKs, and a single STARK proof is on the order of **128 KB** even in size-optimized implementations.

If every object were propagated together with its own full STARK proof, relay bandwidth would scale linearly with the number of objects—quickly reaching untenable levels.

Buterin proposed recursive STARKs as a solution: each node periodically produces one recursive proof attesting to the validity of all objects it holds, and broadcasts that single proof to its peers. This caps per-node proof bandwidth at roughly **128 KB × degree / tick interval** (approximately 2 MB/s for d=8, T=0.5s). It is an elegant approach—but it still requires every node on the path to continuously generate and forward STARK proofs.

### AR-ACE: Moving Proofs Off the Propagation Path

**AR-ACE** (ACE-GF-based Attestation Relay for PQC) poses a more fundamental question:

> Must validity proofs appear on the propagation path at all?

A relay node's decision to forward an object requires only a lightweight assurance that the object is eligible for relay—not a full validity proof. Validity need only be proven once, at the point of inclusion, by the builder.

AR-ACE's design separates these concerns cleanly:

**Submitters** compute `objHash = H(obj)` and produce a compact **attestation**:
```
Attest = Sign(sk, objHash ∥ domain ∥ nonce)
```

**Relay nodes** verify only that Attest is well-formed and correctly bound to `objHash` and `domain`. They do **not** verify the object's full validity. They do not generate, hold, or forward any STARK or other validity proof.

**The builder** collects objects and attestations from the network, selects a set to include, and produces **one aggregated validity proof** (e.g., a recursive STARK) over the selected set. This single proof is published with the block.

The proof-off-path property: **validity proofs are entirely absent from the propagation path, existing only once at the builder.**

### Bandwidth Comparison

| Approach | Per-node proof-related bandwidth | Proof traffic on path |
|---|---|---|
| Per-object STARK | O(n × 128 KB) | O(n) |
| Recursive STARK aggregation | ≈128 KB × d/T (≈2 MB/s) | O(d/T), independent of n |
| **AR-ACE (proof-off-path)** | **0** | **0** |

Compared to recursive STARK propagation, AR-ACE **eliminates proof traffic from the relay path entirely**. Each relay link carries only the object and a compact attestation of tens to hundreds of bytes.

### The Advantage of ACE-GF Instantiation

When attestation keys are derived via ACE-GF using a dedicated context `Ctx_relay = (AlgID, MEMPOOL-ATTEST, domain)`:

- The same REV that backs on-chain ZK-ACE authorization derives the relay attestation key—**one identity root, one backup, no separate key management**;
- The relay attestation key is cryptographically isolated from on-chain authorization keys—a compromised relay-only key exposes nothing about on-chain identity;
- Assigning an ML-DSA `AlgID` makes the entire propagation path PQC-consistent without changing identity or backup procedures;
- Deterministic derivation ensures that the same identity and context always produce the same attestation key, enabling straightforward auditing and attribution.

---

## VII. The Complete Stack: Three Layers, One Root

Viewed together, the three papers describe a coherent post-quantum blockchain identity, authorization, and propagation stack:

```
┌────────────────────────────────────────────────────┐
│                  ACE-GF (Identity Layer)           │
│   REV → Sealed Artifact → Context-Isolated Keys   │
│   No seed storage · Stateless rotation · PQC-ready│
└──────────────────────┬─────────────────────────────┘
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
┌──────────────────────┐   ┌───────────────────────────┐
│       ZK-ACE         │   │         AR-ACE            │
│   (On-Chain Auth)    │   │   (Mempool Propagation)   │
│                      │   │                           │
│ Circle STARK (PQ)    │   │ Lightweight Attest relay  │
│  + Groth16 (compat)  │   │ Proof-off-path design     │
│ Prove: 21 ms         │   │ Single aggregated proof   │
│ Verify: 1.1 ms       │   │ at builder                │
│ ~160 B/tx (aggreg.)  │   │ Zero proof bandwidth      │
└──────────────────────┘   └───────────────────────────┘
```

Each layer applies the same architectural principle: **move heavyweight cryptographic artifacts off the paths that require large-scale replication, replacing them with succinct proofs or lightweight attestations, with full verification performed once at the endpoint.**

---

## VIII. Why This Matters

### For the Post-Quantum Migration Path

The challenge of post-quantum migration is not only developing new primitives—it is controlling their cost within the constraints that blockchain systems can tolerate. ZK-ACE demonstrates that this goal does not require incremental compression of signature sizes. It requires questioning the design assumption that signature objects need to be in the consensus path at all.

With the Circle STARK backend, ZK-ACE achieves what was previously considered a tradeoff: **post-quantum security without sacrificing verification performance**. The entire authorization path—identity commitment, proof generation, on-chain verification—relies only on hash functions, with no elliptic curve assumptions. This is not a theoretical property; it is measured: 21 ms prove, 1.1 ms verify, transparent setup, no trusted ceremony.

### For Account Abstraction

ZK-ACE is a natural fit for ERC-4337-style account abstraction. Under AA, account validation logic is defined by smart contract code. ZK-ACE can serve directly as a validator module, replacing signature-object checks with zero-knowledge proof verification—bringing post-quantum authorization semantics to the chain without modifying the base protocol.

### For Long-Lived Digital Entities

ACE-GF separates key material from identity: keys can be rotated, algorithms can be migrated, and devices can be replaced—while the identity commitment remains stable on-chain. This provides a trust-minimized identity infrastructure for long-lived digital entities, whether AI agents, decentralized services, or large-scale IoT deployments.

### For Rollups

Through batch aggregation and recursive proof composition, ZK-ACE allows a single recursive proof to cover the authorization validity of an entire batch of transactions, amortizing per-proof cost toward zero. Combined with AR-ACE's propagation-layer design, this constitutes an end-to-end scalable authorization architecture for post-quantum blockchain systems.

---

## Conclusion

The evolution of cryptographic standards consistently prompts a re-examination of system architecture. The arrival of post-quantum signatures has exposed a long-unexamined design assumption in blockchain authorization: treating the signature object as the primary unit of verification, rather than treating authorization itself as a semantic property that can be proven independently.

ZK-ACE and AR-ACE do not propose new cryptographic primitives. Their contribution is to redefine the problem: authorization need not depend on signature objects; propagation need not carry validity proofs. Within this framework, the tension between post-quantum security and blockchain scalability transforms from a structural conflict into a tractable engineering problem.

The reference implementation is open-source: [github.com/acechain-io/zk-ace](https://github.com/acechain-io/zk-ace)

---

*This article is based on the following technical publications:*
- *ACE-GF: A Generative Framework for Atomic Cryptographic Entities (arXiv:2511.20505)*
- *ZK-ACE: Identity-Centric Zero-Knowledge Authorization for Post-Quantum Blockchain Systems*
- *AR-ACE: ACE-GF-based Attestation Relay for PQC (arXiv:2603.07982)*
