# 4. Development process

The development is carried out by a CLI agent (Vibe) in stages; the owner makes the decisions. All artifacts of the process live in the main repository.

## Process sources

| File (in equality-trust-networks) | Purpose |
|-----------------------------------|---------|
| `docs/development-plan.md` | The stage plan; after each stage it is marked done, with branches and SHAs |
| `prompts/stage-N-prompt.md` | The stage prompt for the CLI agent; committed before the stage starts |
| `reports/REPORT-STAGE-N.md` | The stage report (what was done, findings, metrics, artifacts) |
| `QUESTIONS.md` | The registry of questions (No. 1, No. 2…). Only the owner makes the decision |
| `docs/KNOWLEDGE.md` + `docs/KNOWLEDGE-1..5.md` | The specification: title/table of contents + parts 1–5 by section boundaries; the version is raised only by authorized changelog entries |

## The questions rule

Questions that require the owner's decision (trade-offs, semantics, parameters) are recorded in `QUESTIONS.md` with the status PENDING and the options. **Do not guess**: if the semantics are undefined, the minimal safe version is implemented, the question is recorded, and the decision does not block the rest of the mandate.

## Branching and merge

- Each stage is a **separate branch off a fresh `main`** (`stage<N>-<topic>`).
- Pushing the branch and merging into `main` — **only after the owner's confirmation**.
- The merge into main is performed by the CLI agent under a separate owner permission: merge `--no-ff`, push main, a report with the merge SHA (decision of November 1, 2026).
- Remote branches are preserved; deletion — only upon the owner's explicit confirmation.
- In a conflict with parallel edits of a prompt, the `main` version takes priority.
- The owner performs git operations in a GUI (GittyUp): name only the branches and target SHAs; do not give them console commands.

## CLI session checkpoints

- Do **not** include time-based checkpoints (~30 min) and post-restart reconciliation in CLI prompts — the agent does not execute them (owner's decision of November 1, 2026). The stage prompt is self-sufficient: steps, the readiness criterion, PENDING rules.
- Composition of intermediate commits: WIP code and progress — yes; binaries, build artifacts, scratch — no. **The final commit of a stage — a clean working tree.**

## Repository hygiene

- Do not commit: binaries (even into the root — `.gitignore` covers the known patterns), Flutter/Gradle build artifacts, heavy scratch files. Screenshots and run data — into `reports/`, watch the weight (files >1 MB — only legitimate data, e.g. metrics.csv).
- After a run — **process cleanup**: stop the Gradle daemons, terminate `tnp-node`/`tnp-testnet`/`qemu`/`java`/`go`; before a new run check `pgrep`. The agent does not close java/go itself — this item is included in every prompt with runs.
- Host system clock shifts for accelerated fleets: the issued sudoers grant is used per task; upon completion — restore the clock (host + emulators), re-enable timesyncd, **remove the sudoers rule**.
- Pushing follows the 3-prompt rule: commits accumulate locally; the push — upon confirmation.

## Code style

- Code comments — in Russian.
- File names and identifiers — English.
- Local directories: `~/projects/equality-trust-networks`, `~/projects/anti-farm-simulation`.
- No private data appears in the repositories.

## Documentation updates (after each stage)

1. `docs/KNOWLEDGE.md` (title and table of contents) and the parts `docs/KNOWLEDGE-1..5.md` — a new spec version only via authorized entries (a changelog heading with the date).
2. `docs/development-plan.md` — the stage ✅, branches, SHAs, findings.
3. `prompts/stage-(N+1)-prompt.md` — the prompt of the next stage.
4. This repository (`equality-programming-docs`) — on architecture/process changes; `equality-user-docs` — on changes visible to the user.

→ [Architecture](01-architecture.md) · [Contents](../README.md)
