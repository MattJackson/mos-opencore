# System KC Loading and Cross-KC Dependency Resolution

## Summary

Adds an optional System Kernel Collection loading path to
OcAppleKernelLib so that Boot KC kext injection can satisfy
`OSBundleLibraries` entries that resolve to classes in the System KC
(for example `com.apple.iokit.IOGraphicsFamily`, which has lived outside
the prelinkedkernel since macOS 11). The existing injection flow is
unchanged whenever no System KC is present on the OpenCore ESP.

## Motivation

On macOS 10.15 and earlier, Apple's prelinkedkernel contained every kext
OpenCore's `OcAppleKernelLib` needs to see in order to satisfy a Boot-KC
kext's library dependencies at prelink time. Starting with macOS 11, kext
bundles split into two immutable kernel collections: the Boot KC (which
OpenCore still injects into) and the System KC (which ships on the
sealed APFS system volume). Common library kexts such as
`IOGraphicsFamily`, `AppleGraphicsDeviceControl`, `AGPM`, and
`IOUSBFamily` moved to the System KC.

A Boot-KC kext whose `OSBundleLibraries` names a System-KC class today
fails at `InternalSolveSymbol ()` with "library kext ... not found";
there is no mechanism for the prelinker to reach into the System KC.

This series teaches OpenCore to optionally load a System KC staged on
its own EFI partition and consult its `LC_FILESET_ENTRY` table when
resolving kext dependencies, enabling static linker resolution for
kexts that extend System-KC classes. When the staged file is absent,
nothing changes: the existing Boot-KC-only code path runs.

## Design

The feature is implemented in three tiers:

1. **Context state (`PrelinkedContext`).** Five optional fields are
   added to hold the raw buffer, its size, outer and inner Mach-O
   contexts (the inner one covering `__TEXT_EXEC` for Boot-KC-style
   collections, or an alias of the outer for System-KC-style ones),
   and a `SystemKCValid` gate.

2. **Collection preparation (`PrelinkedContextLoadSystemKC ()`).** This
   new API takes ownership of the raw KC buffer, initialises the outer
   and inner Mach-O contexts, wires the shared symtab via
   `MachoInitialiseSymtabsExternal ()`, and applies
   `LC_DYLD_CHAINED_FIXUPS` on the outer collection in place so
   downstream pointer reads observe resolved virtual addresses.
   A new static helper `InternalApplyKernelCollectionFixups ()`
   encapsulates the chain walk. Only the `MACH_DYLD_CHAINED_PTR_64_*`
   and `MACH_DYLD_CHAINED_PTR_X86_64_KERNEL_CACHE` pointer formats are
   supported; other encodings are skipped.

3. **Dependency resolution (`InternalCachedPrelinkedKext`).** When the
   Boot KC does not contain a requested bundle identifier and the
   boot is a kernel-collection boot, a new branch calls
   `InternalFindSystemKCDependency ()`. That routine walks the System
   KC's `LC_FILESET_ENTRY` load commands, matches on bundle id, and
   materialises a `PRELINKED_KEXT` that aliases the existing KC
   buffer — the Mach-O context is initialised with the fileset
   entry's `FileOffset` as `HeaderOffset`, exactly like Boot-KC
   dependents. Each fileset entry also ships its own
   `LC_DYLD_CHAINED_FIXUPS`; those are applied by a second static
   helper `InternalApplyFilesetKextFixups ()`. If no match is found,
   the pre-existing kernel-symbol stub fallback runs unchanged.

4. **Link-phase adjustment (`Link.c`).** System KC symbols carry raw
   fileset virtual addresses (typically in the 1 MB .. 512 MB band).
   Feeding those unchanged into `InternalSolveSymbolValue ()` produces
   RIP-relative relocations that can't reach the target after the
   kernel is slid into place. `InternalSolveSymbolNonWeak ()` now
   applies the same
   `+KERNEL_FIXUP_OFFSET + KERNEL_ADDRESS_BASE` translation that
   `KcFixupValue ()` performs for vtables. Kernel-space values, zero,
   and weak test placeholders are not translated.

5. **Glue (`OpenCoreKernel.c`).** A new file-open-time hook attempts
   to load `EFI/OC/SystemKernelExtensions.kc` from `mOcStorage` on
   the first apfs/kernel-cache access, stashing the buffer in three
   file-scope statics. `OcKernelProcessPrelinked ()` then hands the
   buffer to `PrelinkedContextLoadSystemKC ()`; ownership transfers
   to the context, which frees it via `PrelinkedContextFree ()`.

## What's in each patch

- **0001** `OcAppleKernelLib: Add System KC context loading`
  Grows `PRELINKED_CONTEXT` with optional System KC state; adds
  `PrelinkedContextLoadSystemKC ()` and the
  `InternalApplyKernelCollectionFixups ()` helper. No caller yet.
