# Rethinking the Blockchain Trust Stack

Revisiting long-standing assumptions in blockchain through the lens of identity–authorization separation.

## Preface

Over the past decade-plus of blockchain development, many early design choices have solidified into unquestioned industry defaults — private key possession equals ownership, every transaction must carry a signature, wallet recovery depends on mnemonic phrases, and cross-chain means bridging. These assumptions were reasonable in the Bitcoin era, but as post-quantum cryptography, AI agent economies, and institutional compliance requirements emerge, they are becoming systemic bottlenecks.

ACE-GF (Atomic Cryptographic Entity Generative Framework) proposes a seemingly small but structural adjustment: **separating identity binding from per-transaction authorization**. This article follows that separation through several deeply entrenched assumptions in the blockchain stack, exploring what design space opens up when we stop taking them for granted.

---

## Rethink 1: Control ≠ Holding a Private Key

**Industry default:** Control over digital assets is equated with permanent possession of a private key. Transferring control means transferring the key itself.

**The problem:** This model cannot express a wide range of real-world control scenarios — conditional delegation, time-limited authorization, revocable agency, multi-party activation. Inheritance, regulatory freezes, and institutional governance all require control that activates only under specific conditions and can be revoked when those conditions are not met. Existing solutions (multisig, smart contracts, secret sharing) either require all parties to be continuously online or introduce on-chain state mutations and oracle dependencies, essentially pushing conditional control outside the cryptographic trust boundary.

**After separation:** Once the identity root is decoupled from authorization credentials, control rights can be modeled as "dormant authorization paths" (CT-DAP) — composed of user-held credentials and administrative authorization factors, cryptographically nonexistent until all factors are simultaneously present. Revocation is achieved by destroying authorization factors, without touching the identity root or modifying on-chain state. Control shifts from "do you have the key" to "are the activation conditions met."

**Industry significance:** As RWA tokenization and institutional capital enter the crypto asset space, "conditional control" is moving from a theoretical requirement to a compliance necessity. Expressing conditional activation and stateless revocation at a purely cryptographic level means compliance logic can be embedded in the cryptographic structure itself, rather than relying on external contracts or custodians' good-faith execution.

---

## Rethink 2: Transaction Authorization ≠ Carrying a Signature

**Industry default:** Every transaction must carry a cryptographic signature, and consensus nodes verify each signature individually.

**The problem:** This model worked well in the classical cryptography era but faces structural bottlenecks in two directions. First, verification cost scales linearly with transaction volume — O(N) signature verification is a core bottleneck for high-performance chains, forcing Solana to require every validator to run a GPU. Second, post-quantum signatures (e.g., ML-DSA) are approximately 2.4 KB, roughly 38× larger than Ed25519's 64 bytes. Directly posting them on-chain would make rollup calldata and L1 block sizes unacceptable.

A widely discussed mitigation is compressing post-quantum signatures with zero-knowledge proofs — verifying signatures inside ZK circuits and publishing only the succinct proof on-chain. However, this essentially moves expensive lattice arithmetic into the prover circuit, shifting the scalability burden from on-chain data to off-chain computation.

**After separation:** What the consensus layer actually needs is not the signature itself, but the semantic guarantee that "this transaction was authorized by the correct identity." ZK-ACE accordingly replaces signature objects with identity-bound zero-knowledge authorization statements — the prover demonstrates that their identity is consistent with an on-chain commitment and the transaction has been authorized, without exposing any signature. Block verification shifts from "one signature per transaction" to "one proof per block," reducing O(N) to O(1). The post-quantum signature bloat problem is resolved at its root — not by compressing signatures, but by removing signatures from the consensus path entirely.

**Industry significance:** Post-quantum migration is a universally acknowledged necessity, but the industry lacks viable paths that do not introduce significant performance degradation. "Bypassing signatures to prove authorization directly" offers a structural approach rather than incremental optimization within the signature framework.

---

## Rethink 3: Wallet Recovery ≠ Backing Up a Seed

**Industry default:** Deterministic wallet recovery relies on persistent backup of a master seed — typically stored as a mnemonic phrase.

**The problem:** The mnemonic user experience is catastrophic. It is essentially a human-readable encoding of a high-entropy key, requiring users to securely store 12–24 words on a physical medium with zero tolerance for error. Numerous users have permanently lost assets due to lost mnemonics. More fundamentally, the master seed is a single point of failure — once compromised, all keys derived from it, past and future, are irreversibly exposed.

Passkeys/WebAuthn improve local authentication but do not constitute a complete decentralized recovery protocol. Smart contract recovery (e.g., social recovery) requires pre-configured guardians, is visible on-chain, and is chain-specific.

