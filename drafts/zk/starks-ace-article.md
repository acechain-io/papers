# STARKs and ZK/AR-ACE: A Natural Alliance for the Post-Quantum Era

*Why the most powerful proof system and the most structural rethink in blockchain are made for each other — and why it matters for the next twenty years.*

## Summary

STARKs — scalable, transparent, post-quantum-secure proofs of computational integrity — are among the most significant cryptographic inventions of the past decade. Yet in today's blockchain landscape, they are deployed almost exclusively as optimization tools: compressing rollup state, batching signature verifications, shrinking on-chain data. Powerful as these applications are, they assign STARKs a subordinate role — serving primitives (signatures, state roots) that remain architecturally central.

Identity–authorization separation, as instantiated in ZK-ACE and AR-ACE, changes this relationship. By removing signatures from the consensus path entirely, it creates an architecture where the proof system *is* the authorization primitive — not a wrapper around one. In this model, every transaction's authorization flows through a STARK proof. Block verification collapses from O(N) per-transaction signature checks to a single O(1) aggregated proof. And STARKs' three defining properties — scalability, transparency, and post-quantum security — shift from nice-to-have optimizations to consensus-critical parameters.

This article argues that this pairing is not incidental. STARKs and identity–authorization separation are structural complements: each resolves the other's core limitation. Together, they define an architecture that is natively ready for the post-quantum era — not as a migration target, but as a starting point.

---

## The Irony of How We Use STARKs Today

Eli Ben-Sasson and his co-authors introduced STARKs in 2018 with a specific vision: scalable, transparent arguments for computational integrity. No trusted setup. No elliptic curve assumptions. Verification time polylogarithmic in the computation size. Prover time quasilinear. And security grounded in collision-resistant hash functions — a foundation that remains secure against known quantum algorithms.

These properties make STARKs arguably the most future-proof proof system available. And yet, look at how the industry deploys them:

- **Rollup compression.** STARKs verify that a batch of L2 transactions was executed correctly, so that L1 doesn't have to re-execute them. The proof attests to the integrity of *someone else's computation*.
- **Signature aggregation.** Multiple ECDSA or EdDSA signatures are verified inside a STARK circuit, producing a single proof that "N valid signatures exist." The proof attests to the validity of *another cryptographic primitive*.
- **State compression.** Merkle proofs, storage proofs, and state diffs are wrapped in STARK proofs for succinct on-chain verification. The proof makes *existing data structures* smaller.

Every one of these is a legitimate, valuable application. But notice the pattern: in each case, the STARK proof is in service of something else. The signature is the trust anchor. The state root is the canonical reference. The L1 consensus mechanism is the final arbiter. STARKs make these things more efficient, but they do not replace them.

Consider the post-quantum signature problem. ML-DSA (the NIST-standardized lattice-based signature scheme) produces signatures of approximately 2,420 bytes — 38× larger than Ed25519. The natural response is: verify the ML-DSA signature inside a STARK circuit, post the succinct proof on-chain, problem solved.

But look at what this actually does. The lattice arithmetic that makes ML-DSA quantum-resistant is computationally expensive. Moving it inside a STARK circuit does not eliminate that cost — it shifts it from the verifier to the prover. The prover must now perform full ML-DSA verification *plus* the overhead of arithmetization and proof generation. We have traded on-chain bloat for off-chain computational bloat. The signature — the original source of the problem — remains at the center of the architecture.

The STARK, in this model, is doing janitorial work: cleaning up after a primitive that was not designed for the environment it now operates in. It is as if we built a Formula One engine and used it to tow a trailer — impressive engineering, misapplied.

---

## What Changes When You Remove the Signature

Identity–authorization separation, as proposed in the ACE-GF framework, makes a structural adjustment: the identity root (a deterministic reconstruction from a sealed artifact and credentials) is decoupled from per-transaction authorization. Authorization becomes a separate, pluggable layer.

