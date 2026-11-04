# Network merging: why this feature will not be built

**Status: closed by the owner's decision of November 2, 2026.** This is a permanent platform rule, not a deferred roadmap item.

## The essence

The Equality Project platform deliberately **does not implement merging (merge) of network instances or any cross-network bridges**. The merge request ("two divisions of one operator deployed independent networks and decided to combine them") was worked through to the end: a full technical design, then an independent external audit — and rejected based on the audit results. Below are the objective reasons, so that a reader planning a fork "with merge support" can assess the size of the problem before writing code.

## The design work and its verification

- Design: `docs/network-merge-design.md` (main repository, revision `2a42908`) — a merge as launching a new network C with a merge-genesis: ratification by signal transactions in both networks, a snapshot over finalized checkpoints, a halt rule, identity and state import, 6 open decisions, 10 questions to the audit.
- Audit: an independent external audit of the design — `audit/results/AUDIT-network-merge.md` (performed in a separate branch of the [equality-trust-networks](https://github.com/rumatakira74/equality-trust-networks) repository, deleted after the topic was closed). Verdict: **"Rework"** — 4 CRITICAL, 9 HIGH, 7 MEDIUM findings; of the 10 design questions to the audit, 5 received the answer "problem", 1 — "ambiguous".
- Decision: instead of a rework, the owner closed the topic — the complexity and risk do not justify the rare scenario.

## Why a safe merge does not work out (key audit findings)

1. **Cross-network identity is not provable (CRITICAL, F-01).** The central premise of the design — the deduplication "one fingerprint = one person in network C" — is false: a fingerprint is a hash of a device key (`SHA-256(attestedPublicKey ‖ protocolId)`), not a person's identity. One person with two devices gets two wallets in C; "free" Sybil deduplication is not achieved.
2. **No key re-binding on transfer (CRITICAL, F-02).** Transferring state into C does not define with which key the owner proves the right to the imported wallet; a claim-import-like procedure is required, which the core does not have.
3. **Stopping the source networks is non-atomic (CRITICAL, F-03).** A halt rule between two independent networks admits a double spend ("zombie-A": the balance is spent in A after the snapshot and again in C). The audit demands a two-phase prepare/commit protocol — essentially a second consensus on top of the core.
4. **The terminal checkpoint is outside the validity rules (CRITICAL, F-04).** A merge checkpoint conflicts with the basic consensus invariants (non-retroactivity, finalization rules), that is, merging breaks the core invariants the platform promises to every network.
5. **Merge economics are vulnerable (HIGH).** "Farming for a merge": an attacker grows two networks of their own, combines them, and absorbs an honest network by buying votes on ratification — in a simulated scenario up to 77% of the emission of the combined network goes to the attacker. Kill-switch: a minority of the signers of network B can stop network A. Merge inflation and demurrage settlement in C were recognized as unsolved problems.
6. **Complexity for a rare scenario.** The rework would have required: section 3.4 of the specification (RU+ENG), a status-merge lattice (flagged = OR, age = min, suspension = max), a normative settlement algorithm, attest caps after the merge, a transitional checkpointers' quorum from the union of the committees, rules for delayed transfers and ratification thresholds — while half of the design questions of the audit remained "problem".

## Why isolation is a principle, not a v1 limitation

- The address is bound to the network: `address = SHA-256(fingerprint ‖ network_id)` — cross-network identity transfer is impossible without breaking the address derivation.
- `network_id = H(manifest ‖ genesis)`, the manifest is immutable after launch — "combining A and B in place" is impossible by construction; a merge would always mean launching a new network.
- The platform's philosophy: if the rules stopped fitting — a new network is launched, and members move voluntarily (the right to fork). This is already safe, free, and requires no protocol support today.
- The level of guarantees is computed from the network's manifest; merging networks with different guarantees would undermine the predictability of that level.

## If you need "combined" accounting

It is solved without the protocol: a community launches a shared network with the constructor from the very beginning, or members voluntarily register (`register` + vouching) in the new network and carry over their social status. Balances are not transferable by construction between networks with different rules; the public history of each network is preserved and verified independently.

## Planning a fork with merge support?

First read the design (all the mechanisms have already been invented) and the full audit (why they are not safe in this core). The realistic scope: a two-phase halt consensus, a key re-binding procedure, a state-merge lattice, a new settlement, countermeasures against merge inflation and the kill-switch — with unprovable cross-network identity. The audit and the owner agreed: the cost and risk do not match the value of the scenario.

## English TL;DR

Network merging is **permanently rejected** (owner decision, Nov 2, 2026), not deferred. A full design was produced and then audited by an independent external review (verdict: *rework required* — 4 CRITICAL, 9 HIGH, 7 MEDIUM; 5 of 10 design questions rated as unsolved problems). Key findings: fingerprint is a device-key hash, not a cross-network identity, so merge dedup is unsound; key re-binding is undefined; cross-network halt is non-atomic (double spend); the merge checkpoint violates core consensus invariants; merge farming (up to 77% of joint emission) and kill-switch attacks are economical. Isolation of network instances is a platform principle: addresses derive from `network_id`, manifests are immutable, and communities needing joint accounting simply start a new network and join voluntarily. Forking to add merges means rebuilding consensus on top of an unsound identity premise — see the audit before writing code.

© 2026 Kirill Lapchinsky
