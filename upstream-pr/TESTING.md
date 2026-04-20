# Testing

The following matrix was executed before submission. Reproduction
steps are included so a reviewer can re-run any tier locally in one
sitting.

Assumed prerequisites (matches the OpenCorePkg README):

- macOS 13 or newer with Xcode 16 (or a Linux host with EDK2
  cross-compile setup).
- Clone of `acidanthera/OpenCorePkg` at (or after) master branch
  commit `dab2d91b0cba` - the series applies cleanly through 1.0.8
  and with minor context fuzz against newer tips.
- The five patches from `patches/` applied in order.

## Unit-level: `TestProcessKernel`

Purpose: smoke test the library with no EFI at all, ensuring
`OcKernelProcessPrelinked ()` remains functional with and without a
System KC.

```sh
cd OpenCorePkg
make -C Utilities/TestProcessKernel
Utilities/TestProcessKernel/ProcessKernel \
    /System/Library/KernelCollections/BootKernelExtensions.kc
```

Expected:

- Exit status 0.
- `out.bin` written to the current directory.
- No new lines in stderr compared to a baseline run against
  unmodified sources.

With the staged System KC path:

```sh
# Copy the current System KC next to the test harness so the harness
# has a representative sample; TestProcessKernel itself does not read
# the System KC today, so this is a smoke for build integrity only.
cp /System/Library/KernelCollections/SystemKernelExtensions.kc /tmp/
ls -lh /tmp/SystemKernelExtensions.kc
Utilities/TestProcessKernel/ProcessKernel \
    /System/Library/KernelCollections/BootKernelExtensions.kc
```

(Should behave exactly like the baseline above; the harness does not
activate the System KC path, which is exercised through OpenCore
proper.)

## Integration-level: OpenCore image boot

Purpose: end-to-end verification that
`OcKernelFileOpen ()` loads the System KC, hands it to
`OcKernelProcessPrelinked ()`, and that Boot-KC kext injection
succeeds for a kext whose `OSBundleLibraries` references a System-KC
class.

1. Build an OpenCore image (DEBUG, XCODE5):
   ```sh
   cd OpenCorePkg
   ./build_oc.tool DEBUG
   ```
2. Populate an ESP structure under `EFI/OC/` with the built
   `OpenCore.efi`, the reviewer's `config.plist`, the usual drivers,
   and any kext of choice in `EFI/OC/Kexts/`.
3. Copy the System KC onto the ESP:
   ```sh
   sudo cp /System/Library/KernelCollections/SystemKernelExtensions.kc \
       /Volumes/EFI/EFI/OC/
   ```
4. In `config.plist`, set:
   ```xml
   <key>Misc</key>
   <dict>
     ...
     <key>Debug</key>
     <dict>
       <key>Target</key><integer>67</integer>
       <key>DisplayLevel</key><integer>2147483906</integer>
     </dict>
   </dict>
   ```
   (`Target=67`: serial + file log. `DisplayLevel=0x80000042`:
   `DEBUG_INFO | DEBUG_WARN | DEBUG_ERROR`.)
5. Boot the OpenCore image. Inspect the OpenCore log
   (`EFI/OC/opencore-*.txt`). Lines described in the PR description
   appear in the expected order.
6. Log into macOS and run
   `kextstat | grep -i <your-kext-bundle-id>` to confirm the injected
   kext is loaded at runtime.

## Regression: no System KC staged

1. Remove `EFI/OC/SystemKernelExtensions.kc` from the ESP.
2. Boot the image with the same DEBUG settings.
3. Confirm:
   - Zero occurrences of "System KC" in the log.
   - `OC: Prelinked status - Success` still present.
   - macOS boots normally.

## Negative: malformed System KC

| Scenario | Staging | Expected log | Expected behaviour |
|---|---|---|---|
| Non Mach-O file | `echo bogus > SystemKernelExtensions.kc` | `OCAK: System KC header is not a valid Mach-O` + `OC: System KC parse failed - Invalid Parameter` | Boot-KC-only injection continues |
| Empty file | `: > SystemKernelExtensions.kc` | `OCAK: System KC header is not a valid Mach-O` | As above |
| Truncated (first 1 KiB only) | `head -c 1024 <real.kc> > SystemKernelExtensions.kc` | `OCAK: System KC header is not a valid Mach-O` or `OCAK: System KC __TEXT_EXEC inner init failed` depending on truncation point | As above |
| No `LC_DYLD_CHAINED_FIXUPS` | Use a non-KC Mach-O (e.g. BootKernelExtensions.kc's inner kernel) | `OCAK: System KC ready (... bytes, 0 chained fixups)` | No faults. Fileset lookups will subsequently miss and fall back to the kernel stub. |
| Version skew (wrong macOS) | Copy a 14.x System KC into a 15.x install | System KC loads; later kext prelink may fail with `EFI_LOAD_ERROR` for symbols that moved between versions | OpenCore emits a WARN; macOS boot may fail. Document this; no corruption. |

None of the malformed cases should produce a hang, page fault, or
silent wrong-address write. If a reviewer observes one, please attach
the malformed buffer and log.

## Manual reviewer steps

If you want to poke at the feature without macOS at all, the smallest
interesting exercise is:

```sh
cd OpenCorePkg
for p in patches/*.patch; do
    git am < "$p"
done

./build_oc.tool DEBUG
# No runtime path is exercised, but the build success confirms that
# all five patches apply and compile on your toolchain.
```

To exercise the fixup walkers without booting macOS, the quickest
path is to add a trivial test binary under `Utilities/` that opens a
System KC from disk and calls `PrelinkedContextLoadSystemKC ()`. That
stub intentionally isn't shipped as part of this series to keep the
patches minimal; happy to add one in a follow-up if useful.

## What was NOT tested

Stated explicitly to avoid false assurance:

- arm64e Apple-silicon end-to-end boot. The code compiles for aarch64
  and the pointer-format branches would select the 4-byte stride path,
  but no live run has been performed on an M-series Mac.
- 32-bit kexts. OpenCore's 32-bit path remains untouched; the System
  KC branch is guarded by `!Context->Is32Bit` or by `Is32Bit == FALSE`
  where it matters. No 32-bit regression check was performed beyond
  confirming the code still compiles.
- macOS versions below 11. These have no System KC, so the code path
  is simply dormant (`mOcStorage` lookup returns `NULL`), but no
  targeted boot test was done on 10.15 or earlier during this cycle.
