# mos-opencore

OpenCore source patches for running macOS 15 (Sequoia) as a guest in QEMU/KVM. Overlay on OpenCore 1.0.8.

> **Status: v0.5 — experimental; not the active injection path.** The System-KC-injection feature in these patches is an open research item. The mos suite's active approach uses Boot-KC-only injection with a runtime kernel patcher ([mos-patcher](https://github.com/MattJackson/mos-patcher)). These patches remain here for the case where a kext genuinely needs a System-KC class as a direct linker dependency — we're not using that path today.

## What this adds

OpenCore can normally only inject kexts into the **Boot KC**. Kexts whose `OSBundleLibraries` list a class from the **System KC** (e.g. `IOFramebuffer` from `com.apple.iokit.IOGraphicsFamily`) get rejected by the kernel with `library kext ... not found`.

These patches teach OpenCore to load the System KC at EFI time and resolve missing dependencies from it during prelink, so a Boot-KC kext can name a System-KC class as a direct dependency.

## Modified files (overlay on OpenCore 1.0.8)

| File | Purpose |
|---|---|
| `Include/Acidanthera/Library/OcAppleKernelLib.h` | Adds `SystemKC` fields on `PRELINKED_CONTEXT` (buffer, size, MachContexts, valid flag) |
| `Include/Acidanthera/Library/OcMainLib.h` | `OcKernelProcessPrelinked()` gains a `RootFile` parameter |
| `Library/OcAppleKernelLib/Link.c` | Applies DYLD chained fixups for System-KC kexts |
| `Library/OcAppleKernelLib/PrelinkedContext.c` | `PrelinkedContextLoadSystemKC()` initialises a Mach-O context for the System KC and connects symtab via `__TEXT_EXEC` inner context; cleanup in `PrelinkedContextFree()` |
| `Library/OcAppleKernelLib/PrelinkedKext.c` | `InternalFindSystemKCDependency()` walks `LC_FILESET_ENTRY` commands by bundle ID, creates a `PRELINKED_KEXT` with a real `MachContext` |
| `Library/OcMainLib/OpenCoreKernel.c` | Reads `EFI/OC/SystemKernelExtensions.kc` from the EFI partition via `mOcStorage` and feeds it to `PrelinkedContextLoadSystemKC()` |
| `Utilities/TestProcessKernel/ProcessKernel.c` | Test utility passes `NULL` for `RootFile` |

Cumulative standalone diffs: `opencore15-patch1.diff`, `opencore15-patch2-system-kc.diff`.

## Why the System KC lives on the EFI partition

The sealed APFS System volume is **not accessible from EFI** — only macOS's own `boot.efi` can follow the firmlink to read the Boot KC. So to read the System KC pre-boot, we ship it on the OpenCore EFI partition: `EFI/OC/SystemKernelExtensions.kc` (~349 MB on macOS 15.7.5). The file is version-locked to a specific macOS build.

## Where this lands today

**Research-complete, not in production use.** Measured progress:

- System KC loads cleanly from the EFI partition.
- `IOGraphicsFamily` and `AGPM` `LC_FILESET_ENTRY` entries are found and parsed.
- Chained fixups apply correctly; 1590 symbols and 56 vtables resolved for `IOGraphicsFamily`.
- A prototype `IOFramebuffer`-subclass kext (QEMUDisplay) still fails at `InternalPrelinkKext` with `Load Error` — likely one remaining unresolved symbol or a vtable edge case in the final link. Debugging requires `DisplayLevel` with `DEBUG_INFO` (`0x80000042`) to get the actual message.

**Why it's not in the critical path right now:** the mos suite currently uses [mos-patcher](https://github.com/MattJackson/mos-patcher) to hook `IONDRVFramebuffer` in place from Boot KC, so no System KC injection is needed. The patches in this repo stay relevant if we ever want to ship a kext whose *static* link resolution requires a System-KC class (vs. hooking into an existing instance).

## Usage

Overlay these files onto a fresh OpenCore 1.0.8 source checkout, then build:

```bash
git clone https://github.com/acidanthera/OpenCorePkg.git
cd OpenCorePkg
git checkout dab2d91b0cba3bf4d0da4ccf98a6576cc580cdaf   # 1.0.8

# Apply our overlay
git clone https://github.com/MattJackson/mos-opencore /tmp/opencore-mos15
cp -R /tmp/opencore-mos15/Include    .
cp -R /tmp/opencore-mos15/Library    .
cp -R /tmp/opencore-mos15/Utilities  .

# Build (x86_64)
ARCHS=(X64) ./build_oc.tool
# Output: UDK/Build/OpenCorePkg/DEBUG_XCODE5/X64/OpenCore.efi
```

## OpenCore version

Based on OpenCore 1.0.8 (commit `dab2d91b0cba3bf4d0da4ccf98a6576cc580cdaf`). When upgrading, diff each modified file against the new upstream version and merge.

## Part of the mos suite

- [mos-docker](https://github.com/MattJackson/mos-docker) — Docker image, kext source, build pipeline
- [mos-qemu](https://github.com/MattJackson/mos-qemu) — QEMU patches
- [mos-patcher](https://github.com/MattJackson/mos-patcher) — kernel-side hook framework (Lilu replacement)

## License

[GNU AGPL-3.0](LICENSE). OpenCore itself is [BSD-3-Clause](https://github.com/acidanthera/OpenCorePkg/blob/master/LICENSE.txt); the unmodified upstream files retain that license and its attribution requirements. Our modifications and additions in this repo are AGPL-3.0.
