# mos-opencore — Upstream PR status

This repository is a **staging repo** for an upstream submission to
[acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg).
Our changes add **System KC loading and cross-KC dependency
resolution** to `OcAppleKernelLib` / `OcMainLib`, so that a Boot-KC
kext can declare a class from the System KC (for example
`IOFramebuffer` out of `com.apple.iokit.IOGraphicsFamily`) as a
direct linker dependency.

## Submission status

- **PR:** [acidanthera/OpenCorePkg#600 — OcAppleKernelLib: System KC loading and cross-KC dependency resolution](https://github.com/acidanthera/OpenCorePkg/pull/600)
- **Opened:** 2026-04-20
- **State:** **OPEN** — mergeable, awaiting first-pass reviewer
  triage.
- **CI:** **ALL GREEN** on the Uncrustify-fix head `657329a`.
- **Patches:** 6 commits (5 functional + 1 codestyle fix).

### Current CI summary

| Job | Result |
|---|---|
| Uncrustify — Check Codestyle | pass |
| Build — Linux GCC | pass |
| Build — Linux CLANGDWARF | pass |
| Build — Linux CLANGPDB | pass |
| Build — macOS XCODE5 | pass |
| Build — Windows VS2022 | pass |
| Build — Linux Docs | pass |
| Analyze — Python Scripts | pass |
| Analyze — Shell Scripts | pass |
| Analyze — Coverity | skipped (gated to internal runs) |
| Analyze — Documentation | skipped (gated to internal runs) |

Full per-job breakdown with run URLs: [`upstream-pr/PR_600_STATUS.md`](./upstream-pr/PR_600_STATUS.md).

Live re-check at any time:

```sh
gh pr checks 600 --repo acidanthera/OpenCorePkg
```

### Review state

- No reviewer assigned yet. Expected triagers based on historical
  cadence: `vit9696`, `mhaeuser`.
- Typical turnaround on non-trivial `OcAppleKernelLib` PRs: 1 to
  3 weeks from submission to first review pass.
- No style, build, or test failures to respond to as of 2026-04-20.

## What a visitor should do

If you want to try the System-KC-loading patches on your own
OpenCorePkg checkout, you have two options.

### Option A — apply the upstream-facing patch series

The patch set staged in `upstream-pr/patches/` is exactly what
maintainers at acidanthera are reviewing. These are suitable for
applying to a fresh OpenCorePkg clone:

```sh
git clone https://github.com/acidanthera/OpenCorePkg.git
cd OpenCorePkg

# Apply the series (order matters)
for p in /path/to/mos-opencore/upstream-pr/patches/*.patch; do
    git am < "$p"
done

# Build
./build_oc.tool DEBUG
```

If you are just reviewing, it may be faster to read the PR on GitHub
directly: https://github.com/acidanthera/OpenCorePkg/pull/600.

### Option B — overlay our fork on OpenCore 1.0.8

The top of this repo carries the same patches as an overlay on
OpenCore 1.0.8 (`dab2d91`) for local iteration. Usage instructions
are in [`README.md`](./README.md).

### System KC staging

The patches expect `EFI/OC/SystemKernelExtensions.kc` on your
OpenCore ESP. On the host, copy it from the installed macOS:

```sh
cp /System/Library/KernelCollections/SystemKernelExtensions.kc \
    /path/to/ESP/EFI/OC/SystemKernelExtensions.kc
```

Version-locked to the installed macOS build — regenerate after any
OS update.

## What to expect next

1. **Reviewer assignment** from acidanthera — likely 1 to 3 weeks.
2. **Possible style or doc nits** — will be folded into patch 6
   (the style commit), so the functional patches 1 to 5 stay
   bisectable.
3. **Possible reviewer asks:**
   - A `Utilities/TestProcessKernel` command-line mode that
     exercises the System KC path end-to-end.
   - Discussion of the APFS-unreadable-from-EFI rationale for
     staging the KC on the ESP.
4. We will update [`upstream-pr/PR_600_STATUS.md`](./upstream-pr/PR_600_STATUS.md)
   when the state changes (reviewer engaged, feedback, merged, or
   declined).

## Scope note

This repo is **not an OpenCore fork for general use**. The
`System-KC-injection` feature here is an open research item for the
mos suite. The mos suite's active path for macOS 15 is Boot-KC-only
injection with a runtime kernel patcher
([mos-patcher](https://github.com/MattJackson/mos-patcher)); these
patches remain relevant for the case where a kext genuinely needs a
System-KC class as a direct linker dependency.

The authoritative OpenCore project lives at
https://github.com/acidanthera/OpenCorePkg. Once PR #600 merges,
this staging repo will be archived or kept only as a historical
record of the submission.

## Licensing

- Upstream OpenCorePkg files: BSD-3-Clause (unchanged).
- This overlay / staging repo (README, LICENSE.mos-additions,
  glue files): AGPL-3.0.
- Patches destined for upstream carry upstream's BSD-3-Clause
  license and do not introduce AGPL into acidanthera's tree.

## Cross-references

- Upstream PR: https://github.com/acidanthera/OpenCorePkg/pull/600
- Detailed CI snapshot: [`upstream-pr/PR_600_STATUS.md`](./upstream-pr/PR_600_STATUS.md)
- Submission package cover letter: [`upstream-pr/SERIES.md`](./upstream-pr/SERIES.md)
- GitHub PR body source: [`upstream-pr/PR_DESCRIPTION.md`](./upstream-pr/PR_DESCRIPTION.md)
- Test plan: [`upstream-pr/TESTING.md`](./upstream-pr/TESTING.md)
- Mos-suite dashboard (every upstream contribution across the suite):
  `/Users/mjackson/mos/paravirt-re/upstream-submissions-status.md`
