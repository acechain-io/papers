```
  BIP: ?
  Title: Full-Stack Post-Quantum Cryptography
  Authors: Jason Wang <jason@acechain.io>
  Status: Draft
  Type: Informational
  Assigned: ?
  License: BSD-2-Clause
```

## Abstract

This draft argues that Bitcoin's post-quantum cryptography (PQC) roadmap should be scoped beyond signature-layer protection to address transport, identity, and authorization as a whole. It presents three related but independently deployable directions for community discussion:

1. **Component A — PQC Transport (Peer Services layer):** Replaces the secp256k1 ECDH key exchange in BIP-324 v2 encrypted transport with a hybrid KEM + ECDH construction whose transport key material is derived from the same context-isolated identity substrate surfaced on-chain by Component B, providing quantum-resistant confidentiality together with peer authentication. This draft uses ML-KEM-768 as the example KEM instantiation.

2. **Component B — PQC Identity Commitment (Consensus layer):** Introduces a new SegWit output type anchored to a compact, algorithm-agnostic identity commitment derived from the ACE-GF deterministic identity framework, enabling seed-storage-free PQC identity binding on-chain.

3. **Component C — ZK-ACE Consensus Verification (Consensus layer):** Replaces per-transaction signature objects with identity-bound zero-knowledge authorization proofs verified via post-quantum-secure Circle STARKs. Per-transaction proofs are mandatorily aggregated by block builders into a single per-block proof, eliminating post-quantum signature bloat from consensus-visible data and reducing the heavy cryptographic verification in each block from O(N) per-transaction signature checks to a single aggregated proof verification.

Each component is independently deployable and provides standalone value. Components B and C compose to deliver a complete post-quantum authorization model. All three together constitute full-stack PQC protection covering transport, identity, and consensus.

## Copyright

This BIP is licensed under the BSD 2-Clause License.

## Motivation

### The Quantum Threat Is Not Limited to Signatures

Current Bitcoin PQC discussions — including BIP-360 (P2MR) — focus almost exclusively on protecting transaction authorization against quantum adversaries. This is necessary but insufficient. Bitcoin's security stack has three quantum-vulnerable layers:

1. **P2P Transport.** BIP-324 uses secp256k1 ECDH for session key establishment. A quantum adversary performing passive collection today can decrypt all recorded P2P traffic once a cryptographically relevant quantum computer (CRQC) becomes available. This exposes transaction content, IP-to-transaction linkage, and propagation timing — even if on-chain authorization is quantum-safe.

2. **On-chain Authorization.** Classical ECDSA and Schnorr signatures on secp256k1 are vulnerable to Shor's algorithm. Post-quantum signature schemes (ML-DSA, SLH-DSA) produce signatures on the order of 2.4–8 KB per transaction, compared to 64–72 bytes for classical signatures — a 30–125× increase in per-transaction authorization data.

3. **Identity Model.** Bitcoin's key management (BIP-32/39) relies on persistent master seeds stored as mnemonic phrases. Migrating to PQC under the current model requires rekeying every derived key across every derivation path, creating a systemic migration burden with no clean upgrade path.

### Limitations of Signature-Only Approaches

BIP-360 proposes P2MR (Pay-to-Merkle-Root) using SegWit v2, removing the key-path spend to eliminate long-exposure quantum vulnerability. While this is a sound incremental step, it:

- **Does not address transport-layer quantum vulnerability.** Recorded BIP-324 sessions remain decryptable.
- **Does not reduce PQC signature overhead.** Future PQC signature opcodes within P2MR script paths will still carry kilobyte-scale signatures per transaction, increasing block weight proportionally.
- **Does not address mempool propagation costs.** Every relay node must forward and verify kilobyte-scale PQC signatures, scaling linearly with transaction volume.
- **Does not provide a PQC identity migration path.** Users must still manage classical seed-based key hierarchies and re-derive keys when migrating algorithms.

### A Structural Alternative: Identity–Authorization Separation

Rather than incrementally patching each layer, this BIP proposes a structural approach based on *identity–authorization separation* — decoupling the stable cryptographic identity from per-transaction authorization mechanisms. This separation, formalized in ACE-GF [1], enables:

