# 6. Network deployment

The launch sequence of a new protocol network: configuration → genesis package → nodes → seed circle → admitting members → oversight. All commands are run from the root of the main repository; the utilities are `cmd/tnp-testnet` (the configurator and the drivers) and `cmd/tnp-node` (a headless node). References to spec sections — `docs/KNOWLEDGE-*.md` of the main repository.

## 0. Prerequisites

- The Go toolchain and building the core — see [Build and tests](02-build-and-test.md).
- Building the utilities:

```sh
go build -o bin/tnp-testnet ./cmd/tnp-testnet
go build -o bin/tnp-node    ./cmd/tnp-node
```

- For network members — the app (APK) built for the specific network: `equality-{global|slug}-{std|rec}-v{version}.apk`, applicationId `io.equality.<sanitized-slug>.x<hash8>`, label `Eq-<hash8>` (section 2).
- The parameter decisions (the profile template, the seed circle, the thresholds) are made per the specification; the configurator merely assembles a package from a valid combination — genesis validation rejects incompatible combinations.

## 1. Network configuration and the genesis package

The configurator — the `configure` mode of the `tnp-testnet` utility (section 5 of the specification):

```sh
# list the profile templates (1..8; No. 8 subscriber-resource — recurring)
./bin/tnp-testnet configure -list

# a network from a profile template (-slug is required; -genesis-time — unix seconds,
# 0 = now; the full flag list is printed by -list/a flag without a value)
./bin/tnp-testnet configure -template 1 \
  -slug my-network \
  -network-id my-network \
  -seeds alice,bob,carol \
  -genesis-time 1762257600 \
  -display-name "My network" \
  -out net/
```

Parameters: `-slug` (required for a non-global network), `-network-id` (a label for the invitation-key determinism; defaults to the slug), `-seeds` (seed circle labels), `-genesis-time` (unix seconds, 0 = now), `-issuance-amount` (the issuance per period), `-vouch-k` (vouches for entry), `-epoch-seconds` (the period; production — 24 h, tests are accelerated), `-resource-id` (for the recurring profile No. 8), `-display-name`, `-global` (the global network), `-recurring`, `-providers`/`-provider-seed` (provider fingerprints), `-period-seconds`, `-transfer-moratorium-days`, `-mint-challenge-window-hours`, `-migrate-delay-days`, `-logo`, `-app-version`, `-invite-key-hex` (the invite_only invitation key), `-signing-dir` (the directory of the production APK signing keys).

The output in `-out` (z-ai audit: described per the actual configure.go): `genesis.bin` (the package: the manifest + the genesis record; no seed list inside), `manifest.json`, `invite.txt` (plain text: the genesis package, manifest_hash, invite_key; the QR/link `tnp://invite?…` is generated only by `generate`), `guarantees.json` (the guarantee level from the manifest, 3.3), `android.json` (applicationId/label/APK name: `Eq-<hash8>`, `io.equality.<slug>.x<hash8>`), the `icon/` directory. The challenge slot table is printed by the `generate` and `slots` modes — not by configure. Network identity: `network_id = H(manifest ‖ genesis)`, the slug, the label `Eq-<hash8>`.

The basic variant without a template — the `generate` mode: writes `genesis.bin, manifest.json, seedlist.bin, invite.txt` and prints the slot table (it does not create guarantees.json/android.json/icon — that is configure only). Flags: `-network-id`, `-seeds`, `-peer`, `-invite-key-hex`, `-out`, `-genesis-time`, `-epoch-seconds`, `-small-pow`, `-argon2-mib`, `-fresh-window`, `-anchor-epoch-zero`.

Important: the invitation key is stored in the genesis record and is **not published in the manifest** (spec/02 2.1). `genesis.bin` is distributed to the nodes; `manifest.json` may be published.

## 2. Starting the nodes

A node — `tnp-node`. The minimum for a full node:

```sh
./bin/tnp-node -genesis-file net/genesis.bin \
  -db node.db -status-file status.json
```

The node prints its Peer ID and multiaddr addresses — they are given to other nodes via `-peer` (the flag is repeated). Key flags:

| Flag | Purpose |
|------|---------|
| `-genesis-file` | genesis.bin from step 1 (without it — the built-in test genesis) |
| `-peer` | the multiaddr of a peer for bootstrapping (repeated) |
| `-relay` | the multiaddr of a relay node (for nodes behind CGNAT; repeated) |
| `-force-public` | relay-node mode: force public reachability |
| `-no-announce` | do not announce your own addresses (CGNAT emulation) |
| `-level light -own-wallet HEX` | a light node: proof verification over finalized checkpoints for its own wallet only |
| `-wallet LABEL -wallet-domain seed\|wallet` | the node's wallet for signing checkpoints and the vote weight |
| `-db` | SQLite (without the flag — no persistence) |
| `-status-file` | a JSON status (peers, journal_len, state_root, checkpoints) |
| `-link-groups` | peer disjointness by /24, /64 subnets (No. 35): production — true (the default), a localhost fleet — false |
| `-import-db` | restoring the journal from an old SQLite before the network starts |

Seed nodes start first; their addresses are handed to the rest via `-peer`. For networks where some nodes are behind CGNAT, a relay node is raised (`-force-public`), and the rest connect to it via `-relay`.

## 3. The seed circle: issuance and slots

- The seed circle's challenge slots (the windows in which issuance is allowed): `./bin/tnp-testnet slots -genesis net/genesis.bin`.
- Issuance of the equal periodic entitlement: `./bin/tnp-testnet mint -genesis net/genesis.bin -seed alice -period 1 -peer <address>...` — the seed wallet publishes the MINT of a completed period.
- Equivocation debugging: `mint -save file -no-submit` (the saved second branch) → `publish` → `evidence -txa -txb`.

All tx modes accept `-peer` (the nodes' peers), `-wait` (waiting after publication), `-db`, `-node-seed`.

## 4. Admitting members

Entry into a closed network — invite_only with vouching (invite_only_vouch):

1. The member receives the invitation from `invite.txt` (a QR/link) and installs this network's APK.
2. REGISTER: the wallet registers with hardware attestation and hard PoW (in the sandbox — synthetic attestation). Attestation validation (v9.12, stage 14) by every node at apply: the SPKI of the leaf certificate must match the registering key (`E_ATTEST_KEY_MISMATCH`), the chain — to one of the two Google roots, synthetic roots are accepted only with `sandbox = true` (a non-sandbox network with a synthetic root does not start — and neither does the headless utility: the test-only flag `AllowSyntheticRoots` exists for tests, the production path does not set it); the leaf's validity is checked at the `timestamp_claim` of registration: `./bin/tnp-testnet register -genesis net/genesis.bin -wallet dave -peer ...` (the flags `-guardians`, `-android-id`).
3. VOUCH: the seed circle members vouch for the candidate — `vouch_k` vouches are needed: `./bin/tnp-testnet vouch -genesis net/genesis.bin -seed alice -candidate <hex> -peer ...`.

After registration the member is an equal wallet; their node may sign checkpoints (`-wallet` with `-wallet-domain wallet`).

## 5. Oversight of a running network

- A registry snapshot: `./bin/tnp-testnet state -genesis net/genesis.bin -peer ...`.
- Export for analysis (turnover, supply, the Gini inequality — from the public registry): `./bin/tnp-testnet export -genesis net/genesis.bin -format json`.
- Node monitoring: `-status-file` (peers, journal_len, conflicts, state_root, checkpoints).
- Finality — Snowball (k=10, α=0.7, β=4) and N-of-M checkpoints; seed_sunset extinguishes the seed circle at 100 active members OR 24 epochs — after that only members sign checkpoints.

## Pre-launch check: a fleet on one host

A full run of the configuration without hardware — the `fleet` mode (48 persistent headless nodes, <1 GB):

```sh
./bin/tnp-testnet fleet -genesis net/genesis.bin -out fleet/ -nodes 48 \
  -seeds alice,bob,carol -base-port 4100 -bin ./bin/tnp-node
```

On localhost: `-link-groups=false` for all nodes and drivers (all peers in one /24), `epoch_seconds` lowered. Runbooks and run precedents — `reports/stage8/TESTNET-RUNBOOK.md` and the stage 8/8.1 reports of the main repository.

## Limitations

- The seed circle and wallet keys in `tnp-testnet` are deterministic from labels — this is a test network; **production keys are generated on devices** (Keystore/KeyMint), the private key never leaves the device.
- A sandbox manifest → synthetic attestation roots (demo/sandbox only); a production manifest → both Google roots — RSA and ECDSA P-384 Remote Key Provisioning (A2, v9.12). A non-sandbox genesis with a synthetic root or a zero PoW target does not start (genesis validation, v9.12).
- Checks on live phones (Bluetooth payment/vouch, attestation on real hardware, a seed circle of several people, etc.) — the 8-live block; without it pilots do not launch.

→ [Development process](04-development.md)
