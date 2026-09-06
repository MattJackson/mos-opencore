# v0.5 — research-complete, not active path

OpenCore 1.0.8 patches for System KC injection — teaches OpenCore to load the System KC at EFI time and resolve cross-KC dependencies during prelink.

## What this release provides

7 modified OpenCore source files + cumulative diffs. Lets a Boot-KC-injected kext name a System-KC class (e.g. `IOFramebuffer` from `IOGraphicsFamily`) as a direct linker dependency.

## Status — research-complete, not in production use

The mos suite's active approach uses [mos15-patcher](https://github.com/MattJackson/mos15-patcher) to hook `IONDRVFramebuffer` in place from Boot KC, so System KC injection isn't needed today. These patches stay relevant if a future kext requires *static* link-time resolution of a System-KC class.

Measured progress: System KC loads cleanly, 1590 symbols + 56 vtables resolved for `IOGraphicsFamily`. Final `InternalPrelinkKext` step still fails for the prototype QEMUDisplay test kext — likely one unresolved symbol or vtable edge case, debugging requires `DisplayLevel` with `DEBUG_INFO` (`0x80000042`).

## License

BSD-3-Clause; OpenCore-derived files retain BSD-3-Clause attribution per upstream.
