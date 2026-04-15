# From Public-Key Binding to Identity-Authorization Separation: Why We Need a Third-Generation Blockchain

Blockchain technology has undergone three paradigm shifts. The first generation (Bitcoin) achieved decentralized value transfer. The second generation (Ethereum) introduced programmable smart contracts. Yet for over a decade, a set of deep structural problems has remained unsolved — not for lack of engineering effort, but because the foundational architectural assumptions of the first two generations lock out any real solution.

This article examines five industry-level problems, explains why a paradigm reconstruction is necessary, and describes how ACE Chain uses identity-authorization separation as a foundational breakthrough to unlock the solution space for all five simultaneously.

## I. Five Persistent Problems

### 1. The Identity Model: Public Key = Identity, Structurally Locked

BTC, ETH, Solana — every existing public blockchain derives account addresses from public keys. The public key serves as both the identity identifier and the authorization credential, welded together.

This means: **changing your key = changing your identity = changing your address = migrating all assets and contract relationships.**

This is not implementation-level technical debt. It is a protocol-level design assumption. The first two generations built their entire ecosystems on this assumption, and it cannot be changed. The four problems that follow are all downstream consequences of this single root cause.

### 2. Post-Quantum Security: It's Not Just "Adding an Algorithm"

Facing the quantum computing threat, the industry's mainstream plan is "migrate to post-quantum signature algorithms in the future." But the NIST post-quantum signature standard ML-DSA-65 produces 3,309-byte signatures and 1,952-byte public keys. Placing these directly on-chain would collapse throughput back to baseline.

More critically: **public keys are already exposed in on-chain history, and quantum computers can retroactively break them.** Switching algorithms does not solve the problem of already-exposed public keys. Ethereum's PQ roadmap calls for "a future hard fork + full ecosystem migration" — a process measured in years, with enormous coordination costs.

### 3. Ecosystem Fragmentation: Bridges Are ATMs for Hackers

The EVM ecosystem and the Solana ecosystem operate in silos, connected by bridges. Bridges are the single most exploited component in blockchain — Wormhole lost $320 million, Ronin lost $620 million. The Cosmos and Polkadot approach is message passing, which is fundamentally still "two independent state machines communicating." Trust assumptions and latency issues persist.

### 4. User Experience: Seed Phrases and Hex Addresses

12 or 24 seed words, irrecoverable if lost, transfer addresses as hexadecimal strings — the first two generations expose raw cryptographic implementation details directly to end users. The industry has been calling for mass adoption for a decade, and this barrier has never come down.

The root cause is the same: public key = identity. Account recovery means recovering the private key. The address is a hash of the public key. There is no abstraction layer that can be inserted in between.

### 5. Private Key Inheritance: The Digital Estate Black Hole

Lose your private key, and your assets are gone forever. On first- and second-generation chains, **self-custody, inheritability, and yield generation cannot be satisfied simultaneously**: hand assets to an exchange and you can inherit but lose self-custody; hold the private key yourself and your heirs get nothing when you die; multi-sig introduces trusted third parties. This is a structural impossibility triangle under the public-key-as-identity architecture.

## II. One Root Cause, One Decoupling

These five problems are not isolated feature requests. They share a single root cause — **the binding of public keys to identity**. As long as this assumption holds, every problem can only be patched at the application layer, treating symptoms rather than the disease.

ACE Chain's core design decision is to dismantle this assumption at the protocol layer: **identity-authorization separation.**

The concrete implementation is the ACE-GF (Atomic Cryptographic Entity Generative Framework) identity primitive. The sole on-chain identity identifier is IdCom — an identity commitment generated as a SHA-256 hash of the identity root entropy (REV) and a salt. Signature algorithms, key pairs, and authorization methods are all replaceable components, deterministically derived from the same REV via HKDF.

**Change your key without changing your identity. Change your algorithm without changing your address.** The identity layer, authorization layer, and execution layer are fully decoupled.

Once this decoupling is in place, the solution space for all five problems opens simultaneously:

**Post-quantum security becomes a native capability.** Public keys are never exposed on-chain (IdCom is a hash, resistant to quantum retroactive attacks). Transaction authentication uses ML-DSA-44, but signatures never go on-chain (attestation verification uses HMAC, at microsecond cost). Block verification uses Stwo Circle STARK (pure hash functions, no elliptic curve dependencies). The entire verification chain — from transaction authentication to block proof — relies on no classical cryptographic hardness assumptions. No hard fork required. No ecosystem migration needed.

**Multi-VM unified execution eliminates bridging.** ACE Chain's n-VM dispatcher runs EVM, SVM, BVM, and TVM under a single state tree. All VMs share a unified AccountId and balance ledger. Cross-VM transfers are in-memory balance changes — no bridges, no cross-chain messaging, zero trust assumptions. Because identity is chain-level (IdCom), not VM-level (public key address), a single identity naturally maps to addresses across all VMs.

**Wallet recovery and human-friendly payments.** The VADAR protocol enables wallet recovery using a password and an email or phone number. Only hashes are stored on-chain — no plaintext identifiers are ever exposed. HFI Pay enables payments to email addresses or phone numbers in a "pay first, claim later" pattern, where the recipient can receive funds before even having a wallet. These are not application-layer wrappers — they are protocol-level native capabilities, made possible only because identity and keys are decoupled, creating space to insert recovery and mapping mechanisms in between.

The industry significance of this is far greater than the technical community typically recognizes. Blockchain adoption faces two barriers standing between ordinary people and the technology: **wallets and payments**. Performance comes third.

The first wall is the wallet. Creating a wallet means generating a private key, writing down 12 or 24 seed words, and understanding that "if you lose them, they're gone forever." More critically, those 12 words are a **bare asset** — no password protection, no second factor, no access control of any kind. Anyone who obtains those 12 words owns all your assets, without your authorization, irrevocably, unrecoverably. This is equivalent to asking every user to write their entire net worth on a slip of paper, then take personal responsibility for ensuring that slip is never seen by anyone and never lost. No successful internet product has ever demanded this level of security responsibility from its users, yet blockchain requires every user to be their own key manager. The vast majority of ordinary people give up at this step.

The second wall is payments. Sending money to a friend requires them to provide a 42-character hexadecimal address (`0x1a2B...`), copy and paste it, and if you get it wrong, the assets are permanently lost. Compare this to: WeChat Pay scans a QR code, Venmo searches a username. Blockchain's payment experience is stuck in the era of direct IP address connections — DNS hasn't been invented yet.

Ordinary people never even get to the point of caring about "is the chain fast enough" — they abandon the process during wallet setup and their first transfer.

The technical community tends to think "can't you just add a registry or build an ENS?" But the problem is: **under the first two generations' architecture, application-layer wrappers cannot solve the fundamental problem.** Public key = identity means account recovery equals private key recovery. The address equals a public key hash. There is no abstraction layer that can be inserted. No matter how you wrap it — custodial wallets, social recovery, MPC sharding — the underlying constraint remains: "whoever holds the private key is the account owner." ENS simply puts a label on a hex address; the underlying layer is still the public key address, and losing your private key means your ENS name can't save you either. It's a binary choice: sacrifice self-custody (hand the key to a third party) or sacrifice usability (make the user manage the key themselves).

After identity-authorization separation, both walls come down at once — and they come down **at the protocol layer**, not through an application wrapper on top.

Wallet recovery becomes "re-derive the identity root with a password and email, bind a new key" — no private key transfer, no third-party custody, no seed phrases. The experience is consistent with the internet's "forgot password → email verification → reset" flow, but the security model is self-custodial: only hashes are stored on-chain, and the recovery process completes locally. VADAR is a native protocol of the chain, not a proprietary solution from a particular wallet vendor — any wallet, any client can invoke the same recovery mechanism.

Payments become "send money to an email address or phone number" — the recipient doesn't even need a wallet yet; they claim first and register later. HFI Pay is likewise a protocol-layer capability, built into the chain's state machine, not a DApp's smart contract. You don't need to know the recipient's public key, just as sending an email doesn't require knowing the recipient's mail server IP address. This is the interaction paradigm that internet users are accustomed to.

