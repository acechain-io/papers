# Research Papers

A collection of research papers on cryptographic identity, post-quantum blockchain systems, and AI agent economics.

## Papers

### ACE-GF: A Generative Framework for Atomic Cryptographic Entities
- **File:** `2511.20505v2.pdf`
- **Author:** Jian Sheng Wang
- **Date:** February 26, 2026
- **Summary:** Presents ACE-GF (Atomic Cryptographic Entity Generative Framework), a seed-storage-free identity construction that enables deterministic and context-isolated key derivation without storing any master secret at rest. The identity root is reconstructed ephemerally in memory from a sealed artifact and authorization credentials, supporting stateless credential rotation, authorization-bound revocation, and non-disruptive migration toward post-quantum cryptographic domains.

### AESP: A Human-Sovereign Economic Protocol for AI Agents with Privacy-Preserving Settlement
- **File:** `2603.00318v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 27, 2026
- **Summary:** Presents the Agent Economic Sovereignty Protocol (AESP), a layered protocol enabling AI agents to transact autonomously at machine speed on crypto-native infrastructure while remaining cryptographically bound to human-defined governance boundaries. Enforces the invariant that agents are economically capable but never economically sovereign through policy-gated execution, human-in-the-loop review, EIP-712 commitments with escrow, HKDF-based context-isolated privacy, and an ACE-GF cryptographic substrate.

### VA-DAR: A PQC-Ready, Vendor-Agnostic Deterministic Artifact Resolution for Serverless, Enumeration-Resistant Wallet Recovery
- **File:** `2603.02690v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 3, 2026
- **Summary:** Presents VA-DAR, a keyed-discovery protocol for ACE-GF-based wallets that uses device-bound passkeys for day-to-day local unlock while supporting cross-device recovery using only a user-provided identifier and a single recovery passphrase. Introduces a decentralized discovery-and-recovery layer that maps a privacy-preserving discovery identifier to a backup sealed artifact stored on decentralized storage, with computational resistance to public-directory enumeration.

### Condition-Triggered Cryptographic Asset Control via Dormant Authorization Paths
- **File:** `2603.07933v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 10, 2026
- **Summary:** Introduces Condition-Triggered Dormant Authorization Paths (CT-DAP), a cryptographic asset control method built on destructible authorization factors. Asset control rights are modeled as dormant authorization paths that remain cryptographically inactive until all required authorization factors are simultaneously available. Supports conditional activation tied to external events (inheritance, regulatory supervision) and stateless revocation by destroying authorization factors.

### ZK-ACE: Identity-Centric Zero-Knowledge Authorization for Post-Quantum Blockchain Systems
- **File:** `2603.07974v2.pdf`
- **Author:** Jian Sheng Wang
- **Date:** February 26, 2026 (revised March 13, 2026)
- **Summary:** Presents ZK-ACE (Zero-Knowledge Authorization for Cryptographic Entities), an authorization layer that replaces transaction-carried signature objects with identity-bound zero-knowledge authorization statements. Instead of proving the correctness of a post-quantum signature, the prover demonstrates in zero knowledge that a transaction is authorized by an identity consistent with an on-chain commitment, achieving an order-of-magnitude reduction in consensus-visible authorization data.

### AR-ACE: ACE-GF-based Attestation Relay for PQC — Lightweight Mempool Propagation Without On-Path Proofs
- **File:** `2603.07982v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 9, 2026
- **Summary:** Presents AR-ACE, a propagation design for post-quantum blockchain mempools in which relay nodes forward objects plus compact attestations (identity-bound signatures or commitments) without generating, holding, or forwarding full validity proofs. Only the builder performs a single aggregated validity proof, removing proof overhead from the propagation path entirely and yielding an order-of-magnitude reduction in proof-related relay bandwidth.

### ACE Runtime — A ZKP-Native Blockchain Runtime with Sub-Second Cryptographic Finality
- **File:** `2603.10242v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 10, 2026
- **Summary:** Presents ACE Runtime, a blockchain execution layer built upon the ACE-GF identity–authorization separation primitive. Features a three-phase Attest–Execute–Prove pipeline architecture where authorization verification is shifted from cryptographic signatures to lightweight HMAC-based attestations (~1 μs per transaction), while zero-knowledge proof generation is moved off the critical path. Targets 400 ms block times, ~600 ms hard cryptographic finality, and eliminates the GPU requirement for non-builder validators.

### HFIPay: Privacy-Preserving, Cross-Chain Cryptocurrency Payments to Human-Friendly Identifiers
- **File:** `2603.26970v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 27, 2026
- **Summary:** Presents HFIPay, a relay-assisted payment protocol that lets users send cryptocurrency to human-friendly identifiers (email addresses, phone numbers, social handles) while keeping the identifier-to-address mapping private. Separates private routing, sender-side quote verification, and on-chain claim authorization: a relay resolves the identifier off-chain, commits only a per-intent blinded binding plus the quoted payment tuple, and the recipient claims funds by proving in zero knowledge (via ZK-ACE) that the funded intent's blinded binding matches a handle derived from the same deterministic identity. Formalizes enumeration resistance and pre-claim unlinkability, distinguishes baseline from verified-quote deployments, and composes with n-VM for cross-chain settlement.

### n-VM: A Multi-VM Layer-1 Architecture with Shared Identity and Token State
- **File:** `2603.23670v1.pdf`
- **Author:** Jian Sheng Wang
- **Date:** March 23, 2026
- **Summary:** Presents n-VM, a Layer-1 architecture that hosts n heterogeneous virtual machines (EVM, SVM, Bitcoin Script, TVM, Native) as co-equal execution environments over shared consensus and shared state. The design combines a dispatcher that routes transactions by opcode prefix, a unified identity layer in which one 32-byte commitment anchors VM-specific addresses, and a unified token ledger that exposes VM-native interfaces (ERC-20, SPL) over a common balance store. Proves cross-VM transfer atomicity and identity isolation under standard cryptographic assumptions, with a projected throughput of 16,000–66,000 TPS on commodity hardware.
