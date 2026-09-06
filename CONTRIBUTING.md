# Contributing to mos-opencore

`mos-opencore` is a staging repository for OpenCore patches targeting upstream submission to [acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg). The mos suite itself uses vanilla upstream OpenCore; this repo exists to stage and track our contributions back.

See `STATUS.md` for the current state of the upstream-submission pipeline.

## Posture

This repository is **upstream-PR-staging-only**. Code changes must be authored to go upstream unchanged — no project-internal names, no fork-local debug prefixes, no references to downstream consumers.

## How to propose a patch

Stage new work under `upstream-pr/<topic>/` alongside existing directories like `system-kc-loading/`. Each topic package carries:

- `PR_DESCRIPTION.md` — what the patch does and why, in acidanthera's expected format
- `SERIES.md` — commit order when the topic comprises multiple commits
- `TESTING.md` — how to validate locally against upstream master

## Commit style

Match acidanthera's conventions:

- Subject: `<Module>: <Subject>` where `<Module>` matches the source path (e.g. `OcAppleKernelLib:`, `OcMainLib:`, `Legacy/BootPlatform:`). Check recent merged commits for cadence.
- Imperative mood, under 70 characters, no trailing period.
- Body: explain the rationale; upstream reviewers value context on why something needs to change.
- **No project-internal names** — never reference `mos15`, `mos-opencore`, `mos-docker`, or `OpenCore15` in source, comments, or commit messages.
- **License headers:** new files are BSD-3-Clause matching OpenCore's file header style (see any `Library/` file upstream).

## Trailers

- Local commits may carry `Co-Authored-By:` trailers.
- When submitting upstream, trailers are transformed to `Signed-off-by:` per DCO. This is a one-time filter at PR time, not a continuous discipline.

## Testing

Before proposing a PR:

- Apply the patch to upstream `master` HEAD (not just our fork base).
- Run upstream's test suite where it applies.
- Document new tests or infrastructure requirements in `TESTING.md`.

## Reference

See PR [#600](https://github.com/acidanthera/OpenCorePkg/pull/600) — the System KC loading series — as the reference-staged patch package. Its `upstream-pr/system-kc-loading/` layout is the template.

## License

OpenCore-derived files retain BSD-3-Clause (`LICENSE.txt`). The mos-additions (tooling, documentation) are BSD-3-Clause (`LICENSE.mos-additions`).
