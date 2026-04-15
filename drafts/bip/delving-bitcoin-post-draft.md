# Beyond Signatures: Full-Stack Post-Quantum Cryptography for Bitcoin

## Introduction

Hi — This is Jason (@0xJasonw). I'm new to this forum and would like to discuss Bitcoin in a post-quantum setting and get your feedback.

**Disclosure:** I authored the [ACE-GF](https://arxiv.org/abs/2511.20505) (Internet-Draft: https://datatracker.ietf.org/doc/draft-wang-acegf-protocol/) and the [ZK-ACE](https://arxiv.org/abs/2603.07974) paper cited below. I reference them as concrete examples of the identity and ZK angles — not as product pitches. Corrections, criticism, and alternative constructions are welcome.

Most Bitcoin PQC discussions — including BIP-360 (P2MR) — focus on protecting transaction authorization against quantum adversaries. This is necessary but insufficient. Bitcoin's security stack has three quantum-vulnerable layers, and I'd like to open a discussion on whether the PQC roadmap should be scoped more broadly.

This post presents three independently deployable directions and poses specific questions for community feedback. I'm not proposing a formal BIP at this stage — this is an exploration of the design space.

## The Three Vulnerable Layers

**1. P2P Transport.** BIP-324 uses secp256k1 ECDH for session key establishment. A quantum adversary performing passive collection today can decrypt all recorded P2P traffic once a CRQC becomes available. This exposes transaction content, IP-to-transaction linkage, and propagation timing — even if on-chain authorization is quantum-safe.

**2. On-chain Authorization.** Classical ECDSA and Schnorr signatures are vulnerable to Shor's algorithm. Post-quantum signature schemes (ML-DSA, SLH-DSA) produce signatures on the order of 2.4–8 KB per transaction — a 30–125x increase in per-transaction authorization data.

**3. Identity Model.** Bitcoin's key management (BIP-32/39) relies on persistent master seeds. Migrating to PQC under the current model requires rekeying every derived key across every derivation path, with no clean upgrade path.

## Direction A: Hybrid KEM Transport for BIP-324

Replace the secp256k1 ECDH key exchange in BIP-324 with a hybrid KEM + ECDH construction. The hybrid approach ensures security is never worse than current BIP-324, even if the PQC KEM is later found to have weaknesses.

**Handshake overview (using ML-KEM-768 as example):**

- Initiator sends: ElligatorSwift-encoded ECDH pubkey (64 B) + ML-KEM encapsulation key (1,184 B)
- Responder sends: ElligatorSwift-encoded ECDH pubkey (64 B) + ML-KEM ciphertext (1,088 B)
- Both sides combine: `ss_combined = HKDF-SHA256(salt, ss_ecdh || ss_kem, info)`
- Session keys derived from ss_combined via existing BIP-324 key schedule

**Wire cost:** +2,272 bytes per handshake (one-time). At 8–10 outbound connections, ~18–23 KB additional data per node — negligible relative to block relay.

**Negotiation:** Service bit (NODE_PQC_TRANSPORT). Falls back to standard BIP-324 if either side doesn't support it. Post-handshake packet sizes unchanged.

The security model is straightforward: the hybrid is secure if *either* ECDH *or* ML-KEM-768 remains unbroken. This provides harvest-now-decrypt-later resistance with zero risk of regression.

## Direction B: Algorithm-Agnostic Identity Commitment

Instead of anchoring outputs directly to PQC public keys (which are large and algorithm-specific), anchor them to a compact, algorithm-agnostic identity commitment.

**How the identity model works:** The identity root is a 256-bit entropy value, but unlike a BIP-32/39 master seed, it is not stored persistently. Instead, it is sealed into a compact artifact (74 bytes) encrypted under a user credential via Argon2id + AES-256-GCM-SIV, and reconstructed ephemerally only when needed. The sealed artifact alone is not sufficient to recover the identity without the credential — there is no master seed to back up or protect at rest.

From this single identity root, keys for any algorithm are derived via HKDF with context tuples `(AlgID, Domain, Index)`. Adding a new PQC algorithm is just a new AlgID — the identity root, sealed artifact, and all existing keys remain unchanged. No on-chain migration, no rekeying.

The on-chain commitment is simply:

```
ID_com = SHA-256(identity_root || salt || domain)
```

**Comparison with existing approaches:**

| Feature | P2TR (v1) | P2MR / BIP-360 (v2) | Identity Commitment |
|---------|-----------|---------------------|---------------------|
| On-chain anchor | 32-byte tweaked pubkey | 32-byte Merkle root | 32-byte hash commitment |
| Key-path spend | Yes (quantum-vulnerable) | No | No |
| Algorithm agility | Fixed (Schnorr) | Via OP_SUCCESS upgrade | Native (context-isolated derivation) |
| Identity persistence across PQC migration | No | No | Yes |

This would require a new SegWit witness version (deployed via soft fork). The on-chain commitment is always 32 bytes regardless of the underlying PQC key size.

The identity derivation model is formalized in [ACE-GF](https://datatracker.ietf.org/doc/draft-wang-acegf-protocol/) (IETF individual draft; see disclosure above), which describes deterministic identity derivation with context-isolated HKDF key generation. The sealed-artifact construction enables seed-storage-free identity reconstruction — the identity root exists only ephemerally in volatile memory during active operations.

## Direction C: ZK Authorization with Mandatory Proof Aggregation

Rather than compressing PQC signatures via ZK circuits (which requires expressing lattice arithmetic — millions of constraints), prove authorization *directly* from the identity root using only ZK-friendly hash evaluations:

> The prover demonstrates: "I know an identity root such that (i) it matches the on-chain commitment, and (ii) it authorizes this specific transaction."

**Why this matters for PQC:**

The proof system must itself be post-quantum secure. Pairing-based SNARKs (Groth16, PLONK with KZG) rely on discrete-log assumptions broken by Shor's algorithm — using them for "post-quantum authorization" merely shifts the quantum vulnerability from signatures to verification. Circle STARKs (hash-based, no trusted setup) avoid this entirely.

**The aggregation architecture resolves the proof size problem:**

Individual STARK proofs are ~50–100 KB (larger than signatures), but:

1. Wallets generate per-transaction proofs off-chain
2. Block builders recursively aggregate all proofs into a single per-block proof (~100–200 KB regardless of tx count)
3. Consensus verifies one proof per block (constant time) + per-tx binding checks (lightweight hash/lookup)

Net per-transaction on-chain data: ~160 bytes (public inputs only), compared to ~2,452 bytes for direct ML-DSA-65 signatures.

Reference implementation using Stwo Circle STARKs achieves ~15 ms proving, ~1.1 ms verification (single-threaded, Apple M-series) with ~240 AIR constraints. Details in [ZK-ACE](https://arxiv.org/abs/2603.07974); see disclosure above.

**Proof system agility:** Because the on-chain identity commitment depends only on the hash function — not the proof system — upgrading to a better prover in the future does not require changing on-chain commitments or consensus rules for the identity layer. Only the verifier needs to be updated to accept the new proof format. This avoids locking Bitcoin into a specific proof system at the consensus level.

**Honest caveat:** Mandatory proof aggregation significantly increases the computational burden on block builders, which could create centralization pressure. This tradeoff deserves careful analysis.

## Questions for Discussion

I'd appreciate focused responses to any of these:

1. **Should transport-layer PQC be part of Bitcoin's PQC roadmap?** BIP-324 sessions recorded today are decryptable by a future CRQC. Is a hybrid KEM upgrade worth pursuing independently of consensus-layer PQC?

2. **Is an algorithm-agnostic identity commitment a useful abstraction?** BIP-360 anchors to Merkle roots of algorithm-specific script trees. An identity commitment anchors to the identity itself. Is this additional indirection justified by the migration benefits, or unnecessary complexity?

3. **Is ZK authorization in scope for Bitcoin, or should PQC remain signature-centric?** Direction C is a larger departure from Bitcoin's current model. Should this be explored within Bitcoin, or is it better suited for new systems?

