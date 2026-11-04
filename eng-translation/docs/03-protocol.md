# 3. Protocol summary

A condensed summary of the rules implemented in the core. The full text and rationale — the specification in the main repository: `docs/KNOWLEDGE.md` (v9.16; title page, version history, and table of contents; the content is in `docs/KNOWLEDGE-1..5.md`), sections 4–13, 17 (parts 2–5); parameters — section 17 (part 5) and the genesis proto.

## Transactions and the registry

- Core types: `register`, `vouch`, `mint`, `transfer`, `migrate`, `evidence`, `guardian_offer`, `update_guardians`; modular: `pool_*`, `redeem`/`receipt`, `contribution_*`.
- `update_guardians` — re-election of personal guardians (≤ 3) with their consent (a signature over the hash of the change, in person — like approving a migration); an activation window of 48 h, PIN-only cancellation; a limit of 150 effective changes per wallet (spec, section 11).
- Every wallet has **its own chain**: `{wallet, seq (+1), prev, type, payload, timestamp_claim, signature}`. Double spending = two signed continuations of the same link — trivially provable.
- The balance = the sum of the valid history; every node recomputes it itself.
- A claimed timestamp is admitted when it differs from the node's clock by no more than ±30 min (`timestamp_tolerance_seconds`) — **C9, the network's admission layer, not a validation rule** (v9.13): checked at tx admission/propagation, the core and replay do not apply it; in sandbox/test networks the tolerance is controllable (the check is not applied).

## Time

- **Validation time — txNow = the transaction's own `timestamp_claim` (v9.13, the time-fix stage).** All time-dependent rules (M2, M4, M7, T3, V2, recurring burn, evidence deadlines, `ActivatedPeriod`, modules) are checked from the moment the tx was signed, not from the moment of publication: one tx — one deterministic verdict; the same journal yields the same `state_root` on any node and in replay (live = replay by construction; before v9.13 live validation read the node's clock at apply time — near a period boundary the verdicts diverged; a finding of the 12-testnet run).
- **Settlement — monotonic from max(txNow) over the journal of accepted txs (v9.13):** demurrage, quota burns, period/epoch boundaries, and the deadlines of delayed operations settle from the claim times of accepted transactions; settlement never rolls back, is idempotent, and a recomputation of the journal after a restart reproduces the same state.
- **Issuance periods — arithmetic from `genesis_time`:** `period = (t − genesis_time) / period_seconds` (genesis field 61; 0 — the 30-day convention); the period length scales the challenge slot and the period boundaries identically for all nodes (the challenge window ±48 h — field 51 `mint_challenge_window_hours`).
- The network clock (NetClock) — from finalized checkpoints, not from the median of timestamps (No. 39: anchored to genesis time); the validation of time-dependent rules is from txNow (above); the node's clock participates only in C9 admission.

## Consensus

- Conflicts are resolved by sampled voting, **Snowball** (Avalanche-like): k=10, α=0.7, β=4; the weights of the polled wallets — by age and attestation. The voting reply is signed (wallet + ECDSA).
- **Checkpoints:** an epoch of 24 h; a threshold signature N=14 of M=21 epoch validators; the hash — the root of the sparse Merkle tree of the state (balances + seq); `checkpoint_hash` excludes the signatures (determinism). Modes: `decentralized`, `members_quorum` (small networks, up to ~100 members; a checkpoint of O(N)), and the transitional `seed` mode of the seed circle.
- `seed_sunset`: the seed circle loses its special rights at 100 `active` wallets **or** 24 epochs.
- Decision No. 42 (stage 9, approved (A)+(D)): a queue of checkpoint builders from the first K by composition, with rotation; a stuck (empty) epoch is skipped — the definition of an "empty epoch" is fixed in spec v9.6.
- **Journal reconciliation:** `SyncJournalFrom` — a positional anchor (No. 41): the position in the peer's journal numbering + the txid at the boundary, per-peer continuation, realignment on divergence. Restart from the DB: `ImportJournal` / the `-import-db` flag — importing txs with clocks pinned to `timestamp_claim` (No. 40: `cons.Replay` pinned only NetTime → demurrage from the DB diverged from a cold reconciliation; regression tests lock this in).

## Entry, identity, anti-farming

