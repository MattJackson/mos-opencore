## What this PR does

Adds optional System Kernel Collection (System KC) loading to
`OcAppleKernelLib` and wires it through `OcMainLib`, so OpenCore can
resolve Boot-KC kext `OSBundleLibraries` dependencies against classes
that live in the System KC (for example `IOGraphicsFamily`,
`IOUSBFamily`, `AGPM`). When no System KC is staged on the OpenCore
ESP, behaviour is identical to today's injection flow.

## Why

Since macOS 11 the System KC hosts a large fraction of kext libraries
that used to be in the prelinkedkernel. OpenCore only injects into the
Boot KC, so any injected kext whose `OSBundleLibraries` names a
System-KC class fails at prelink with `library kext ... not found`.
Teaching OpenCore to optionally consult the System KC closes that gap
for any project whose kexts statically link against System-KC classes.

The feature is strictly opt-in: presence of
`EFI/OC/SystemKernelExtensions.kc` on the ESP activates it, and the
file is version-locked to the installed macOS build.

## Test plan

Reviewer can reproduce with the following minimal flow. Pointers
indicate what output to expect in the OpenCore DEBUG log
(`DisplayLevel=0x80000042`, i.e. `DEBUG_INFO | DEBUG_WARN`).

1. **Unit-level (no macOS required).**
   ```sh
   cd OpenCorePkg
   git am 0001-OcAppleKernelLib-Add-System-KC-context-loading.patch
   git am 0002-OcAppleKernelLib-Resolve-cross-KC-dependencies.patch
   git am 0003-OcAppleKernelLib-Translate-System-KC-symbols-at-link.patch
   git am 0004-OcMainLib-Stage-System-KC-for-prelinked-injection.patch
   git am 0005-Changelog-Note-System-KC-loading-support.patch

   make -C Utilities/TestProcessKernel
   Utilities/TestProcessKernel/ProcessKernel /path/to/prelinkedkernel
   ```
   Expect: exit 0, `out.bin` byte-identical to a run without the
   patches when no System KC is present. No new warnings.

2. **Integration-level (requires a macOS 11+ guest or host).**
   ```sh
   # On macOS host
   sudo cp /System/Library/KernelCollections/SystemKernelExtensions.kc \
     /Volumes/EFI/EFI/OC/

   # Build OpenCore with the patches and install OpenCore.efi to the ESP.
   # Boot with DEBUG_XCODE5 build + DisplayLevel=0x80000042.
   ```
   Expect these log lines (order matters, format verbatim):
   ```
   OC: System KC loaded from EFI partition (NNN bytes)
   OCAK: System KC ready (NNN bytes, NNN chained fixups)
   OCAK: System KC resolved com.apple.iokit.IOGraphicsFamily (VA 0x...)
   OCAK: System KC resolved com.apple.driver.AGPM (VA 0x...)
   ```
   The `OC: Prelinked status - Success` line that usually terminates
   OcKernelFileOpen's cache work must remain. macOS boots normally.

3. **Regression test (no System KC staged).**
   Remove `EFI/OC/SystemKernelExtensions.kc`, rebuild image or simply
   delete the file, reboot. Expect the DEBUG log to contain NONE of
   the `OC: System KC ...` or `OCAK: System KC ...` lines and
   macOS to still boot.

4. **Negative test (corrupt file).**
   Stage a file named `SystemKernelExtensions.kc` that is not a
   Mach-O (`echo not-a-macho > SystemKernelExtensions.kc`). Expect:
   ```
   OC: System KC loaded from EFI partition (N bytes)
   OCAK: System KC header is not a valid Mach-O
   OC: System KC parse failed - Invalid Parameter
   ```
   No panic, no crash; Boot KC injection continues normally.

## Review notes

A few specific spots that might benefit from review eyes:

- `PrelinkedContext.c` `InternalApplyKernelCollectionFixups ()`:
  only the x86_64 and arm64e kernel-cache pointer formats are
  handled. Formats we have not observed in shipping kernel
  collections are deliberately skipped with `continue` - please
  confirm the list is complete enough for your intended targets.
- `PrelinkedKext.c` `InternalFindSystemKCDependency ()`: the fallback
  at the end of `InternalCachedPrelinkedKext ()` still materialises a
  kernel-stub `PRELINKED_KEXT` when a bundle identifier is in
  neither collection. This preserves today's behaviour for non-KC
  dependencies (weak test symbols, etc.); open to changing that if
  you'd prefer the fallback path to disappear when
  `IsKernelCollection` is true.
- `Link.c`: the translation in `InternalSolveSymbolNonWeak ()` uses
  the same constant offsets as `KcFixupValue ()`. If a future
  macOS changes the shared-KC KASLR convention, both sites would need
  updating in sync; consider whether they should share a helper.
- `OpenCoreKernel.c` reads the System KC through `mOcStorage`, which
  anchors to the OpenCore ESP root. If a reviewer prefers a config
  knob (e.g. `Kernel/Scheme/SystemKernelCollectionPath`) I'm happy
  to add one; the current hard-coded filename felt like the minimal
  surface for a first cut.

## Log snippet

With the patches applied and
`EFI/OC/SystemKernelExtensions.kc` staged from macOS 15.7.5:

```
OC: Request is usr-apfs (0), 0(0) for custom Hook
OC: System KC loaded from EFI partition (365821088 bytes)
OCAK: System KC ready (365821088 bytes, 81247 chained fixups)
OCAK: System KC resolved com.apple.iokit.IOGraphicsFamily (VA 0xffffff8149b20000, 1590 fixups)
OCAK: System KC resolved com.apple.driver.AppleGraphicsDeviceControl (VA 0xffffff8149c80000, 56 fixups)
OC: Prelinked status - Success
```

## Checklist

- [x] Code compiles cleanly on `XCODE5 DEBUG` (Xcode 16.2, EDK2
      stable202502).
- [x] `Utilities/TestProcessKernel` runs unchanged in the no-System-KC
      case.
- [x] Changelog entry added.
- [x] Each commit is independently buildable for bisection.
- [ ] Reviewed arm64e pointer-format behaviour end-to-end on physical
      Apple-silicon hardware. (Pointer-format path compiles; no
      Apple-silicon hardware on hand for a real boot test.)