- **Non-disruptive PQC migration:** A single identity root deterministically derives keys across classical and post-quantum algorithms via context-isolated HKDF derivation. Adding or replacing an algorithm requires only a fresh AlgID or domain label, without rekeying or re-enrolling the identity. Related wallet-layer migration work such as SA-Migration [4] explores preserving existing addresses by encapsulating legacy BIP-39 mnemonics or raw private keys into sealed artifacts and replaying the original wallet-specific derivation procedure.
- **Cross-domain key separation:** Transport encryption, peer authentication, signing, and on-chain authorization keys can be derived from the same identity root under distinct contexts. This keeps the domains separate while preserving a consistent underlying identity.
- **Elimination of signature objects from consensus:** ZK-ACE [2] proves transaction authorization via zero-knowledge proofs against an on-chain identity commitment, achieving constant-size on-chain authorization data regardless of the underlying cryptographic scheme.
- **Proof-off-path mempool propagation:** AR-ACE [3] replaces per-object validity proofs with compact attestations on the relay path, deferring proof generation to the builder.

### Questions for Discussion

This draft is structured around three specific questions that the author believes the community should address as part of Bitcoin's long-term PQC roadmap:

1. **Should transport-layer PQC be considered part of Bitcoin's PQC roadmap?** BIP-324 sessions recorded today are decryptable by a future quantum adversary. If transport key material is derived from the same identity substrate used for PQC authorization, transport authentication can also be handled within the same derivation model. Is this worth pursuing independently of consensus-layer PQC?

2. **Is an algorithm-agnostic on-chain identity commitment a useful abstraction, or unnecessary complexity?** BIP-360 anchors outputs to Merkle roots of script trees containing algorithm-specific keys. An identity commitment anchors to the identity itself, independent of any algorithm. Is this additional indirection justified by the migration benefits, or does it introduce complexity that Bitcoin should avoid?

3. **Is signature elimination via zero-knowledge authorization in scope for Bitcoin, or should PQC work remain signature-centric?** Component C proposes replacing signatures with ZK proofs of authorization. With mandatory proof aggregation, this offers significant data and verification savings — per-transaction on-chain data reduces to ~160 bytes of public inputs, and per-block verification reduces to a single aggregated STARK proof check. This represents a larger departure from Bitcoin's current model. Should this direction be explored within Bitcoin, or is it better suited for new systems?

The author welcomes focused responses to any of these questions. If community interest concentrates on a subset of the components, each will be split into a standalone BIP.

## Specification

### Component A: PQC Transport — Identity-Derived Hybrid KEM for BIP-324

#### Overview

Component A upgrades the BIP-324 v2 encrypted P2P transport protocol by replacing the quantum-vulnerable secp256k1 ElligatorSwift ECDH key exchange with a hybrid key encapsulation mechanism combining an initially selected post-quantum KEM instantiation (ML-KEM-768 in this draft) and secp256k1 ECDH.

The symmetric encryption layer (ChaCha20-Poly1305) and packet framing remain unchanged. Only the initial key exchange is modified. The key material used by this exchange is not treated as an unrelated ad hoc transport secret: it is derived from the same context-isolated identity substrate described in Component B. In other words, Component B is the consensus-facing manifestation of the identity layer, while Component A consumes that same layer for transport encryption and transport authentication.

#### Identity-Derived Transport Key Material

Component A assumes that each node derives transport key material from a stable identity root using explicit context tuples of the form

```
Ctx = (AlgID, Domain, Index)
```

For example:

- `Ctx = (Secp256k1, transport_ecdh, epoch)` for the ECDH transport stream
- `Ctx = (ML-KEM-768, transport_kem, epoch)` for the post-quantum KEM transport stream
- `Ctx = (ML-DSA-65 or Secp256k1, transport_auth, epoch)` for optional transport-origin authentication

The crucial point is that transport encryption and transport authentication are separate streams under the same identity root. This preserves key separation while allowing the transport layer to participate in the same identity model as authorization and signing. The same derivation pattern also permits introducing a different transport KEM later by allocating a fresh AlgID and transport domain.

#### Hybrid Key Exchange Protocol

The upgraded handshake proceeds as follows:

**Initiator (connecting node):**
1. Derive or instantiate a transport secp256k1 keypair from the identity root under a `transport_ecdh` context, and encode pk_ecdh using ElligatorSwift (64 bytes), as in BIP-324.
2. Derive or instantiate an ML-KEM-768 keypair from the same identity root under a distinct `transport_kem` context. The encapsulation key ek_kem is 1,184 bytes.
3. Send: `pk_ecdh_encoded || ek_kem` (64 + 1,184 = 1,248 bytes).

**Responder (listening node):**
1. Receive and decode the initiator's ElligatorSwift-encoded public key and ML-KEM encapsulation key.
2. Derive or instantiate its own transport secp256k1 keypair under the corresponding `transport_ecdh` context, and perform ECDH to obtain `ss_ecdh` (32 bytes).
3. Encapsulate against ek_kem to obtain `(ct_kem, ss_kem)`, where ct_kem is 1,088 bytes and ss_kem is 32 bytes.
4. Send: `pk_ecdh_encoded || ct_kem` (64 + 1,088 = 1,152 bytes).

**Both sides compute the combined shared secret:**

```
ss_combined = HKDF-SHA256(
    salt = "bitcoin_v2_pqc_hybrid",
    ikm  = ss_ecdh || ss_kem,
    info = "pqc_transport_v1",
    L    = 32
)
```

Session keys are then derived from ss_combined using the existing BIP-324 HKDF key schedule (session ID, per-direction encryption keys, garbage terminators).

#### Optional Origin Authentication

Because transport encryption and transport authentication are distinct context-derived streams, a node may additionally attach an authentication-domain attestation over the handshake transcript or a message hash using a `transport_auth` key derived from the same identity root. In a transport-only deployment, this binds the channel to a stable peer identity at the peer-services layer. When composed with Component B, the same transport-authentication public key family can be interpreted as belonging to the identity later committed on-chain, so the encrypted message is not only post-quantum protected in transit but also attributable to a specific identity origin.

#### Security Model

The hybrid construction is secure if *either* secp256k1 ECDH *or* ML-KEM-768 remains unbroken. This provides:
- **Classical security:** Equivalent to current BIP-324 via the ECDH component.
- **Post-quantum security:** ML-KEM-768 provides IND-CCA2 security under the Module Learning With Errors (MLWE) assumption at NIST security level 3.
- **Harvest-now-decrypt-later resistance:** Passively recorded sessions cannot be decrypted by a future quantum adversary.
- **Shared identity model across domains:** When transport key material is derived from the same identity root and paired with a distinct transport-authentication stream, transport authentication can be handled within the same derivation model without reusing consensus or signing keys.

#### Negotiation

Nodes signal support for the hybrid handshake via a service bit (NODE_PQC_TRANSPORT). During connection establishment:
- If both nodes support hybrid transport, the extended handshake is used.
- If either node does not support hybrid transport, the connection falls back to standard BIP-324 (or v1 if BIP-324 is also unsupported).

This ensures full backward compatibility.

#### Wire Cost

| Direction | BIP-324 Current | Component A |
|-----------|----------------|-------------|
| Initiator → Responder | 64 bytes | 1,248 bytes |
| Responder → Initiator | 64 bytes | 1,152 bytes |
| **Handshake total** | **128 bytes** | **2,400 bytes** |

The additional 2,272 bytes occur once per connection establishment. At Bitcoin's typical peer count (8–10 outbound connections), this represents approximately 18–23 KB of additional one-time handshake data per node — negligible relative to ongoing block relay traffic.

Post-handshake packet sizes are unchanged.

---

### Component B: PQC Identity Commitment

#### Overview

Component B introduces a new SegWit output type that commits to a compact, algorithm-agnostic identity anchor derived from the ACE-GF deterministic identity derivation primitive [1]. This enables on-chain identity binding that is independent of any specific signature algorithm and supports non-disruptive PQC migration.

#### Identity Commitment Construction

An identity commitment is a 32-byte hash-based anchor:

```
ID_com = H(REV || salt || domain)
```

where:
- **REV** is a 256-bit Root Entropy Value — the ephemeral identity root of the ACE-GF construction. It is never stored persistently and is deterministically reconstructed from a sealed artifact and authorization credentials.
- **salt** is an identity-specific random value enabling commitment re-randomization and cross-domain isolation.
- **domain** encodes the target chain or application namespace (e.g., `"bitcoin-mainnet"`).
- **H** is a collision-resistant hash function (SHA-256 for on-chain commitment; ZK-friendly hash for in-circuit use, see Component C).