ZK-ACE takes this separation to its logical conclusion on the consensus path. Instead of generating a signature and then compressing it into a proof, ZK-ACE removes the signature entirely. The prover demonstrates in zero knowledge that:

1. Their identity is consistent with an on-chain commitment.
2. The transaction has been authorized according to the policy governing that identity.

No signature is generated. No signature is transmitted. No signature is verified — not on-chain, not off-chain, not inside a circuit. The proof does not attest that "a valid signature exists somewhere." It directly attests that "this transaction is authorized by this identity."

This is not a compression of the same statement. It is a *different statement* — a simpler one, a more direct one, and one that is native to what proof systems do best: proving computational claims.

The moment you remove the signature, several things happen simultaneously:

**The proof system moves from periphery to center.** In the signature-compression model, the trust chain is: identity → key → signature → proof. The proof is the outermost layer, furthest from the trust anchor. In the direct-authorization model, the trust chain is: identity → commitment → proof. The proof *is* the authorization. It is not optimizing another primitive; it is the primitive.

**Post-quantum bloat becomes irrelevant.** ML-DSA's 2,420-byte signatures do not need to be compressed, because they do not exist. The on-chain footprint is determined by the proof size and the commitment size — both independent of the underlying signature scheme's parameters.

**Prover efficiency improves structurally.** Proving "I know credentials that reconstruct an identity consistent with this commitment, and I authorize this transaction" is a fundamentally lighter statement than proving "I verified a valid ML-DSA signature." The lattice arithmetic that dominates ML-DSA verification is no longer inside the circuit. The prover's work is proportional to the complexity of the authorization statement, not the complexity of a signature verification.

---

## O(N) to O(1): The Block-Level Collapse

ZK-ACE handles individual transaction authorization. The block-level collapse emerges from the full stack: AR-ACE optimizes the relay layer (proof-off-path propagation, so mempool bandwidth does not scale with proof size), while ACE Runtime's Attest–Execute–Prove pipeline aggregates all per-transaction authorization proofs into a single block-level proof at the builder stage.

In today's blockchains, block verification requires O(N) independent signature checks — one per transaction. This is a fundamental bottleneck. Solana, for example, requires every validator to run a GPU dedicated to parallelizing Ed25519 signature verification. Post-quantum signatures would make this dramatically worse: each verification becomes ~10× more expensive in compute, and the data to verify is ~38× larger.

Aggregation schemes exist (BLS signatures, for instance), but they are limited to specific signature schemes, require a pairing-friendly curve, and are not post-quantum secure.

The ZK-ACE + ACE Runtime stack takes a different approach. Since every transaction's authorization is already a zero-knowledge proof (via ZK-ACE), the block builder can aggregate all per-transaction proofs into a single block-level proof. One proof covers the entire block. Validators verify one proof, regardless of how many transactions the block contains.

**O(N) → O(1).**

This is not achievable in the signature-based model, because signatures are scheme-specific objects that do not compose naturally. You cannot take N Ed25519 signatures and produce a single "super-signature" without changing the scheme (and even BLS aggregation, which supports this, is not post-quantum secure). But zero-knowledge proofs *do* compose: a proof of N valid proofs is itself a proof. Recursive composition is a native property of proof systems.

The implication is precise: **the O(N) → O(1) collapse is only possible because signatures were removed.** It is not a feature of better signature schemes or faster verification hardware. It is a structural consequence of making the proof system the authorization primitive.

---

## Why STARKs — Specifically

The direct-authorization model does not technically require STARKs. Any succinct proof system could, in principle, fill this role. But STARKs possess three properties that make them the most natural fit for a *consensus-critical* authorization layer in a post-quantum blockchain:

### 1. Post-Quantum Security by Construction

STARKs rely on collision-resistant hash functions — no elliptic curves, no pairings, no lattice assumptions. Their security is not "believed to be quantum-resistant" in the way that lattice-based schemes are; it rests on a problem (finding hash collisions) for which no quantum speedup beyond Grover's square root is known, and the hash output length can be adjusted to compensate.

