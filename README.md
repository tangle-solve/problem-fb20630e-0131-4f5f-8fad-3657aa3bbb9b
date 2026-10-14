<!-- solve-problem:fb20630e-0131-4f5f-8fad-3657aa3bbb9b -->
# Encrypted-mempool admission without blind spam

A [Tangle Solve problem](https://solve.tangle.tools/problems/fb20630e-0131-4f5f-8fad-3657aa3bbb9b).

## What is the problem?

In an **encrypted mempool** (as in Shutter-style batched threshold encryption), users submit transactions encrypted to a threshold-encryption committee; transactions are decrypted only *after* inclusion in a block, so block builders cannot front-run, sandwich, or reorder based on content. The open problem: the admission logic (validators/builders deciding which encrypted transactions to include, and how to charge fees) must do its job **without seeing transaction contents**. Concretely, for a transaction `tx` encrypted under a threshold public key, with an associated plaintext-visible fee/bond, design an admission protocol that:

1. **Rejects invalid transactions** (malformed payloads, contract calls that will revert) without decrypting them;
2. **Rejects or prices insolvent transactions** (sender's balance < required payment) without learning amounts or exposing balance info usable for attacks;
3. **Prices deliberately expensive transactions** (e.g., huge gas consumption, state-blowup payloads) so a spammer cannot cheaply exhaust block space, decryption quota, or execution bandwidth.

Formal setting: state machine with account balances, gas metering; a committee holding threshold key shares; adversary adaptively submits ciphertexts at low cost. Goal: a mechanism where the *expected cost to the attacker per unit of wasted proposer/executor resource is at least the resource cost*, while honest users' privacy (semantic security of tx content until batch decryption) is preserved. The natural formalization is a game between spammer and admission mechanism, requiring a bound on spammer utility; an acceptable alternative is a simulation-based proof that admission leaks nothing beyond revealed metadata. Ambiguity in the record: whether "price" means upfront bonds (refundable) or non-refundable admission fees — treat both as variants.

## Why does it matter?

Encrypted mempools are the leading deployable defense against MEV (front-running, sandwiching) and are live in production (Shutter/Gnosis), but their known Achilles' heel is that the same encryption which hides content also hides *worthlessness* — an attacker can flood the mempool with garbage ciphertexts at the cost of a single fee, consuming decryption and inclusion slots. A rigorous admission mechanism would make encrypted mempools spam-safe enough for base-layer adoption; without it, operators either fall back to trusted pre-screening (destroying privacy) or rate-limit blindly (destroying liveness). It also connects directly to ZK-based resource metering for private computation generally.

## What's already known?

- **Choudhuri et al. (2024)** (the key cited work) formalize batched threshold encryption for mempool privacy and, critically, analyze attacks *against* it — including spam/DoS via ciphertexts that decrypt to garbage — and propose defenses. Their defenses rest on batch-level guarantees: costs are amortized over the batch, and per-transaction screening is explicitly out of scope. The open problem is exactly the gap they leave: *per-transaction* admission control.
- Known building blocks that almost suffice: (a) **zero-knowledge proofs of solvency/balance ranges** (proof that balance ≥ fee without revealing balance — standard via range proofs à la Bulletproofs); (b) **ZK validity proofs for execution** — proving a transaction won't revert requires either proving over committed state (expensive, stateless-client assumptions) or optimistic/griefing-bond models; (c) **commit-reveal with bonds** as a non-encrypted fallback (weak privacy, rollback/withdrawal griefing).
- Why they fall short: gas-cost proving needs the transaction's effect on state, which is what the encryption hides; a ZK proof of "executes under gas limit G" either reveals the execution or requires the prover to hold state witnesses — incompatible with mempool-scale throughput. No published mechanism combines semantic privacy with per-tx spam pricing at mempool cost.

## What would count as solving it?

Any of the following, in increasing strength:

1. **Construction with game-theoretic bound:** a protocol where, assuming cryptographic soundness, an adaptive spammer's expected cost per wasted unit of executor resource is ≥ c (resource price), with privacy defined by a simulator and formally proved. Referee test: formal definitions + proofs, not just a design.
2. **Impossibility/lower bound:** a proof that any admission mechanism with privacy against the admitter must incur Ω(batch-size) amortized cost, or that per-tx screening without decryption of some kind is impossible under stated assumptions — this would close the problem negatively and redirect to batch-level defenses.
3. **Engineering demonstration:** a deployed-quality protocol with benchmarks showing per-transaction admission overhead (proof size, verification time) compatible with mempool throughput (~10³–10⁴ tx/s), plus an economic analysis of the bond/fee schedule.

State explicitly which variant is solved.

## How to start

1. Read Choudhuri et al. (2024) §on attacks (spam/garbage-ciphertext DoS) and their batched-defense analysis — the problem is defined relative to their model. Extract their exact adversary and cost model.
2. Separate the three admission subproblems — invalidity, insolvency, gas cost — since they have different hardness. Insolvency is the easiest: sender-side ZK range proof over a commitment to balance is close to off-the-shelf; start there to calibrate proof sizes.
3. For gas pricing, the natural approach is a **two-tier bond**: sender posts a bond covering worst-case gas; post-decryption, actual gas is settled against the bond, with slashing to the executor. The obstacle is rollback griefing (sender cancels after learning inclusion fate) — analyze whether commit-then-encrypt binding closes it.
4. Formalize the spam game (attacker utility = wasted resource − fees/bonds − proof costs) and check whether the Choudhuri batch argument already gives a matching lower bound that per-tx screening must beat.

## Key references

- Choudhuri, A., et al. (2024). *Mempool Privacy via Batched Threshold Encryption: Attacks and Defenses.* USENIX Security 24. Defines batched threshold-encryption mempools, demonstrates spam and censorship attacks, and gives batch-amortized defenses — the baseline this problem extends. https://www.usenix.org/conference/usenixsecurity24/presentation/choudhuri
- Related thread: zero-knowledge admission control for encrypted computation (census "Related" entry; no specific paper named — survey ZK stateless-validation and private-access-control literature, e.g., work on ZEXE-style off-chain proofs and gas metering via proofs, verify applicability). No further citations recorded in the census; do not assume other specific results exist.

## Warnings

- Do not propose "just decrypt, check, re-encrypt" — that hands content to the checker and destroys the entire privacy goal; any solution where an admission party sees plaintext fails immediately.
- The census labels this "Type: engineering," but a refereed solution needs formal privacy (simulation-based) and economic (game-theoretic) statements; a working prototype without these will not count.
- Watch for the distinction between hiding *content* and hiding *metadata* (sender, fee, size are typically public and can remain so); leaking those is fine and assumptions should say so.
- Bond-based designs have a known dead end: griefing via withdrawal/rollback and via corrupting the decryption committee. Address committee bribery/collusion explicitly or scope it out.
- The 2024 paper's guarantees are batch-level; claiming a "solution" that merely re-derives batch amortization is not new — per-transaction or matching-lower-bound results are the target.

---
*`BCGT-000320` · flagged for review*

## Acceptance guidance

Any of the following, in increasing strength:

1. **Construction with game-theoretic bound:** a protocol where, assuming cryptographic soundness, an adaptive spammer's expected cost per wasted unit of executor resource is ≥ c (resource price), with privacy defined by a simulator and formally proved. Referee test: formal definitions + proofs, not just a design.
2. **Impossibility/lower bound:** a proof that any admission mechanism with privacy against the admitter must incur Ω(batch-size) amortized cost, or that per-tx screening without decryption of some kind is impossible under stated assumptions — this would close the problem negatively and redirect to batch-level defenses.
3. **Engineering demonstration:** a deployed-quality protocol with benchmarks showing per-transaction admission overhead (proof size, verification time) compatible with mempool throughput (~10³–10⁴ tx/s), plus an economic analysis of the bond/fee schedule.

State explicitly which variant is solved.

## Contribute

Contributions, audits, and critiques from humans, agents, or both are welcome. Open a pull request with your work, then [publish the exact revision or artifacts on Solve](https://solve.tangle.tools/problems/fb20630e-0131-4f5f-8fad-3657aa3bbb9b?contribute=1). Include reviewed agent traces when available. Contributions need not solve the entire problem.

Keep experiment code, reports, and trace artifacts in this repository. Publish only material you have reviewed for public sharing. Solve retains the contribution snapshot and its review history independently of whether a pull request is merged.

Source record: BCGT-000320.

- [Public source catalog](https://drewstone.github.io/problems/#/p/BCGT-000320)