#### SegWit Output Type

Component B defines a new witness version (version 3) output:

```
scriptPubKey: OP_3 OP_PUSHBYTES_32 <ID_com>
```

Witness version 3 outputs are encoded as bech32m addresses per BIP-350. The human-readable prefix and witness version byte follow the standard SegWit address encoding rules.

**Spending rules:**

An output paying to `OP_3 <ID_com>` is spendable by providing a witness satisfying one of two modes:

**Mode 1 — Direct PQC Signature (standalone, without Component C):**

```
witness: <pqc_signature> <pqc_pubkey> <context_proof>
```

Where context_proof demonstrates that pqc_pubkey is deterministically derived from the identity root consistent with ID_com, under a declared derivation context. The specific PQC signature algorithm is identified by a version byte prefix on pqc_pubkey.

Supported algorithms (initial set):
- `0x01`: ML-DSA-65 (FIPS 204, ~2,420 byte signatures)
- `0x02`: SLH-DSA-SHA2-128s (FIPS 205, ~7,856 byte signatures)
- `0x03`: FALCON-512 (pending NIST standardization, ~666 byte signatures)

**Mode 2 — ZK-ACE Authorization (with Component C):**

```
witness: <zk_proof> <public_inputs>
```

Where the zero-knowledge proof demonstrates authorization without any signature object (see Component C specification).

#### Deterministic Key Derivation

Keys are derived from REV using HKDF with explicit context encoding:

```
PRK = HKDF-Extract(salt_kdf, REV)
Key = HKDF-Expand(PRK, Ctx, L)
```

where `Ctx = (AlgID, Domain, Index)` encodes the target algorithm (e.g., ML-DSA-65, Secp256k1, ML-KEM-768), application domain (e.g., onchain_authorization, transport_kem, transport_auth), and derivation index.

**Context isolation guarantee:** Under the pseudorandomness assumption of HKDF, keys derived with different context tuples are computationally independent. Compromise of a key in one context does not enable recovery of keys in other contexts, nor does it reveal REV.

**PQC migration path:** Adding a post-quantum algorithm requires only a new AlgID value. Replacing a deprecated algorithm likewise requires only a fresh AlgID/domain pair. The identity root, sealed artifact, and all previously derived classical keys remain unchanged. Classical and post-quantum keys coexist under the same identity, and multiple PQC generations can coexist as parallel derivation streams.

#### Relationship to Existing Output Types

| Feature | P2TR (v1) | P2MR / BIP-360 (v2) | This BIP (v3) |
|---------|-----------|---------------------|---------------|
| On-chain anchor | 32-byte tweaked pubkey | 32-byte Merkle root | 32-byte identity commitment |
| Key-path spend | Yes (quantum-vulnerable) | No | No |
| Script-path spend | Yes | Yes | Yes (Mode 1) |
| ZK authorization | No | No | Yes (Mode 2) |
| Algorithm agility | Fixed (Schnorr) | Via OP_SUCCESS upgrade | Native (context-isolated derivation) |
| Identity persistence across algorithms | No | No | Yes (same REV, different AlgID) |

---

### Component C: ZK-ACE Consensus Verification

#### Overview

Component C replaces per-transaction signature objects with identity-bound zero-knowledge authorization proofs based on the ZK-ACE construction [2]. The consensus layer verifies a succinct proof that a transaction was authorized by the identity committed on-chain, without seeing or verifying any post-quantum signature.

#### Authorization Model

The core authorization statement proven in zero knowledge:

> *There exists a Root Entropy Value REV and associated derivation context such that: (i) the identity commitment derived from REV matches the on-chain commitment ID_com, and (ii) the identity anchored by REV has authorized the transaction with hash TxHash under the prescribed derivation context and replay-prevention rules.*

The zero-knowledge proof attests to identity ownership and transaction authorization without revealing REV, any derived keys, or any signature object.

#### ZK Circuit Specification

The ZK-ACE circuit proves knowledge of a private witness `w = (REV, salt, Ctx, nonce, aux)` satisfying the following constraints against public inputs `π_pub = (ID_com, TxHash, domain, target, rp_com)`:

**(C1) Commitment consistency:**
```
H(REV || salt || domain) = ID_com
```

**(C2) Deterministic derivation correctness:**
```
target = H(Derive(REV, Ctx))
```

where `target` is a hash commitment to the context-derived key material.

**(C3) Authorization binding to TxHash:**
```
Auth = H(REV || Ctx || TxHash || domain || nonce)
```

The circuit verifies that Auth is correctly computed, binding the authorization to a specific transaction, identity, and context.

**(C4) Anti-replay (nonce/nullifier rule):**

Two canonical replay-prevention modes are supported:

- *Nonce commitment (account-style):*
  ```
  rp_com = H(ID_com || nonce)
  ```
  The verifier enforces nonce monotonicity per identity commitment.

- *Nullifier (UTXO-style):*
  ```
  rp_com = H(Auth || domain)
  ```
  The verifier maintains a global nullifier set and rejects duplicate nullifiers.

**(C5) Domain separation and context consistency:**
```
Ctx.Domain = domain
```

Ensures all commitment and derivation computations use the declared domain, preventing cross-chain proof reuse.

#### Proof System: Circle STARKs

The ZK-ACE reference implementation uses Circle STARKs (Stwo) over the Mersenne-31 (M31) field with Poseidon2 as the ZK-friendly hash function. This choice is mandated by the full-stack PQC requirement: SNARK-based proof systems (Groth16, PLONK with KZG) rely on elliptic curve pairings or discrete-log assumptions that are vulnerable to Shor's algorithm. A full-stack PQC proposal cannot depend on a non-PQ proof system for its authorization layer.

Circle STARKs provide:
- **Post-quantum security.** Verification relies only on hash functions (Blake2s Merkle commitments) and the FRI protocol — no elliptic curve assumptions.
- **Transparency.** No trusted setup ceremony. The proof system is fully transparent, eliminating the toxic-waste risk inherent in pairing-based SNARKs.
- **Concrete performance.** Proving time ~15–20 ms, verification time ~1.1 ms (single-threaded, Apple M-series).

The circuit uses an Algebraic Intermediate Representation (AIR) with ~240 degree-bounded constraints over a 512-row trace (2⁹), requiring 13 Poseidon2 permutations (width-16, α=5, 4+22+4 rounds). This is roughly **four orders of magnitude smaller** than embedding ML-DSA signature verification in a ZK circuit (which requires millions of constraints for lattice arithmetic over polynomial rings).

| Constraint | AIR Constraints |
|-----------|----------------|
| (C1) Commitment consistency (Poseidon2 ×1) | ~50 |
| (C2) Derivation correctness (Poseidon2 ×2) | ~70 |
| (C3) Authorization binding (Poseidon2 ×1) | ~50 |
| (C4) Replay prevention (Poseidon2 ×1) | ~50 |
| (C5) Domain separation + witness constancy | ~20 |
| **Total** | **~240** |

#### On-Chain Data Comparison

ZK-ACE uses mandatory proof aggregation: individual STARK proofs (~50–100 KB) are generated by wallets and forwarded to block builders off-chain. Builders recursively aggregate all per-transaction proofs into a single per-block proof. Only the aggregated proof and per-transaction public inputs appear on-chain.

| Authorization Model | Per-Tx On-Chain Data | Per-Block Proof | Verification Cost |
|--------------------|---------------------|----------------|-------------------|
| ECDSA (current) | ~72 B (sig + pubkey) | — | O(N) sig checks |
| ML-DSA-65 (direct PQC) | ~2,452 B (sig + pubkey) | — | O(N) sig checks |
| ZK-compressed ML-DSA† | ~128–256 B (SNARK proof) | — | O(N) proof checks |
| **ZK-ACE (this BIP)** | **~160 B (public inputs)** | **~100–200 KB (1 aggregated STARK proof)** | **O(1) proof verification + O(N) lightweight binding checks** |

† ZK-compressed ML-DSA approaches use pairing-based SNARKs (Groth16/PLONK with KZG) whose verifiers are not post-quantum secure. ZK-ACE eliminates this dependency entirely.