In the direct-authorization model, this matters enormously. If the proof system is the authorization primitive — if every transaction's validity depends on it — then the proof system's security assumptions become the chain's security assumptions. A proof system that requires a trusted setup, or relies on discrete-log hardness, or depends on pairing assumptions, introduces quantum vulnerability at the most critical layer of the stack.

STARKs do not introduce these vulnerabilities. The consensus path is post-quantum secure not because of a migration, but because the foundational primitive was never classically dependent.

### 2. Transparency (No Trusted Setup)

SNARKs that require a structured reference string (SRS) — Groth16, original PLONK, KZG-based systems — introduce a trust assumption at the very foundation of the proof system. If the SRS generation ceremony is compromised, proofs can be forged.

In a rollup context, this risk is bounded: the worst case is a fraudulent state transition, which can be caught by other mechanisms. But in the direct-authorization model, a forged proof means a forged *authorization* — the ability to spend someone else's assets. The trust assumption becomes existential.

STARKs eliminate this entirely. Verification requires only public randomness (derived from the proof itself via Fiat-Shamir), not a pre-generated secret. The authorization layer has no trapdoor.

### 3. Scalability for Block-Level Aggregation

AR-ACE's O(1) block verification relies on recursive proof composition: the block builder generates a proof that attests to the validity of all per-transaction proofs. The efficiency of this recursion is determined by the proof system's verification complexity — because the "inner" verification becomes the circuit that the "outer" proof must prove.

STARK verification is polylogarithmic in the computation size. This means the recursive overhead grows slowly as block sizes increase. For a system targeting hundreds of thousands of transactions per block, this scaling property is not an optimization — it is a structural requirement.

---

## The Natural Alliance

The pairing between STARKs and identity–authorization separation is not a coincidence of design choices. It reflects a deeper structural alignment:

| | **STARKs provide** | **Identity–auth separation requires** |
|---|---|---|
| **Security** | Hash-based, post-quantum by construction | Authorization layer that survives quantum transition without migration |
| **Trust model** | No trusted setup, no trapdoor | Authorization primitive where forgery = asset theft; no SRS risk acceptable |
| **Scalability** | Polylogarithmic verification, efficient recursion | Block-level O(1) aggregation via recursive composition |
| **Expressiveness** | General-purpose computational integrity proofs | Authorization statements richer than "valid signature exists" |

Each column was designed independently. STARKs were invented to prove computational integrity at scale. Identity–authorization separation was designed to decouple identity from per-transaction signing. But the requirements of one align remarkably well with the capabilities of the other.

STARKs without identity–authorization separation remain optimization tools — compressing signatures, batching verifications, shrinking data. Powerful, but peripheral.

Identity–authorization separation without STARKs could work with other proof systems, but would inherit their limitations: trusted setup risks on the consensus-critical path, quantum vulnerability in the authorization layer, or verification overhead that undermines block-level aggregation.

Together, they form an architecture where:

- **Every transaction** is authorized by a STARK proof — no signatures on the consensus path.
- **Every block** is verified by a single aggregated STARK proof — O(1) regardless of transaction count.
- **The entire consensus path** is post-quantum secure by construction — no migration needed when quantum computers arrive.
- **The authorization layer** has no trapdoor, no trusted setup, no ceremony — the security model is as clean as the hash function it rests on.

---

## A Twenty-Year View

The post-quantum transition is not a distant hypothetical. NIST finalized its first post-quantum standards in 2024. The NSA's CNSA 2.0 suite requires transition to post-quantum algorithms by 2035. "Harvest now, decrypt later" attacks mean that data encrypted or signed today is already at risk from future quantum capabilities.

For blockchains, the stakes are especially high. Every transaction ever recorded on a public ledger is permanently visible. Every public key ever exposed is a potential target. And the immutability that makes blockchains trustworthy also makes them impossible to retroactively patch.

