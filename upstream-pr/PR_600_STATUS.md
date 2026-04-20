# PR #600 — CI Status

**PR:** [acidanthera/OpenCorePkg#600 — OcAppleKernelLib: System KC loading and cross-KC dependency resolution](https://github.com/acidanthera/OpenCorePkg/pull/600)
**Snapshot taken:** 2026-04-20
**State:** OPEN, MERGEABLE, no review decision yet (awaiting acidanthera reviewer)
**Author:** MattJackson (Matthew Jackson <matthew@pq.io>)
**Head commit:** `657329a` (uncrustify fix applied on top of original 5-patch series)

## Verdict

**ALL CHECKS GREEN.** Every required CI job on the post-uncrustify head
passed. Post-Uncrustify-fix (`657329a`) run completed cleanly — no
follow-up compile-matrix failures, no style regressions, and no new
issues surfaced by the Linux, macOS, or Windows builds. Nothing to fix
right now.

## Per-job results (run 24693544415 / 24693544419 / 24693544420)

| Job | Workflow | Result | Duration | URL |
|---|---|---|---|---|
| Check Codestyle | Uncrustify | **pass** | 48s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544419/job/72220979505 |
| Linux GCC | Build | **pass** | 18m16s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979472 |
| Linux CLANGDWARF | Build | **pass** | 18m44s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979459 |
| Linux CLANGPDB | Build | **pass** | 20m19s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979485 |
| macOS XCODE5 | Build | **pass** | 16m39s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979461 |
| Windows VS2022 | Build | **pass** | 29m34s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979448 |
| Linux Docs | Build | **pass** | 9m5s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544415/job/72220979470 |
| Python Scripts | Analyze | **pass** | 36s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544420/job/72220979502 |
| Shell Scripts | Analyze | **pass** | 22s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544420/job/72220979467 |
| Coverity | Analyze | skipped | 0s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544420/job/72220979877 |
| Documentation | Analyze | skipped | 0s | https://github.com/acidanthera/OpenCorePkg/actions/runs/24693544420/job/72220979788 |

Coverity and Documentation jobs are gated and only run on acidanthera
internal runs or manual dispatch; skipping is the expected behaviour
for an external PR.

## Compile matrix verdict

- **Linux GCC** — pass
- **Linux CLANG (CLANGDWARF + CLANGPDB)** — pass
- **macOS XCODE5** — pass
- **Windows VS2022** — pass

All four of the target compile environments came back clean on the
post-uncrustify head. There is nothing we need to patch on top of
`657329a` based on CI output alone. Any further changes will come from
reviewer feedback, not from a test failure.

## Commit list (PR #600)

1. `2bb5794` — OcAppleKernelLib: Add System KC context loading
2. `64dabc6` — OcAppleKernelLib: Resolve cross-KC deps via LC_FILESET_ENTRY
3. `6c09720` — OcAppleKernelLib: Translate System KC symbol addresses at link time
4. `189fe0e` — OcMainLib: Stage System KC for prelinked injection
5. `1cb29bb` — Changelog: Note System KC loading support
6. `657329a` — OcAppleKernelLib: Fix uncrustify codestyle for System KC patches

## Follow-up fix patch

**Not required.** No compile-matrix failure to respond to. We will
prepare follow-up patches only if and when acidanthera reviewers ask
for changes; right now the PR is in "wait for review" state.

## What to watch for next

1. **Reviewer assignment.** acidanthera PRs are typically triaged by
   `vit9696` or `mhaeuser`. Expected first-pass turnaround on a
   non-trivial OcAppleKernelLib PR: 1 to 3 weeks (historical cadence
   from their closed PRs).
2. **Style nits.** Their Uncrustify job caught one pass already — if
   further style feedback comes, amend patch 6 (the cleanup commit)
   rather than the functional patches 1 to 5, so bisect stays clean.
3. **Test coverage question.** Likely reviewer ask: "is there a
   `Utilities/TestProcessKernel` invocation that exercises System KC
   loading?" We have the `ProcessKernel` test tool call documented in
   `TESTING.md`; be ready to port that into a new command-line mode if
   they request it.
4. **System KC staging path.** A reviewer may push back on reading
   the kernel collection from `EFI/OC/SystemKernelExtensions.kc`.
   Rationale ("sealed APFS is unreadable from EFI") is already in
   `OcMainLib` commit body, but have the alternate-path discussion
   ready: we considered APFS driver support and rejected it on
   complexity grounds.

## Live re-check

```sh
# Refresh status any time
gh pr checks 600 --repo acidanthera/OpenCorePkg

# Or the structured version
gh pr view 600 --repo acidanthera/OpenCorePkg --json statusCheckRollup,reviewDecision,state
```
