# Changelog

All notable changes to this repository are documented here.

The format is based on [Keep a Changelog 1.1.0](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

This CHANGELOG tracks the fork's meta state only (README, staged upstream PR
package, overlay notes, repo hygiene). Upstream OpenCore has its own release
cadence and changelog at
[acidanthera/OpenCorePkg](https://github.com/acidanthera/OpenCorePkg). Do not
look here for upstream-release-level changes.

## [Unreleased]

### Added

- Repository hygiene meta files: `CHANGELOG.md`, `SECURITY.md`,
  `CODE_OF_CONDUCT.md`, `CONTRIBUTING.md`.

### Changed

- Relicensed the fork-local additions (`LICENSE.mos-additions`) from
  AGPL-3.0 to BSD-3-Clause to match upstream OpenCore. Upstream
  OpenCorePkg files remain BSD-3-Clause under their original headers and
  `LICENSE.txt`. Standardized `README.md` (badge row, Contributing /
  Security / Changelog / License sections).

## [0.5.0] - 2026-04-20

Research-complete snapshot. Not on the mos suite's active injection path.

### Added

- OpenCore 1.0.8 source patches teaching OpenCore to load the System KC at
  EFI time and resolve cross-KC dependencies during prelink.
- Seven modified OpenCore source files plus cumulative standalone diffs.
  Lets a Boot-KC-injected kext name a System-KC class (for example
  `IOFramebuffer` from `com.apple.iokit.IOGraphicsFamily`) as a direct
  linker dependency.
- `PrelinkedContextLoadSystemKC ()` in `OcAppleKernelLib` — takes ownership
  of a staged System KC buffer, initialises outer and inner Mach-O
  contexts, wires the shared symbol table, and applies
  `LC_DYLD_CHAINED_FIXUPS` in place.
- `InternalFindSystemKCDependency ()` in `OcAppleKernelLib` — walks
  `LC_FILESET_ENTRY` commands by bundle identifier and synthesises a
  `PRELINKED_KEXT` with a real Mach-O context for cross-KC resolution.
- `OpenCoreKernel.c` path for reading `EFI/OC/SystemKernelExtensions.kc`
  from the OpenCore ESP via `mOcStorage` and handing it to the new loader.
- `upstream-pr/` staging package for `acidanthera/OpenCorePkg` (cover
  letter, PR description, series of five functional patches plus one
  codestyle patch, test plan, changelog-entry snippet).

### Changed

- Repository positioned as an upstream-PR staging repo only; the mos
  suite's runtime path no longer consumes this fork.
- Measured status: System KC loads cleanly; `IOGraphicsFamily` and `AGPM`
  `LC_FILESET_ENTRY` entries found and parsed; chained fixups apply
  correctly; 1590 symbols and 56 vtables resolved for `IOGraphicsFamily`.
  Prototype `IOFramebuffer`-subclass test kext still fails at
  `InternalPrelinkKext` with `Load Error` — likely one remaining
  unresolved symbol or vtable edge case; debugging requires `DisplayLevel`
  with `DEBUG_INFO` (`0x80000042`).

### Licensing

- Upstream OpenCorePkg files remain under BSD-3-Clause.
- Fork-local overlay, README, and glue files are AGPL-3.0 per
  `LICENSE.mos-additions`.
- Patches destined for upstream carry the upstream BSD-3-Clause license and
  do not introduce AGPL into acidanthera's tree.

[Unreleased]: https://github.com/MattJackson/mos-opencore/compare/v0.5.0...HEAD
[0.5.0]: https://github.com/MattJackson/mos-opencore/releases/tag/v0.5.0