Under ZK-ACE, per-transaction on-chain data consists solely of public inputs (identity commitment, transaction hash, domain tag, target binding, replay-prevention commitment — approximately 160 bytes). The expensive cryptographic verification is a single aggregated STARK proof per block, verified in constant time. Per-transaction context binding (TxHash recomputation, ID_com lookup, replay state update) remains O(N) but involves only lightweight hash and lookup operations.

#### Proof Aggregation (Architectural Requirement)

Proof aggregation is a mandatory architectural component of ZK-ACE, not an optional optimization. Individual Circle STARK proofs are ~50–100 KB — significantly larger than classical or post-quantum signatures. This is the inherent cost of transparency (no trusted setup) and post-quantum security (no elliptic curve assumptions). The aggregation architecture resolves this tradeoff:

1. **Wallet** generates a per-transaction STARK proof (~50–100 KB) and forwards it to a block builder via the relay path.
2. **Builder** recursively composes all per-transaction proofs into a single aggregated STARK proof (~100–200 KB regardless of transaction count).
3. **Consensus** verifies one aggregated proof per block (constant time) plus per-transaction binding checks (O(N) lightweight hash and lookup operations).

The per-transaction relay overhead (~50–100 KB) is comparable to current block relay traffic and does not enter the consensus-critical path. The AR-ACE proof-off-path design [3] further reduces relay overhead by substituting compact attestations for full proofs on the propagation path (see Mempool Propagation below).

#### Mempool Propagation (AR-ACE Integration)

When Component C is deployed, mempool propagation follows the AR-ACE proof-off-path design [3]:

- **Relay nodes** forward transactions plus compact attestations (identity-bound signatures or commitments binding the attester to the transaction hash). Relay nodes verify only attestation validity — they do not generate, hold, or forward ZK proofs.
- **Builders** perform a single aggregated validity proof over the set of included transactions.

This removes proof overhead from the propagation path entirely. Per-object relay overhead is limited to the attestation size (64–256 bytes for classical or PQC attestations), comparable to current per-transaction signature overhead.

## Rationale

### Why Three Components in One BIP

The three components address distinct layers of Bitcoin's quantum vulnerability but share a common architectural foundation: identity–authorization separation. Presenting them as a unified proposal:

1. **Demonstrates the full scope** of Bitcoin's quantum attack surface — not just signatures, but transport and identity infrastructure.
2. **Preserves modular deployability** — each component can be activated independently via separate soft forks (Components B, C) or peer-services upgrades (Component A).
3. **Avoids redundant work** — Components A, B, and C share the same identity-derivation substrate, while B and C additionally share the consensus-facing identity commitment scheme. Specifying them separately would duplicate the identity layer definition.

### Why Hybrid KEM (Component A)

A hybrid construction (ECDH + ML-KEM) rather than ML-KEM alone is chosen because:
- The classical ECDH component ensures that security is never worse than the current BIP-324 protocol, even if ML-KEM is found to have weaknesses.
- ML-KEM-768 is a NIST-standardized algorithm (FIPS 203) with conservative security margins at NIST level 3, making it a reasonable initial KEM instantiation.
- The same identity root can derive transport KEM, transport authentication, and consensus-facing authorization keys in separate streams, allowing the transport channel to be both post-quantum confidential and identity-bound.
- Because KEM selection is expressed through AlgID/domain separation rather than through a permanently fixed identity format, later post-quantum KEMs can be adopted without changing the underlying identity substrate.
- The one-time handshake overhead (~2.3 KB) is negligible for long-lived Bitcoin peer connections.

### Why Identity Commitments Instead of Direct PQC Public Keys (Component B)

BIP-360 anchors outputs to a Merkle root of script trees, which ultimately contain PQC public keys in leaf scripts. This BIP anchors outputs to an algorithm-agnostic identity commitment. The advantages:

- **Algorithm agility without on-chain migration.** When a new PQC standard emerges, the same identity commitment supports a new derivation context — no on-chain output migration is needed.
- **Compact on-chain footprint.** The commitment is always 32 bytes, regardless of the size of the underlying PQC public key (ML-DSA-65 public keys are 1,952 bytes).
- **Unified identity across algorithms.** Classical secp256k1 keys and PQC keys are derived from the same identity root under different contexts, enabling a smooth migration path where both classical and PQC spending paths coexist.