- `register`: hardware attestation + memory-hard PoW. Attestation (v9.12, stage 14): **leaf binding** — the SPKI of the leaf certificate must equal the registering `attestedPublicKey`, a mismatch is a reject `E_ATTEST_KEY_MISMATCH` (reusing someone else's chain is impossible); **both Google roots** are embedded (the active RSA one and the ECDSA P-384 Remote Key Provisioning one); synthetic roots — only with `sandbox = true` (a non-sandbox network with a synthetic root does not start); the leaf's validity — at the `timestamp_claim` of registration (replay = live). Attestation confirms a real device with protected hardware, but not the uniqueness of the device — several attested keys on one device are possible. One fingerprint — one address in the network: `address = SHA-256(fingerprint ‖ network_id)` (mini-stage 12.2; in force in spec v9.16). The seed circle and providers are defined in the manifest/genesis by public keys (fingerprints); duplicates and empty keys are rejected at network assembly; wallet ownership is proven by a signature.
- Activation: K=3 `vouch` from different `active` wallets (vouching member age ≥3 months, slots 3/5/10, the seed circle — 10 in the first year). The reward to vouchers R=10% of the vouched-for member's first 12 issuances (from their issuance, not from the emission).
- Issuance: periodic and equal (1000/month by default), only `active` wallets, only completed periods (the first issuance — in the period after activation), the challenge in the slot of the anchor period: `H(address ‖ period)`; with v9.14 (mint-fix) the anchor is the current issuance period (`mint_slot_mode = 1`, protocolVersion 3), the slot is publicly computable in advance; including hash(checkpoint of the previous epoch) in the formula — an open question, PENDING (not applied in the code). The accrual window is 6 months.
- The `transfer` moratorium: 7–30 days after `register`.
- Equivocation → `evidence` from any node → the `flagged` mark, issuance suspended for 3 months (a repeat — forever), vouching from `flagged` is forbidden. There are no monetary penalties or deposits.
- Revoked attestation certificates: a revocation mechanism (No. 43) is deferred for v1; currently only the patch level is checked (not expired; a "future patch date" — a minor fix in stage 10).

## Networks and recurring (stage 12, spec v9.10–9.11)

- Network identity: `network_id = H(manifest ‖ genesis)` — self-certifying (the manifest hash is computed over the manifest with an empty `network_id`); `slug` — a human-readable label, not the identity; the global network — the reserved constant `network_id = "global"`.
- Recurring (profile No. 8 "Subscription resource"): every unit carries the period mark `period`; transferring a unit of period P is rejected after the boundary of period P+1 (the `E_BURNED` error); there is no balance reset — the history stays verifiable. Allowed only with `unit = quota(resource_id)`, a non-global `network_id`, and `transfer_mode ∈ {restricted, peer-redeem}` — combinations are rejected by genesis validation at assembly.
- `peer-redeem`: transfers member → member and member → provider (for the flow of recurring quotas); the provider is defined by a fingerprint, no separate address is specified in the parameters.
- Configurator (an artifact of the builder, not of the protocol): applicationId `io.equality.<sanitized-slug>.x<hash8>`, global — `io.equality.global.x<hash8>`, label `Eq-<hash8>` (hash8 — the first 8 hex of SHA-256(network_id)); APK names `equality-{global|slug}-{std|rec}-v{version}.apk`.

## Economics

- The transfer fee is **burned**: `fee = max(fee_min, rate × amount)`, `fee_min = 0.01` (in the core code — 1 whole unit; the discrepancy is recorded by the z-ai audit, QUESTIONS), rate ∈ [0.5; 2]% adaptively (a deterministic recomputation at the epoch boundary from checkpoint data; all nodes get one value). The target supply growth — 15% over a window of K=6 epochs, a step of 0.05 pp (v9.9; previously 6%).
- Demurrage: 1% above the floor of 3000, a deterministic recomputation by the number of elapsed epochs; the charged amount is burned. Mandatory for profiles with transferable value.
- Delayed transfers: threshold P=3000, maximum P_max=30000, a fixed delay D=48 h, cancellation by the owner's signature during the delay.

## Network

- Bootstrap cascade: the peer cache (SQLite) → signed seed lists (Telegram channels, best-effort) → the list embedded in the APK → NSD/mDNS → a Bluetooth scan → QR → manual entry; then DHT + gossip. Eclipse protection: mixing of sources, peer disjointness by /24 and /64 subnets (No. 35; ASN — PENDING), corroboration from N disjoint peers.
- `network_id` distinguishes instances; in `invite_only` — the addresses of the first nodes and the network key are in the invitation.
- Bluetooth: only handing a signed tx to the recipient; it becomes a payment after publication and confirmation by the network.
- Local fleets (localhost, one /24): run with `-link-groups=false`, otherwise the linked-subnet classifier will consider all nodes one group and block corroboration. In production the default is on.

## Governance

- There are no governance roles. `params_mutable=false` by default; with `true` — only within the `mutable_bounds` from genesis, a threshold of ≥2/3 `active`, a delay of ≥30 days, non-retroactively. Protective parameters (K, D=48 h, `transfer_mode`, the emission, `device_policy`, `signing_auth`, `checkpoint_mode`, `params_mutable`) never change.
- The right to fork: any member may continue the network under the same rules with the same history (a fork package from a checkpoint).
- `protocolVersion` in every transaction and checkpoint (currently **3**, raised in v9.14; release v3.0.0); APK releases and seed lists — an N-of-M multisignature.

→ [Development process](04-development.md)