**After separation:** ACE-GF first eliminates the concept of a "master seed" — the identity root exists only ephemerally in memory, deterministically reconstructed from a sealed artifact and authorization credentials, never persistently stored. On this foundation, VA-DAR builds a decentralized discovery-and-recovery layer: users need only an identifier (e.g., email) and a recovery passphrase for cross-device recovery, while the registry's discovery identifier is passphrase-derived, preventing attackers from enumerating wallet holders by traversing identifiers.

The prerequisite for recovery is no longer "you safeguarded that piece of paper," but "you remember your email and passphrase."

**Industry significance:** Wallet recovery is one of the last barriers to cryptocurrency mass adoption. A recovery scheme that is vendor-agnostic, chain-agnostic, enumeration-resistant, and PQC-ready directly impacts the onboarding path for the next billion users.

---

## Rethink 4: Cross-Chain ≠ Bridging

**Industry default:** Users maintain different key pairs and addresses on different chains, and cross-chain asset transfers rely on bridges — third-party systems that lock, mint, and verify.

**The problem:** Bridges are the single largest source of security failures in decentralized finance. Between 2021 and 2024, bridge exploits resulted in losses exceeding $2.8 billion. The root cause is identity fragmentation: the same user is an entirely different entity on EVM, SVM, Bitcoin, and TVM, and cross-chain operations must establish trust between these different entities through external systems.

Existing multi-VM projects (Movement Labs, Eclipse, Sei v2) attempt to run multiple execution environments on a single chain, but typically subordinate one VM to another. Identity remains fragmented across VMs, and token transfers still require bridge-like mechanisms.

**After separation:** When the identity layer is decoupled from the execution environment, a single 32-byte identity commitment (id_com) can deterministically generate native addresses for each VM through context-isolated HKDF derivation. n-VM builds on this to construct a multi-VM Layer-1 architecture where EVM, SVM, BVM, and TVM coexist as co-equal execution environments on the same chain, sharing consensus, state, and a unified token ledger. Cross-VM token transfers are atomic operations on the same ledger, not bridge transactions.

**Industry significance:** Cross-chain interoperability is a decade-old unsolved problem. n-VM demonstrates that starting from identity-layer unification, the "cross-chain" problem can be redefined as "cross-VM calls on the same chain," architecturally eliminating the need for bridges.

---

## Rethink 5: Agent Autonomy ≠ Economic Sovereignty

**Industry default:** When AI agents participate in economic activity, they are either fully trusted (given a wallet) or fully untrusted (every transaction manually approved).

**The problem:** This binary model is unsustainable as agent capabilities grow rapidly. Full trust means an agent can unilaterally drain a wallet — an unacceptable risk. Per-transaction approval negates the entire value of agent automation — if every operation requires human confirmation, the agent is nothing more than a glorified form-filling tool.

**After separation:** Identity–authorization separation naturally provides a foundation for layered control over agent economic behavior. AESP defines the central invariant "economically capable but never economically sovereign": agents possess cryptographic identities and transaction capabilities, but every economic action must pass through deterministic policy gates (per-transaction limits, address allowlists, budgets, time windows), with actions that fail policy checks escalated to human approval. Each transaction uses an HKDF-derived ephemeral address, and different agent contexts are cryptographically isolated — the transaction patterns of a user's procurement agent and investment agent cannot be correlated by on-chain observers.

**Industry significance:** AI agents are evolving from information-processing tools into economic actors. What the industry needs is not "bigger wallets for agents" or "more frequent human approvals," but a protocol-level boundary definition — specifying what agents can do autonomously, what they cannot, and how to escalate when boundaries are exceeded. The absence of standardization at this layer will become a structural barrier to scaling the agent economy.

---

## Rethink 6: Verification ≠ Every Node Doing the Same Work

**Industry default:** In blockchain networks, every node performs the same verification workflow for every transaction — including signature verification, transaction execution, and state transition.

**The problem:** This model leads to resource waste at two levels. At the network layer, if every propagated object carries a full validity proof (approximately 128 KB in post-quantum settings), relay bandwidth grows linearly with object count. At the execution layer, every validator independently performs O(N) signature verification, and in post-quantum settings must handle bloated signature data.

**After separation:** Identity–authorization separation enables verification responsibilities to be stratified by role. AR-ACE implements "proof-off-path" at the network layer — relay nodes verify only lightweight attestations to decide whether to forward, and full validity proofs are generated only once by the builder at the final stage. ACE Runtime replaces cryptographic signature verification at the execution layer with HMAC attestation (~1 μs per transaction), moving zero-knowledge proof generation off the critical path.

The result is a layered verification architecture: relay nodes perform the lightest admission checks, validators handle attestation verification and transaction execution, and builders perform aggregated proof generation. Each role performs only the verification work appropriate to its responsibilities.

