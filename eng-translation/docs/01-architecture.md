# 1. Architecture

Full source: the specification in the main repository — `docs/KNOWLEDGE.md` (table of contents; the content is in `docs/KNOWLEDGE-1..5.md`), sections 3, 9, 12 (parts 1 and 4). This file is a practical summary of the current implementation.

## Application layers

```
Flutter/Dart (UI)
   │  platform channels
Kotlin: NodeService (foreground service), Keystore, biometrics, PIN
   │  AAR calls (gomobile bind)
Go core: core · cons · net · mobile · sim · cmd
```

- **Flutter/Dart** — the entire member-facing interface: onboarding, wallet, registry, vouching, migration, guardian re-election, decoy mode.
- **Kotlin** — the platform layer: Android Keystore (wallet keys), biometrics, PIN (PinManager), foreground service (NodeService). Keys never leave the Keystore: the core verifies signatures but never holds them.
- **Go core** — the network stack (go-libp2p: QUIC/TCP, Noise, Kademlia DHT, GossipSub, hole punching, circuit relay), transaction and attestation-chain verification, PoW, consensus, checkpoints, storage.
- **The binding:** `gomobile bind` → AAR; message exchange via protobuf; measured bridge overhead — 5–22 µs per call.

## Threading (critical for Android)

- **All cgo is off-main.** A blocking `startNode` on the main thread caused an ANR (bootstrap up to ~11 s) — a release blocker, fixed: `onStartCommand` moves to the background, readiness arrives via a callback. All bridge calls are checked for UI blocking.
- The Kotlin bridge is thread-safe: `mainHandler` / `executor` / `ConcurrentHashMap` (stage 9 audit — no findings).
- The UI's read layer into the registry is `net/query.go` (getters under `sync.RWMutex`; `Apply` holds the Lock). Direct reads of the core's internal structures from the UI are forbidden (data race — stage 7, an authorized exception).
- The UI does **not duplicate** protocol formulas: the issuance screen takes `claim_period`/`claim_amount` and the challenge window from the node via the bridge (`MintWindowJSON` — `open`, `window_opens_at/closes_at`, `submit_at`). The slot anchor is the current issuance period (v9.14, mint-fix; genesis field 63 `mint_slot_mode` = 1), window verification is by the transaction's `timestamp_claim` (v9.13); the UI clamps the claimed period to the accrual window (D2, AUDIT-MINT-FIX-02), otherwise a wallet after ≥ accrual_window missed challenges would get a "poisoned" claim. The exact logic is in Go (`CheckMintChallengeWindow`/`QueryMintWindowBounds`).

## Storage

- **SQLite** — the ncruces/go-sqlite3 driver (WASM SQLite via wazero; stage 16, No. 38): transaction chains, the "fingerprint → address" index, the fingerprint blacklist, checkpoint headers, the state snapshot. The driver has no CGo and no raw legacy syscalls — the SIGSYS problem of x86_64 emulators (finding No. 38, the former modernc.org/sqlite driver) was eliminated by replacing the driver: SQLite works both on a live phone and on emulators. Running without a DB path means no persistence (headless utilities, the light node).
- **allowBackup=false** — DB backup is disabled (protection against accidental equivocation on a `seq` rollback).
- Snapshots are state commitments: the root of the sparse Merkle tree of the state, in the checkpoint.
- A light node stores its own chain, its incoming transfers, and checkpoint headers; it verifies balances with Merkle proofs.

## Node tiers

| Tier | Behavior | Mode |
|------|----------|------|
| Full | Stores and verifies the entire registry, relays, votes | Charging + Wi-Fi, foreground service |
| Light | Own data + checkpoint headers, Merkle proofs | Normal phone mode |

Behind mobile NAT (CGNAT) there are no inbound connections — hole punching and relaying via volunteer full nodes (libp2p circuit relay). These are not project servers, but it is not "pure P2P" either — stated honestly.

## Cryptography

- **Wallet signatures:** ECDSA P-256 (Keystore reliably supports it; Ed25519 — not everywhere). Verified on target devices.
- **Canonicality:** strict DER + low-s in **all** signing paths (stage 9: ECDSA malleability — the TxID includes the signature — produced false conflicts; fixed, regression tests exist).
- **Attestation:** KeyMint/Keymaster parsing via `encoding/asn1`; each node verifies the chain locally against Google root certificates embedded in the APK. Checked: hardware level, `verifiedBootState=Verified`, `deviceLocked`, patch level. ✅ v9.12 (stage 14): **both Google roots** are embedded (the active RSA one and the ECDSA P-384 Remote Key Provisioning one) — a chain is valid if it reaches either of them; **leaf binding** — the SPKI of the leaf certificate must equal the registering `attestedPublicKey`, a mismatch is a reject `E_ATTEST_KEY_MISMATCH` (checked by every node at apply, deterministic from the tx data — a replay yields the same verdict); synthetic roots — only with `sandbox = true` (a non-sandbox genesis with a synthetic root does not start; a zero PoW target outside the sandbox is an invalid genesis); the leaf's validity is checked at the `timestamp_claim` of registration (replay = live). Play Integrity and Google servers are not used.
- **PoW:** light SHA-256 (target 2^40 ≈ 2^24 hashing iterations, ~1–4 s) on `transfer`; memory-hard Argon2id 32 MiB (target 2^56 ≈ 256 iterations, ~10–20 s) on `register` and issuance (audit amendment A4: the target ≠ the number of iterations).
- `walletFingerprint = SHA256(attestedPublicKey ‖ protocolId)`; the address = `SHA-256(fingerprint ‖ network_id)` — a single derivation point from the final `network_id` (mini-stage 12.2). The manifest and genesis store no addresses: the seed circle and providers are defined by public keys (fingerprints); duplicates and empty keys are rejected at network assembly.

## UI security (stage 9 rules, spec 8.2)

- Sign-in — fingerprint or PIN.
- All destructive operations (a transfer, including the Bluetooth path; issuance; migration propose; guardian re-election (publish/cancel); PIN change/removal) — **explicit PIN entry only**. Biometrics after the PIN — only for "key unlock".
- Every PIN prompt is an entry point into decoy mode (ordinary or duress PIN); biometric confirmation of transfers is forbidden outright.
- While the duress PIN is active, fingerprint sign-in is disabled.

→ [Build and tests](02-build-and-test.md)
