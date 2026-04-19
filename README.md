# opencore-mos15

OpenCore source patches for macOS 15 (Sequoia) VMs. Overlay on OpenCore 1.0.8.

## What This Adds

OpenCore can normally only inject kexts into the **Boot KC**. Kexts that
depend on classes in the **System KC** (e.g. `IOFramebuffer` from
`com.apple.iokit.IOGraphicsFamily`) can't be injected — the kernel rejects
them with `library kext ... not found`.

These patches teach OpenCore to load the System KC at EFI time and resolve
missing dependencies from it during prelink, so a Boot-KC kext can name a
System-KC class as a dependency.

## Modified Files (overlay on OpenCore 1.0.8)

| File | What |
|------|------|
| `Include/Acidanthera/Library/OcAppleKernelLib.h` | `SystemKC` fields on `PRELINKED_CONTEXT` (buffer, size, MachContexts, valid flag) |
| `Include/Acidanthera/Library/OcMainLib.h` | `OcKernelProcessPrelinked()` accepts `RootFile` |
| `Library/OcAppleKernelLib/Link.c` | DYLD chained-fixup application for System KC kexts |
| `Library/OcAppleKernelLib/PrelinkedContext.c` | `PrelinkedContextLoadSystemKC()` initialises Mach-O context for the System KC and connects symtab via `__TEXT_EXEC` inner context; cleanup in `PrelinkedContextFree()` |
| `Library/OcAppleKernelLib/PrelinkedKext.c` | `InternalFindSystemKCDependency()` walks `LC_FILESET_ENTRY` commands by bundle ID, creates a `PRELINKED_KEXT` with a real `MachContext` |
| `Library/OcMainLib/OpenCoreKernel.c` | Reads `EFI/OC/SystemKernelExtensions.kc` from the EFI partition (`mOcStorage`) and feeds it to `PrelinkedContextLoadSystemKC` |
| `Utilities/TestProcessKernel/ProcessKernel.c` | Test utility passes `NULL` for `RootFile` |

Standalone diffs (cumulative): `opencore15-patch1.diff`, `opencore15-patch2-system-kc.diff`.

## Why on the EFI Partition

The sealed APFS System volume is **not accessible from EFI** — only `boot.efi`
can use the firmlink to read Boot KC. So the System KC has to ship on the
OpenCore EFI partition itself (`EFI/OC/SystemKernelExtensions.kc`, 349MB on
macOS 15.7.5). It's version-locked to a specific macOS build.

## Status (as of writing)

- System KC loads from the EFI partition.
- `IOGraphicsFamily` and `AGPM` fileset entries are found and parsed.
- Chained fixups applied; 1590 symbols, 56 vtables resolved for `IOGraphicsFamily`.
- `IOFramebuffer`-subclass kext (QEMUDisplay) still fails at `InternalPrelinkKext`
  with `Load Error` — likely an unresolved symbol or vtable in the final link
  step. Use `DisplayLevel` with `DEBUG_INFO` (`0x80000042`) to see the actual
  message.

The Lilu-plugin approach (patching `IONDRVFramebuffer` in place — see
[lilu-mos15](https://github.com/MattJackson/lilu-mos15)) is the alternative
currently in production: no System KC injection needed, runs from Boot KC.

## Usage

Overlay these files on a fresh OpenCore 1.0.8 source checkout, then build:

```bash
git clone https://github.com/acidanthera/OpenCorePkg.git --branch dab2d91
cd OpenCorePkg
# overlay
cp -R /path/to/opencore-mos15/Include .
cp -R /path/to/opencore-mos15/Library .
cp -R /path/to/opencore-mos15/Utilities .
ARCHS=(X64) ./build_oc.tool
# Output: UDK/Build/OpenCorePkg/DEBUG_XCODE5/X64/OpenCore.efi
```

## OpenCore Version

Based on OpenCore 1.0.8 (`dab2d91b0cba3bf4d0da4ccf98a6576cc580cdaf`).
When upgrading, diff these files against upstream and merge.

## Part of mos15

- [docker-macos](https://github.com/MattJackson/docker-macos) — Docker image, kext source, build pipeline
- [qemu-mos15](https://github.com/MattJackson/qemu-mos15) — QEMU patches
- [opencore-mos15](https://github.com/MattJackson/opencore-mos15) — OpenCore patches (this repo)
- [lilu-mos15](https://github.com/MattJackson/lilu-mos15) — Lilu patches

## License

OpenCore is BSD-3-Clause. This patch follows the same license as upstream OpenCore.
