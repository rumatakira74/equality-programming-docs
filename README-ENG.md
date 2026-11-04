# Equality Project — developer documentation

Russian original: [README.md](README.md).

> **Unofficial English translation.** In case of any discrepancy, the Russian original ([README.md](README.md)) prevails. The full English mirror of this repository's documentation lives in [eng-translation/](eng-translation/).

The technical documentation of the **Equality Project** trust networks platform: architecture, build and tests, a protocol summary, the development process.

**DOI:** [10.5281/zenodo.23558842](https://doi.org/10.5281/zenodo.23558842)

> **Source of truth** — the specification in the main repository: [equality-trust-networks](https://github.com/rumatakira74/equality-trust-networks), file `docs/KNOWLEDGE.md` (v9.16; title page, version history, and table of contents) with the content in the parts `docs/KNOWLEDGE-1.md` … `docs/KNOWLEDGE-5.md`. This repository is a summary and a practical guide; in case of a discrepancy, the specification prevails.

© 2026 Kirill Lapchinsky

## Contents

1. [Architecture](eng-translation/docs/01-architecture.md) — the application and core layers, threading, storage
2. [Build and tests](eng-translation/docs/02-build-and-test.md) — the toolchain, AAR/APK, runs, CI
3. [Protocol summary](eng-translation/docs/03-protocol.md) — transactions, consensus, checkpoints, economics
4. [Development process](eng-translation/docs/04-development.md) — stages, branching, checkpoints, reports
5. [Network merging: why it will not be built](eng-translation/docs/05-network-merge-rejected.md) — the owner's decision, the audit results, an answer for those planning a fork
6. [Network deployment](eng-translation/docs/06-network-deployment.md) — the configurator, the genesis package, nodes, the seed circle, admitting members, oversight

## Repository map

| Repository | Purpose |
|------------|---------|
| [equality-trust-networks](https://github.com/rumatakira74/equality-trust-networks) | The main repository: the Go core, the Android app, the specification (`docs/KNOWLEDGE.md` — the table of contents; the content is in `docs/KNOWLEDGE-1..5.md`), the development plan (`docs/development-plan.md`), stage prompts (`prompts/`), reports (`reports/`), the question registry (`QUESTIONS.md`) |
| [equality-user-docs](https://github.com/rumatakira74/equality-user-docs) | The user documentation |
| [equality-programming-docs](https://github.com/rumatakira74/equality-programming-docs) | This repository |
| [equality-anti-farm-simulation](https://github.com/rumatakira74/equality-anti-farm-simulation) | A Python model of the farm economy; the justification of the parameters (K=3, slots, R=10%, a seed circle ≥300). Release v1.0.0 on Zenodo |

## Development status

Stages 0–10 and the mini-stages of the series 11 are complete: specification → the economy model → crypto/core → network → consensus → the Android platform → the Flutter app → a closed test network (emulators) → a 50-node simulation → audit (cryptography, consensus, owner protection) → the platform of profiles and modules (the adaptive fee, the attest verification library, redeem/pool/contribution, the network configurator, the templates of seven profiles, the recovery card, i18n — five languages: ru, en, es, pt, zh); the mini-stages of the series 11: stage11-prep — formalizing decisions No. 44–No. 46 and the emulator run "theft → recovery"; stage11-1-fixes — fixes of the run's findings (low-s Keystore signatures, the challenge window of the UI's claimed period, the StartMigration nonce, the `-migrate-delay-days` flag); stage11-2-guardians — re-election of personal guardians `update_guardians` (a 48 h window, PIN-only cancellation, a limit of 150 effective changes per wallet, the "Guardians" screen and the consent screen, a run demonstrating publish/cancel/effect); stage 12 — the recurring profile No. 8 "Subscription resource" (the period mark, burn at the boundary of P+1 without a balance reset, peer-redeem), network identity `network_id = H(manifest ‖ genesis)` + `slug` + the label `Eq-<hash8>`, genesis validation of combinations, APK names `equality-{global|slug}-{std|rec}-v{version}.apk`; mini-stage 12.1 — applicationId `io.equality.<sanitized-slug>.x<hash8>`, a Material palette; mini-stage 12.2 — a fingerprint manifest (the seed circle and providers = public keys, `address = SHA-256(fingerprint ‖ network_id)`, wire SeedMember=public_key); mini-stage 12.3 — the `x` prefix in the applicationId. Spec v9.16 (protocol_version 3). After stage 12 the following were completed: stage 13 (network merging is not implemented — the owner's decision), stage 14 (attestation A1/A2/A12, v9.12), stage 15 (the txNow time fix, v9.13), stage 16 (node persistence), the net-DoS topics (A3), the owner's UI protections (A5/A6), audit-frontier-pass, fix-f5-f16, fuzz-debts, A8 (the production APK signing keys), mint-fix (v9.14), fix-cp-threshold, fix-mint-window, fix-release-keys, a10-docs, pre-live-audit. The checks on live phones are set aside in the `8-live` block; they do not block development but do block pilots. Next — a mini-testnet on the owner's laptop, then pilots.

## Disclaimer

The project is in pre-release status; the boundaries of the warranties and liability are described in [DISCLAIMER.md](DISCLAIMER.md) (Russian and English).

## License

The source code is open for **non-commercial use**; commercial use and derivative works — under a paid license. Details and the patent strategy — section 16 of the specification.

The license text: [LICENSE](LICENSE) (the Russian original, legally governing) · [LICENSE-ENG.md](LICENSE-ENG.md) (an unofficial English translation).
