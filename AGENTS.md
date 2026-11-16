# AGENTS.md

This document provides development guidelines for the ZNextended fork. It retains ZNNext's independently authored implementation and adds a separate, opt-in standard Zygisk compatibility roadmap.

## CORE IDENTITY AND CONSTRAINTS

Before writing, refactoring, or modifying any code, you must internalize the following absolute truths about this project. Violating these is considered a critical failure.

1. The existing ZygiskNext module loader must remain functional, with its current API and selectable hooking backends preserved.
2. This is the independently maintained ZNextended fork, not the original closed-source ZygiskNext implementation or the upstream ZNNext project.
3. A goal of this fork is to independently implement a **separate** standard Zygisk API v5 loader. Never conflate the Zygisk API and ZygiskNext API, or invoke one entrypoint with the other's API table.
4. Standard Zygisk support is **not implemented yet**. Until its real pre/post-specialization lifecycle has been built and tested, do not advertise drop-in compatibility or run standard modules through an incomplete shim.

See also [the staged dual-API implementation plan](docs/dual-api-compatibility.md). Do not install test builds as replacements for working Zygisk providers.

## ZN CLOSED-SOURCE AND STRICT CLEANROOM MANDATE

ZygiskNext (ZN) has been closed-source since version v4-0.9.2. The ZN developers enforce a strict copyright notice prohibiting modifications, redistribution, picking (extracting code snippets/binaries), and claims of succession. 

To comply with this and maintain legal and technical independence, you must adhere to the following Cleanroom Design principles:

1. NO COPYING OR PICKING: Under no circumstances should you copy, paste, extract, or directly translate code, functions, or binaries from the original ZygiskNext repository or its released binaries.
2. REVERSE ENGINEERING + CLEANROOM ONLY: If understanding ZN's behavior is absolutely necessary to ensure API compatibility, you may use reverse engineering to observe its external behavior or API contract. The actual implementation in this repository must be written 100% from scratch (Cleanroom) based on those observations, and must not completely copy or blindly replicate ZN's internal source logic.
3. INDEPENDENT SOLUTIONS: If a bug, crash, or missing feature is identified in ZN, do not simply mirror or port ZN's fix. You must analyze the root cause and design an independent, original solution from first principles.
4. API SURFACE ONLY: Rely only on the publicly documented ZygiskNext Module API specifications for compatibility. Internal implementation details (memory management, injection flow, hook chaining) must remain 100% original and distinct from ZN (unless a specific functionality can strictly only be implemented in that exact manner).

## ARCHITECTURE OVERVIEW

- loader/: The core native component (C++), injected into target processes; it loads modules and manages hooks by lots of hook engines. Its sources are split by area under loader/src/: hook/ (engine selection, inline and PLT dispatch, one file per backend in hook/backend/), api/ (ZygiskNextAPI implementation and the per-version tables), hyos/ (HyperOS Runtime), ipc/ (daemon protocol), companion/ (companion process), module/ (zn_modules.txt matching and loading), config/, process/ and utils/ (ELF/maps helpers, memfd dlopen). Only entry.cpp exports anything (`znn_loader_init`).
- daemon/: The standalone injector executable shipped as bin/<abi>/znnd. NOT part of loader/ and never built by it; it is a Rust crate (cargo) with its own project module, which drives cargo for every shipped ABI. Every `unsafe` block lives under daemon/src/sys/, and the crate root denies `unsafe_code` so that stays enforceable.
- loader/src/include/: The ZygiskNext API header (zygisk_next_api.h). DO NOT EDIT THIS.
- module/: The Magisk / KernelSU / APatch module wrapper. Handles installation, environment setup, permission management, and status reporting. It collects the loader library and the daemon binary from their own project modules when building the zip.
- webui/: The KernelSU WebUI, a Vue 3 app with two switchable skins under src/skins/. material/ renders @m3e/web custom elements (Material 3 Expressive) and miuix/ renders miuix-vue components; each skin is a lazy chunk that owns its own component library, so never import one skin's library from the other. src/ui-mode.ts (localStorage znn_ui, default material) selects the skin, and src/composables/use-app-data.ts feeds both. `npm run dev` stubs the KernelSU bridge through dev/ksu-mock.js, with ?ui= ?theme= ?locale= ?status= ?modules= ?fail= overrides. The @m3e/web theme host is declared in index.html and MUST stay a direct child of <body>: that is the only position in which @m3e/web publishes the Material colour tokens on :root, which the page background depends on.

## DEVELOPMENT GUIDELINES

1. Code Generation and Modification: Always prioritize stability and security when modifying hooking logic in loader/. Use modern C++ standards as defined in the project's build system. Ensure all new code is free of redundant or unnecessary comments. No nonsense comments.
2. Dependency Management: When interacting with Dobby, LSPlt, rv64hook, or more, respect their individual licenses and integration patterns.
3. Error Handling: Fail gracefully. If a ZN module fails to load or a hook fails, log the error clearly and ensure both the daemon and the host process continue to run without crashing.
4. Rust Code (daemon/): Keep every module small and single-purpose; a new file belongs in daemon/src/ rather than growing main.rs. Confine FFI to daemon/src/sys/ and wrap it in safe interfaces, then run `cargo clippy --all-targets -- -D warnings` for the Android targets before finishing.
5. C++ Code (loader/): Keep every file small and single-purpose; a new file belongs in the loader/src/<area>/ directory it fits rather than growing an existing one. Hook backends stay one-per-file under loader/src/hook/backend/ behind the declarations in loader/src/hook/backend.h, and every index in the ZygiskNextAPI tables is chosen by the module's target_api_version in loader/src/api/table.cpp.
6. You must run `fmt.sh` located in the project's root directory.

## WORKFLOW

When tasked with a feature implementation or bug fix, follow this strict sequence:

1. Analyze: Distinguish the existing ZygiskNext API loader from the proposed separate standard Zygisk compatibility runtime.
2. Constraint Check: Explicitly verify that the proposed solution does not violate the Cleanroom Policy or ZN's copyright restrictions (No Picking, No Copying).
3. Design: Outline an original, from-scratch approach to solve the problem. If reverse engineering insights are used, ensure the implementation is purely cleanroom.
4. Implement: Write the code with clear, descriptive variable names.
5. Self-Review: Verify API separation, real specialization timing, independent implementation, correct licenses, safe failure on unsupported capabilities, and no proprietary ZygiskNext code extraction.