The bottleneck to mass adoption has never been "is the chain fast enough." It has always been "can ordinary people use it." If this problem isn't solved at the protocol layer, no amount of application-layer polish will help. Performance is the third wall — but only those who cross the first two walls ever encounter it.

**Private key inheritance becomes a protocol capability.** The identity (IdCom) can be bound to an heir's new key through the VADAR recovery protocol, without transferring assets or exposing the original private key. Self-custody and inheritability are no longer in conflict.

## III. Verification Architecture: From O(N) Signatures to O(1) Proofs

Identity-authorization separation also delivers a significant performance dividend: **signatures vanish from the transaction critical path.**

The verification model of the first two generations is per-transaction signature verification — every transaction requires verifying a digital signature, with costs growing linearly with transaction count. This is a hard constraint on the performance ceiling.

ACE Chain's verification pipeline has three layers:

1. **Attestation verification** (critical path): HMAC-SHA256 checks at approximately 1–5 microseconds per transaction, replacing traditional signature verification.
2. **Asynchronous STARK proving** (non-blocking): Authorization proofs for all transactions in a block are tiled into a single Stwo Circle STARK proof — 13 Poseidon2 permutations × N transactions, one prove, one verify.
3. **O(1) block verification**: Regardless of how many transactions the block contains, the verifier only needs to check a single STARK proof. Verification cost is completely decoupled from transaction count.

The STARK proof is based on Blake2s hashing and the FRI protocol, with no elliptic curve pairings. Verification itself is post-quantum secure. No trusted setup is required — a fundamental distinction from Groth16.

At the consensus level, Tendermint-style consensus votes are transmitted as P2P messages, consuming no on-chain throughput — all bandwidth is reserved for user transactions. Combined with HKDF context-isolated zero-coordination state sharding, throughput scales linearly with the number of shards in theory.

## IV. MEV: From Profit Redistribution to Ordering Power Elimination

MEV (Maximal Extractable Value) is another structural problem in blockchain: the block producer can extract value from users by reordering, inserting, or censoring transactions. On Ethereum alone, cumulative extracted MEV exceeded $680 million by mid-2023.

The limitations of existing approaches:

- **Commit-reveal**: Hides transaction contents, but does not authenticate who is eligible to create admissible commitments, and provides no transferable proof of omission.
- **Threshold encryption** (Shutter): Requires a decryption committee, introducing additional trust assumptions and latency.
- **Fair ordering services** (Chainlink FSS): Outsources ordering to an oracle network — shifting rather than eliminating the trust assumption.
- **PBS** (Flashbots MEV-Boost): Transfers MEV profit from miners to builders. This is redistribution, not elimination.

In our paper MEV-ACE, we propose an identity-authenticated fair ordering protocol that leverages the ACE-GF identity system to address MEV. Three layers of mechanism:

**Layer 1: Identity-authenticated admission control.** Every commitment must be signed by a registered identity backed by a staking bond, with a per-identity quota per slot. Commitment stuffing (the producer fabricating many low-cost commitments to dilute the ordering pool) goes from near-zero cost to requiring real locked capital.

**Layer 2: VDF-delayed random ordering.** After the admissible set is locked, a Verifiable Delay Function (VDF) computes the ordering seed. The producer cannot predict final transaction positions during the commitment phase, eliminating the information advantage needed for front-running and sandwich attacks.

**Layer 3: Receipt-backed accountable inclusion.** Both the commit and open phases require receipts from 2f+1 validators. These receipts serve as transferable omission proofs — any participant can prove to the entire network that "this transaction should have been included but was dropped by the producer." The producer cannot silently censor transactions.

Formal proofs show that, under standard cryptographic assumptions and correctly calibrated economic parameters, MEV-ACE simultaneously satisfies three security properties: order-unpredictability, commitment authenticity, and accountable inclusion. When the producer's and users' bonds exceed the maximum single-slot gain from violation, honest execution is the producer's best response.

MEV-ACE targets proposer-controlled ordering MEV (front-running, sandwich attacks, censorship), which constitutes the vast majority of MEV losses in historical on-chain data. Information-based MEV — such as cross-domain arbitrage and oracle price retroaction — falls outside the protocol's scope, but its share is comparatively limited and does not rely on producer privilege, making it closer to normal market efficiency than structural exploitation of users.

