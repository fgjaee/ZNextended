# ZNextended — dual-API compatibility work

> Status: **foundation only**. This branch **does not yet load standard Zygisk modules**. Do not install it in place of a working Zygisk provider to run modules such as PGS Root.

## Goal

Keep the existing ZygiskNext (`zn_module`) loader while adding a separately
implemented runtime for standard Zygisk (`zygisk_module_entry`) modules.
The operator may select the **process injection strategy** and, for modules
using the ZygiskNext API, the **inline/PLT hook backend**.

A configurable hook backend cannot replace a hooking library bundled privately
inside a third-party module.

## What is implemented

- Existing `zn_module` loading remains unchanged.
- When an explicitly declared module is loaded and exports
  `zygisk_module_entry` instead, the loader reports the interface mismatch and
  **does not call** the standard entry point.
- This is diagnostic recognition, **not standard Zygisk support**. The current
  detection is after `dlopen`, so ELF constructors may already have run.
  Ordinary standard Zygisk modules also are not discovered automatically, since
  ZNNext currently uses `zn_modules.txt`.

## Why a second dlsym is not enough

Standard Zygisk modules are normally initialized in the child process forked
from Android's zygote, and may expect calls before and after app or system-server
specialization. The public Zygisk API v5 includes:

- `onLoad`, `preAppSpecialize`, `postAppSpecialize`,
  `preServerSpecialize`, and `postServerSpecialize` callbacks.
- An API table with `registerModule`, `hookJniNativeMethods`,
  `pltHookRegister/Commit`, `connectCompanion`, `getModuleDir`,
  `setOption`, `getFlags`, and `exemptFd`.
- A root companion entrypoint (`zygisk_companion_entry`).

ZNNext's `zn_module` callback instead runs through its own entrypoint-based
injector and receives a different API table. Late/ptrace/proc injection **does
not by itself recreate zygote's pre-specialization privileges or timing**.
Calling `zygisk_module_entry` with the wrong API pointer would be unsafe.

## Implementation milestones

1. **M0 — distinguish APIs without enabling unsupported calls.** Add clear
   logging and document lifecycle and API differences. This is the initial
   development branch.
2. **M1 — safe discovery and test fixtures.** Build a bounded, read-only ELF
   export scanner before any `dlopen`; recognize standard `zygisk/*.so`
   module layout and per-app inclusion rules; unit-test each supported ABI.
3. **M2 — standard Zygisk lifecycle.** Implement a true zygote-fork-aware
   bootstrap and the v5 API registration/dispatch table. Explicitly fail when
   pre-specialization context cannot be provided; do not emulate it after the
   sandbox is established.
4. **M3 — companions and parity.** Add root companion IPC, module directory
   descriptors, JNI/PLT hooks, options, FD exemptions, and per-module status.
5. **M4 — compatibility tests.** Exercise a harmless standard sample module
   across app and system-server specialization; test existing `zn_module`
   modules for regressions and verify conflict-free startup with exactly one
   configured provider.
6. **M5 — optional integration testing.** Only after M4 passes, assess whether
   proprietary modules such as PGS Root operate without loader collisions.
   Module-native code may still crash, regardless of chosen injector.

## Acceptance criteria

- A ZygiskNext sample works exactly as before.
- A standard Zygisk sample receives correct `onLoad` and specialization
  callbacks under valid JNI and sandbox contexts.
- Unsupported module types or missing early injection contexts fail with a
  useful log, not an unsafe fallback.
- Switching hook backends does not falsely claim to control hooks inside
  third-party libraries.
- Runtime has independent module enable/disable, logs, and a rollback path.

## Licensing and source references

This fork retains upstream ZNNext's GPL-3.0 obligations and attribution.
The standard Zygisk API contract is documented in the public header from
`topjohnwu/zygisk-module-sample`; the goal is an independent implementation,
not redistribution of third-party proprietary loader binaries.

- ZNNext upstream: https://github.com/VeryBaaad/ZNNext
- Zygisk API header: https://github.com/topjohnwu/zygisk-module-sample/blob/master/module/jni/zygisk.hpp