- **0002** `OcAppleKernelLib: Resolve cross-KC deps via LC_FILESET_ENTRY`
  Adds `InternalFindSystemKCDependency ()` and
  `InternalApplyFilesetKextFixups ()`; wires the System-KC branch
  into `InternalCachedPrelinkedKext ()` for kernel-collection boots.
- **0003** `OcAppleKernelLib: Translate System KC symbols at link time`
  Adds the constant address-space translation to
  `InternalSolveSymbolNonWeak ()` and a diagnostic log in
  `InternalCalculateDisplacementIntel64 ()`.
- **0004** `OcMainLib: Stage System KC for prelinked injection`
  Reads `EFI/OC/SystemKernelExtensions.kc` in `OcKernelFileOpen ()`,
  passes it to `PrelinkedContextLoadSystemKC ()` in
  `OcKernelProcessPrelinked ()`.
- **0005** `Changelog: Note System KC loading support`
  Changelog line under the current version section.

Each patch builds in isolation (bisection-friendly): patch 0001 adds
unreferenced state, 0002 and 0003 are independent consumers of it,
0004 is the only patch that can exercise the path end-to-end, and
0005 is documentation.

## What's NOT in scope

The patches intentionally leave the following for separate work:

- **Dual-KC injection.** OpenCore still emits a Boot KC only; we do
  not attempt to rewrite or regenerate the System KC. The feature
  exists purely to resolve Boot-KC-side link dependencies.
- **Exhaustive bounds checking.** The fixup walkers and fileset-entry
  scan trust the structures emitted by Apple's `kmutil`. Defence
  against deliberately malformed System KC buffers belongs in a
  follow-up hardening patch alongside the rest of `OcMachoLib`.
- **Automatic staging.** The System KC file has to be copied to the
  OpenCore ESP manually (or by the installer/snapshot tooling);
  OpenCore does not harvest it at runtime.

## Testing

- `UDK/Build/OpenCorePkg/DEBUG_XCODE5/X64/OpenCore.efi` builds cleanly
  on Xcode 16.2 with EDK2 stable202502 after applying 0001-0005.
- `Utilities/TestProcessKernel` builds and runs against a macOS 15.7.5
  prelinkedkernel. With `SystemKernelExtensions.kc` absent, output is
  byte-identical to the unmodified `make` run. With it staged,
  `IOGraphicsFamily` and `AGPM` are resolved from the System KC (1590
  symbols + 56 vtables) and the remaining link stages run unchanged.
- QEMU/KVM boot of macOS Sequoia (15.7.5) with
  `SystemKernelExtensions.kc` present on the ESP: System KC loads at
  the expected size (~349 MB), fixup pass reports the expected rebase
  count, and the Boot KC prelink completes without regression against
  a stock OpenCore build.

See `TESTING.md` for exact reproduction steps.

## Limitations / known issues

- Support is currently validated for macOS 11 through 15 on x86_64.
  The arm64e kernel-cache path is compiled but has not been exercised
  end-to-end on real hardware; the pointer-format check in the fixup
  walkers would correctly skip unsupported formats, but arm64e review
  is welcome.
- The staged file is version-locked to the booting macOS build.
  Loading a System KC from a different macOS version produces
  unresolved symbols that surface as `EFI_LOAD_ERROR` from
  `InternalPrelinkKext ()`. No automatic version check is done; we
  rely on administrator workflow for now.
- One observed edge case on macOS 15.7.5: a prototype
  `IOFramebuffer`-subclass kext links most symbols successfully but
  still fails final `InternalPrelinkKext` with `EFI_LOAD_ERROR`.
  It is unclear whether this is a missing vtable entry or a late
  relocation issue; the failure is specific to that third-party kext
  and not to the patch set itself. Review comments on which additional
  `OSBundleLibraries` entries might be needed, or hints about
  System-KC-side vtables that need re-walking, would be gratefully
  received.
- The chained-fixup walker is deliberately simple. It does not
  currently distinguish `cacheLevel` values beyond treating both
  encountered values as "Target is the resolved virtual address",
  which matches observed macOS 11-15 behaviour but would need a
  revisit if Apple introduced a format with fully absolute
  pointers.

## References

- `Library/OcAppleKernelLib/KernelCollection.c` - upstream's existing
  `KcFixupValue ()` and `MACH_DYLD_CHAINED_*` handling, which
  motivated the helper split and the address translation used in
  patch 0003.
- Apple's `kmutil(8)` documentation describes the `LC_FILESET_ENTRY`
  layout used by both Boot and System KCs on macOS 11+.
- `dyld3/MachOLoaded.cpp` in Apple's open-source `dyld` tarball is
  the canonical reference implementation of the chain walk the
  helpers in 0001/0002 reproduce (subset that EFI-side code needs).