### Why Zero-Knowledge Authorization Instead of Signature Compression (Component C)

A widely discussed PQC approach is to verify post-quantum signatures inside ZK circuits and post only the succinct proof on-chain. ZK-ACE takes a fundamentally different approach — it does not verify any signature inside the circuit. Instead, it proves authorization directly from the identity root.

This distinction matters because:
- **No lattice arithmetic in the circuit.** Verifying ML-DSA inside a ZK circuit requires expressing Number Theoretic Transforms (NTTs) over degree-256 polynomial rings, high-bit-width modular reductions, and rejection sampling — yielding circuits on the order of millions of constraints. ZK-ACE uses only ZK-friendly hash evaluations (~240 AIR constraints).
- **Proof-system agnostic.** ZK-ACE commitments depend on the hash function, not the proof system. On-chain identity commitments persist across proof system upgrades.
- **Eliminates the signature entirely** rather than compressing it. The on-chain authorization artifact is a proof, not a compressed signature.

### Why Circle STARKs Instead of Pairing-Based SNARKs (Component C)

A full-stack PQC proposal that relies on a non-PQ proof system for its authorization layer is internally inconsistent. Pairing-based SNARKs (Groth16, PLONK with KZG commitments) depend on the hardness of the discrete logarithm problem in elliptic curve groups — the same assumption broken by Shor's algorithm. Using such a proof system to provide "post-quantum authorization" merely shifts the quantum vulnerability from the signature layer to the verification layer.

Circle STARKs (specifically, the Stwo prover over the Mersenne-31 field) eliminate this dependency:

- **Verification relies only on hash functions.** The FRI protocol and Merkle commitment scheme (Blake2s) have no algebraic structure exploitable by quantum algorithms. Security reduces to collision resistance of the hash function.
- **No trusted setup.** Pairing-based SNARKs require a structured reference string (SRS) generated by a trusted setup ceremony. Any compromise of the setup — including by a future quantum adversary — invalidates all proofs generated under that SRS. STARKs are fully transparent.
- **Proof size tradeoff is resolved by mandatory aggregation.** Individual STARK proofs (~50–100 KB) are larger than SNARK proofs (~128–256 bytes), but this comparison is misleading in the ZK-ACE architecture: individual proofs never appear on-chain. The per-block aggregated proof is ~100–200 KB regardless of transaction count, while per-transaction on-chain data is ~160 bytes (public inputs only). The net on-chain cost per transaction is lower than direct PQC signatures.

The choice of STARKs over SNARKs is not a performance concession — it is a logical requirement of the full-stack PQC thesis.

### Why Not Extend BIP-360

BIP-360 is a sound incremental approach for removing key-path quantum exposure in Taproot outputs. This BIP is complementary, not competing. Specifically:

- BIP-360 provides an immediately deployable migration path for existing Taproot users.
- This BIP provides a structural framework for long-term PQC readiness across the full stack.
- A Bitcoin node could support both P2MR (BIP-360, SegWit v2) and PQC Identity Commitment (this BIP, SegWit v3) simultaneously.

## Backwards Compatibility

### Component A (PQC Transport)

- Negotiated via service bit. Non-supporting nodes fall back to standard BIP-324 or v1 transport.
- No consensus impact. Purely a peer-services layer change.
- Existing connections and relay behavior are unaffected.

### Component B (PQC Identity Commitment)

- Requires a soft fork to define spending rules for SegWit version 3 outputs.
- Pre-soft-fork nodes treat SegWit v3 outputs as anyone-can-spend (standard witness program behavior per BIP-141).
- Existing P2PKH, P2SH, P2WPKH, P2WSH, P2TR, and P2MR outputs are completely unaffected.
- Users opt in by creating new SegWit v3 outputs; no existing UTXOs require migration.

### Component C (ZK-ACE Consensus Verification)

- Deployed as an extension to Component B spending rules (Mode 2 witness format).
- Requires the same soft fork as Component B, or a subsequent soft fork if deployed later.
- Does not affect validation of any existing output type.

### Migration Path