The industry's current approach — migrate to post-quantum signature schemes — addresses the immediate vulnerability but inherits all the structural costs: bloated signatures, expensive verification, O(N) scaling, and the certainty that another migration will be needed when today's "post-quantum" schemes are broken or superseded.

The STARK + identity–authorization separation architecture offers a different posture: **build on a foundation that does not need to migrate.** Hash-based proof systems do not become obsolete when a new quantum algorithm is discovered. They do not require parameter updates when NIST revises its security levels. They do not force asset migration when cryptographic generations change. The identity root is algorithm-agnostic; the authorization layer is proof-based; the proof system is hash-based. The entire stack is designed to outlast any individual cryptographic assumption.

This is not a claim that the architecture is permanent or beyond improvement. It is a claim that its security model degrades gracefully and upgrades cleanly — that the twenty-year horizon is one it can navigate without the kind of disruptive migrations that signature-based systems will inevitably face.

Twenty years from now, the blockchains that survive will not be the ones that migrated fastest to post-quantum signatures. They will be the ones that were built on primitives that did not require migration in the first place.

---

## Open Questions

- **Prover cost at scale.** Direct authorization proofs are structurally lighter than signature-verification proofs, but absolute prover performance for hundreds of thousands of transactions per block remains an engineering challenge. Hardware acceleration (FPGAs, ASICs for NTT) is an active area, but production readiness is unproven.
- **Recursive proof overhead.** AR-ACE's block-level aggregation relies on efficient recursive composition. The concrete overhead of STARK-in-STARK recursion at production block sizes needs benchmarking beyond analytical models.
- **Ecosystem adoption.** This architecture cannot be retrofitted onto existing chains. It requires new infrastructure — wallets, SDKs, developer tooling — built around proof-based authorization rather than signature-based authorization. The cold-start problem is real.
- **Proof system evolution.** STARKs are not static. Circle STARKs, DEEP-FRI, and other advances continue to improve concrete performance. The architecture should be designed to absorb these improvements without structural changes — but validating this adaptability requires implementation experience.

---

## Conclusion

STARKs were invented to prove computational integrity at scale, without trusted setup, with security rooted in hash functions. Identity–authorization separation was designed to decouple identity from per-transaction signing, making the proof system — rather than the signature — the authorization primitive. Neither was designed with the other in mind. But their structural alignment is striking: what one provides maps closely to what the other requires.

The result is an architecture where STARKs are no longer doing janitorial work — compressing bloated signatures, batching redundant verifications, optimizing around primitives that were not designed for the post-quantum world. Instead, they occupy the role they were built for: scalable, transparent, quantum-resistant proofs of computational claims. The claim is simply: "this transaction is authorized by this identity." The mathematics does not change. What changes is whether we ask the mathematics to serve a peripheral role or a foundational one.

For the next twenty years — the window in which quantum computing will transition from theoretical threat to practical capability — the blockchain industry faces a choice. It can continue to build on signature-based authorization and absorb the escalating costs of each cryptographic migration. Or it can build on proof-based authorization, where the consensus path is post-quantum secure by construction, block verification is O(1), and the authorization primitive is the most powerful tool in the cryptographer's arsenal, finally deployed at the center of the architecture rather than at its edges.

The alignment between STARKs and identity–authorization separation is not a product decision. It is a structural observation: these two ideas, developed independently, address each other's core limitations. Recognizing that — and building on it — may open a cleaner path to the next generation of blockchain infrastructure.

---

*This article builds on the architecture described in [Rethinking What We Assumed About Blockchain](https://paragraph.com/@0xjasonw/rethinking-everything-we-assumed-about-blockchain). The underlying protocols — [ZK-ACE](https://arxiv.org/abs/2603.07974), [AR-ACE](https://arxiv.org/abs/2603.07982), and [ACE Runtime](https://arxiv.org/abs/2603.10242) — are available on arXiv.*

*Contact: jason@acechain.io · [@0xJasonw](https://x.com/0xJasonw)*
