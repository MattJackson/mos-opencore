# upstream-pr

Staged upstream submission package for
[acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg).
The contents of this directory are exactly what a maintainer would
receive if this PR were sent today.

## Contents

| File | Purpose |
|---|---|
| `SERIES.md` | Cover letter: summary, motivation, architecture, per-patch breakdown, testing, limitations, references. |
| `PR_DESCRIPTION.md` | Body of the GitHub PR (paste into `gh pr create --body-file`). |
| `TESTING.md` | Full test plan with reproduction commands. |
| `CHANGELOG_ENTRY.md` | The single `Changelog.md` line this PR adds. |
| `patches/` | Five format-patch-style files, ready for `git am`. |

## Status

**DRAFT** (prepared 2026-04-20)

State transitions:
- [x] DRAFT - package assembled, source cleanup committed locally
- [ ] SUBMITTED - PR opened on github.com/acidanthera/OpenCorePkg
- [ ] IN REVIEW - acidanthera reviewer engaged
- [ ] MERGED
- [ ] DECLINED

## How to submit

Prerequisite: a fork of `acidanthera/OpenCorePkg` under your GitHub
account, plus `gh` CLI authenticated.

### Option A: apply commits and push a branch

```sh
# From a fresh clone of your fork of acidanthera/OpenCorePkg at origin/master
cd /path/to/your/OpenCorePkg-fork
git checkout -b system-kc-loading origin/master

# Apply the patches in order
for p in /Users/mjackson/Developer/opencore-mos15/upstream-pr/patches/*.patch; do
    git am < "$p"
done

git push -u origin system-kc-loading

# Open the PR using the staged description
gh pr create \
    --repo acidanthera/OpenCorePkg \
    --title "OcAppleKernelLib: Add System KC loading and cross-KC dependency resolution" \
    --body-file /Users/mjackson/Developer/opencore-mos15/upstream-pr/PR_DESCRIPTION.md
```

### Option B: format-patch mail workflow (if acidanthera later adopts one)

```sh
# Patches already formatted; just attach them.
ls /Users/mjackson/Developer/opencore-mos15/upstream-pr/patches/*.patch
```

## Maintainer-side verification

The fastest way for an acidanthera reviewer to validate locally:

```sh
# In their OpenCorePkg checkout:
for p in *.patch; do
    git am < "$p" || { echo "FAILED on $p"; break; }
done

# Build
./build_oc.tool DEBUG

# Smoke test with the test harness
make -C Utilities/TestProcessKernel
Utilities/TestProcessKernel/ProcessKernel \
    /System/Library/KernelCollections/BootKernelExtensions.kc
```

Full integration flow is documented in `TESTING.md`.

## Notes for the submitter (us)

- The repository's README and `LICENSE.mos-additions` document the
  fork-local usage and do **not** go upstream.
- The source tree under `Include/`, `Library/`, and `Utilities/`
  carries BSD-3-Clause headers matching upstream. Every file header
  and debug string has been scrubbed of fork-local identifiers
  (`OpenCore15`, `mos15`, etc.).
- Do not push the patches/ series to acidanthera directly; open a PR
  from a branch on the submitter's fork so reviewer comments and CI
  have a home.
- When a patch is requested to be reworked, amend the patch file in
  `patches/` **and** land a matching commit on our fork's `main` so
  the two representations stay in sync.