**Industry significance:** Blockchain scalability bottlenecks largely stem from the implicit assumption that "all nodes do the same work." Layered verification is not a new concept (proposer-builder separation is already a trend), but identity–authorization separation provides a clean cryptographic foundation for this stratification, rather than an architectural workaround.

---

## Rethink 7: Paying Someone ≠ Knowing Their Address

**Industry default:** Sending cryptocurrency requires the sender to know the recipient's on-chain address. Human-friendly identifier schemes (e.g., ENS, Solana Name Service) map identifiers directly to addresses via public registries, making the mapping globally visible.

**The problem:** Opaque hexadecimal addresses are a primary UX barrier to cryptocurrency adoption. Yet naïve identifier-to-address mapping introduces a critical privacy vulnerability: anyone who knows a user's email, phone number, or social handle can derive their on-chain address and inspect their entire transaction history, balances, and counterparty relationships. For public figures, a single known identifier can reconstruct a complete financial profile across every supported chain.

Stealth-address schemes (e.g., ERC-5564) mitigate linkability but still require the sender to know the recipient's public key or meta-address, and the recipient must scan every transaction to detect incoming payments. Neither approach achieves the UX simplicity of "send to email" with full privacy.

**After separation:** HFIPay decouples identifier routing, sender-side quote verification, and on-chain claim authorization. The sender specifies only a human-friendly identifier; the relay privately resolves it off-chain, derives a one-time deposit address, and commits only a per-intent blinded binding plus the quoted payment tuple on-chain. The chain sees neither the identifier nor a reusable recipient tag. The recipient later claims by proving in zero knowledge — via ZK-ACE — that the funded intent's blinded binding matches a handle derived from the same deterministic identity, authorizing release to a chosen destination. In a verified-quote deployment, the relay furnishes a sender-verifiable proof that the quoted binding is tied to the attested recipient, preventing substitution before funding.

**Industry significance:** "Send crypto to an email address" is a long-sought UX goal, but every prior approach sacrificed either privacy (public registries) or decentralization (custodial intermediaries that can redirect funds). HFIPay achieves the UX target while keeping identifier resolution private, claim authorization on-chain, and the relay non-custodial — relays are privacy and availability dependencies, but cannot redirect funds. When composed with n-VM, the same claim mechanism extends to cross-chain settlement, making "send to email on any chain" a single unified flow.

---

## Architecture

The seven rethinks above are not independent observations but logical consequences of the same structural separation applied at different levels of the blockchain stack:

```
        Identity–Authorization Separation
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
  Identity Layer  Authorization  Application
       │          Layer              │
       ├─ ACE-GF  ├─ ZK-ACE     ├─ CT-DAP (Asset Control)
       ├─ VA-DAR  ├─ AR-ACE     ├─ AESP (Agent Economics)
       │          ├─ ACE Runtime ├─ HFIPay (Payments)
       │          │              │
       └──────────┴───────┬──────┘
                          ▼
                   n-VM (Multi-VM)
```

One primitive, one separation, seven rethinks, covering the full trust stack from key management to payment experience.

---

## Open Questions

Any challenge to established paradigms must honestly face its own uncertainties:

- **Engineering feasibility.** Performance figures in the papers come from analytical modeling and component-level benchmarks. The engineering complexity of an end-to-end system — state management, error handling, debugging toolchains — has not yet been validated in production.
- **Ecosystem adoption path.** Restructuring at the identity layer means this cannot be deployed as an incremental upgrade to existing chains. It must be built as a new chain or Layer-2 from scratch, significantly raising the bar for market adoption.
- **Operational reliability.** In the seed-storage-free model, simultaneous availability of the sealed artifact and authorization credentials is a prerequisite for system availability. Real-world performance at scale — including credential loss rates, recovery success rates, and user cognitive burden — requires deployment data.
- **Security under adversarial conditions.** Formal proofs are based on mature standard assumptions, but the security of composed protocols under adversarial conditions still requires independent audits and long-term operational validation.

---

## Conclusion

Many "impossibilities" in the blockchain industry are, at their core, "impossibilities under current assumptions." When we revisit these assumptions — private key as control, signature as authorization, seed as identity, bridge as cross-chain, address as payment target — and systematically explore the consequences of identity–authorization separation, some limitations previously considered fundamental begin to reveal structural resolution paths.

This does not mean these paths will necessarily lead to success. The distance from papers to production, from proof of concept to ecosystem building, is often longer than the papers themselves. But at the very least, it is worth pausing to reconsider: are the things we take for granted truly the only options?

---

*All papers referenced in this article are available in the [papers repository](https://github.com/ya-xyz/papers). Each paper is an arXiv preprint — see the README for a full listing with summaries.*