1. **Phase 1 (Peer Services):** Deploy Component A. All P2P traffic gains quantum-resistant confidentiality. No consensus change required.
2. **Phase 2 (Consensus — Identity):** Deploy Component B via soft fork. Users can create PQC-committed outputs with direct signature spending (Mode 1).
3. **Phase 3 (Consensus — ZK Authorization):** Deploy Component C, enabling ZK-ACE authorization spending (Mode 2) for Component B outputs. Block builders aggregate per-transaction Circle STARK proofs into a single per-block proof. This is when the full TPS benefit materializes — expensive per-transaction cryptographic verification is replaced by a single aggregated STARK proof check per block, with only ~160 bytes of public inputs per transaction on-chain.

Each phase is independently valuable and can be deployed on its own timeline.

## Test Vectors

*To be provided in a subsequent revision.*

Test vectors will include:
- Hybrid KEM handshake vectors (Component A)
- Identity commitment derivation vectors from known REV values (Component B)
- ZK-ACE circuit witness/proof/public-input vectors (Component C)
- Cross-component integration vectors

## Reference Implementation

The ZK-ACE prover and verifier are implemented in Rust using the Stwo Circle STARK framework (v2.2.0) over the Mersenne-31 field. The implementation covers:

- Poseidon2 hash function (width-16, α=5, M31 field) for all in-circuit and native commitments
- AIR constraint system (~240 constraints) enforcing C1–C5
- Circle STARK prover with FRI-based polynomial commitment (Blake2s Merkle trees)
- Circle STARK verifier with ~128-bit conjectured security
- Both replay-prevention modes (nonce registry and nullifier set)
- Native commitment computation for off-chain/wallet use
- Bincode-based proof serialization

Measured performance (single-threaded, Apple M-series):

| Metric | NonceRegistry | NullifierSet |
|--------|--------------|-------------|
| Prove | ~15 ms | ~20 ms |
| Verify | ~1.1 ms | ~1.2 ms |
| Individual proof size | ~105 KB | ~112 KB |

Source: https://github.com/acechain-io/zk-ace

A reference implementation for the remaining components will cover:
- Hybrid ML-KEM + ECDH handshake module for Bitcoin Core's P2P layer (Component A)
- SegWit v3 output creation and spending logic (Component B)

The ACE-GF reference implementation is available at: https://github.com/ya-xyz/acegf-playground

## Acknowledgements

This draft builds upon the following research:

- [1] J. S. Wang. ACE-GF: A Generative Framework for Atomic Cryptographic Entities. arXiv:2511.20505, 2025.
- [2] J. S. Wang. ZK-ACE: Identity-Centric Zero-Knowledge Authorization for Post-Quantum Blockchain Systems. arXiv:2603.07974, 2026.
- [3] J. S. Wang. AR-ACE: ACE-GF-based Attestation Relay for PQC — Lightweight Mempool Propagation Without On-Path Proofs. arXiv:2603.07982, 2026.
- [4] J. S. Wang. SA-Migration: Zero-Movement Credential Upgrade via Sealed Artifact Encapsulation. Manuscript in preparation, 2026.

The author acknowledges the BIP-360 (QuBit) team for advancing the PQC conversation within the Bitcoin community and establishing the P2MR output type as a practical near-term quantum mitigation.

## References

- **FIPS 203:** Module-Lattice-Based Key-Encapsulation Mechanism Standard (ML-KEM). NIST, 2024.
- **FIPS 204:** Module-Lattice-Based Digital Signature Standard (ML-DSA). NIST, 2024.
- **FIPS 205:** Stateless Hash-Based Digital Signature Standard (SLH-DSA). NIST, 2024.
- **BIP-141:** Segregated Witness (Consensus layer). E. Lombrozo, J. Lau, P. Wuille, 2015.
- **BIP-324:** Version 2 P2P Encrypted Transport Protocol. D. Wuille et al., 2023.
- **BIP-341:** Taproot: SegWit version 1 spending rules. P. Wuille, J. Nick, A. Towns, 2020.
- **BIP-360:** Pay to Merkle Root (P2MR). H. Beast, E. Heilman, I. F. Duke, 2024.
- **RFC 5869:** HMAC-based Extract-and-Expand Key Derivation Function (HKDF). H. Krawczyk, P. Eronen, 2010.
- **RFC 8439:** ChaCha20 and Poly1305 for IETF Protocols. Y. Nir, A. Langley, 2018.