The entire protocol completes within a single slot, requires no threshold decryption committee, and is natively compatible with post-quantum signatures (ML-DSA-44).

## V. Defining Generations: What Constitutes "Third Generation"

There is no industry-consensus standard for "which generation." But if we use foundational paradigms — identity model, verification model, security model — as the basis for classification:

| Dimension | Gen 1 (Bitcoin) | Gen 2 (Ethereum) | Gen 3 (ACE Chain) |
|-----------|----------------|-------------------|-------------------|
| Identity model | Public key = address | Public key = address | IdCom (identity commitment), keys replaceable |
| Verification model | Per-tx signatures O(N) | Per-tx signatures O(N) | HMAC attestation + STARK O(1) |
| Security model | ECDSA (quantum-vulnerable) | ECDSA (quantum-vulnerable) | ML-DSA + STARK (natively post-quantum) |
| Programmability | None | Smart contracts (single VM) | Multi-VM unified execution |
| PQC performance cost | N/A | Signature bloat collapses throughput | Zero performance loss (classical and PQC at equal performance) |
| Account recovery | Private key / seed phrase | Private key / seed phrase | Password + human-readable identifier |
| MEV protection | None | PBS (redistribution) | Identity-authenticated ordering (elimination) |

The first generation solved trustless value transfer. The second generation solved programmable application ecosystems. The third generation solves the structural defects caused by the architectural assumptions of the first two — security, performance, user experience, ecosystem fragmentation, MEV. The common root cause of these problems is the binding of public keys to identity, and ACE Chain dismantles this assumption at the protocol layer.

This is not incremental improvement. The paradigm has changed.

## VI. Honest Boundaries

Finally, a clear statement of what has not yet been achieved:

- **Sharding performance is a theoretical model.** The zero-coordination state sharding architecture has been implemented, but linear scaling in a distributed multi-node environment has not yet been validated at production scale.
- **MEV-ACE has not yet been integrated into mainnet.** The paper has completed formal analysis; protocol implementation and parameter calibration are next steps.
- **The tension between privacy and compliance.** IdCom hides identity material, but transactions from the same IdCom can be correlated. A "auditable by regulators, private to the public" selective disclosure scheme is still in design.
- **Information-based MEV is outside MEV-ACE's scope.** Cross-domain arbitrage, oracle retroaction, and other MEV based on public information require orthogonal mechanisms such as batch auctions or delayed disclosure.

Building infrastructure for the future requires equal honesty about what has been accomplished and what remains to be done.

## References

- **ACE Chain Whitepaper**: [acechain.io/ACE_Chain_Whitepaper.pdf](https://acechain.io/ACE_Chain_Whitepaper.pdf)
- **ACE-GF**: Atomic Cryptographic Entity Generative Framework — [arXiv:2511.20505](https://arxiv.org/abs/2511.20505)
- **ZK-ACE**: Zero-Knowledge Authorization for Cryptographic Entities — [arXiv:2603.07974](https://arxiv.org/abs/2603.07974)
- **AR-ACE**: Attestation Runtime for ACE Chain — [arXiv:2603.07982](https://arxiv.org/abs/2603.07982)
- **n-VM**: Multi-VM Unified Execution Architecture — [arXiv:2603.23670](https://arxiv.org/abs/2603.23670)
- **CT-DAP**: Context-Isolated Trust Domain Attestation Protocol — [arXiv:2603.07933](https://arxiv.org/abs/2603.07933)
- **VA-DAR**: Vendor-Agnostic Deterministic Artifact Resolution — [arXiv:2603.02690](https://arxiv.org/abs/2603.02690)
- **HFI Pay**: Human-Friendly Identifier Payment Protocol — [arXiv:2603.26970](https://arxiv.org/abs/2603.26970)
- **AESP**: ACE Economic Security Protocol — [arXiv:2603.00318](https://arxiv.org/abs/2603.00318)
- **ACE Runtime**: ZKP-Native Blockchain Runtime — [arXiv:2603.10242](https://arxiv.org/abs/2603.10242)
- **MEV-ACE**: Identity-Authenticated Fair Ordering (forthcoming)
