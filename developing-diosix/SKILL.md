---
name: developing-diosix
description: >-
  Rules, idioms, and enterprise hypervisor standards for writing Zig 0.17,
  RISC-V assembly, and system software in Diosix. Activate this skill whenever
  writing, refactoring, or planning M-mode hypervisor code, S-mode emulation
  code, or U-mode userland tools (diosix-ctl, diosix-gui, diosix-config,
  guest-shim).
---

# Developing Diosix

Prioritize **Safety > Security > Correctness > Maintainability**. Write
idiomatic Zig 0.17.0 and RISC-V assembly that is self-documenting, memory-safe,
and strictly bounded.

## Core Mandates

1. **Working, Complete & Buildable Code (Table Stakes)**: Every change must
   compile cleanly across both the Root VM/hypervisor build and the host unit
   test suite (`./scripts/build.sh` and `./scripts/build.sh test`). Never leave
   broken code, stubbed `TODO` placeholders, mocked production paths, or
   partially implemented features.
2. **Line & File Limits**: Wrap all code and comments at **<= 80 columns**.
   Keep source files **<= 500 lines**; split larger files into cohesive
   domain submodules before they become monolithic.
3. **Concise Filenames**: Use `snake_case` filenames without redundant project
   or directory prefixes (use `api.zig` and `main.c`, never `diosix_api.zig`
   or `diosix-alsa-init.c`).
4. **Idiomatic Dispatch**: Never write `if (std.mem.eql(...)) else if (...)`
   chains for commands, flags, or config keys. Define an `enum`, parse with
   `std.meta.stringToEnum(EnumType, slice)`, and dispatch with `switch`.
5. **Dynamic Allocation Over Static Guessing**: Only early M-mode boot code
   executing before heap initialization may use fixed static buffers. Once an
   allocator is available, allocate and grow buffers dynamically rather than
   guessing fixed stack buffer sizes (e.g., avoid `var buf: [4096]u8`).
6. **Memory Ownership & Cleanup**: Pair every allocation (`alloc`, `create`,
   `dupe`, `print`, `mmap`) immediately with `defer` or `errdefer`. Zeroize
   freed guest pages and cryptographic secrets (`std.crypto.secureZero`).
7. **Enterprise Hypervisor Isolation**: Never panic in M-mode due to guest
   input or state; return an SBI error or terminate the offending guest
   subtree. Copy shared guest memory once before validation to prevent
   double-fetch (TOCTOU) vulnerabilities.
8. **Mechanism in Code, Policy in Config**: Do not hardcode magic numbers or
   tunable defaults in source code; load operational parameters from
   `/etc/diosix/*.toml` with safe fallback defaults.
9. **Co-located Unit Tests & Separate Integration Tests**: Place unit `test`
   blocks inside the source file they test (calling
   `std.testing.refAllDecls(@This())`). Place multi-component integration
   tests in dedicated `tests/` directories.
10. **When Unsure, Stop and Ask the Lead Maintainer**: Never guess on
    architectural decisions, ambiguous requirements, or missing subsystem
    capabilities. For example, if a rule recommends `allocator.print(...)` and
    a custom allocator (such as M-mode's `HeapAllocator`) does not directly
    provide a `print` method, stop and ask the lead maintainer whether to
    implement one or use an alternative before proceeding.

## M-Mode vs. U-Mode Execution Environments

| Aspect | M-Mode Hypervisor (`hypervisor/`) | U-Mode Tooling (`tools/`) |
| :--- | :--- | :--- |
| **Target** | `riscv64-freestanding-none` | `riscv64-linux-musl` (or host in tests) |
| **Pre-Heap Boot** | Static per-core slabs (`0..31`) in `entry.s` | N/A (OS process environment ready at `main`) |
| **Allocator** | Per-core `pcore.this().allocator.allocator()` | `std.heap.SafeAllocator` / `page_allocator` / `ArenaAllocator` |
| **Crash Policy** | Never panic on guest input; isolate & terminate offending guest VM | Return descriptive error and clean exit code |
| **Hardware Access** | Direct CSR/MMIO (must guard with `if (is_test)` for host unit tests) | `/dev/diosix` ioctls (`api.zig`), DRM/evdev, Linux syscalls |

## Focused Reference Guides

Read the relevant reference file before writing or modifying code:

- **[Zig 0.17.0 Idioms & Builtins](./references/zig-0-17-idioms.md)**:
  Up-to-date Zig 0.17 syntax (`@backingInt`, `@splat`, `allocator.print`,
  `SafeAllocator`, `stringToEnum`, `@ptrCast` for `extern struct`, build
  cache purity).
- **[Memory Management & Dynamic Buffers](./references/memory-and-buffers.md)**:
  M-mode vs. U-mode allocator patterns, dynamic buffer growth, `defer` /
  `errdefer` discipline, secret zeroization, and out-pointer initialization.
- **[Architecture, Modularity & TigerStyle](./references/architecture-and-style.md)**:
  Complete feature implementation, file sizing, naming rules, bounded loops,
  positive invariants, assertion pairing, and configuration externalization.
- **[RISC-V Assembly & Enterprise Hypervisor Rules](./references/riscv-and-mmode.md)**:
  RV64 assembly conventions, CSR mocking, blast-radius containment, double-fetch
  prevention, IRQ-safe spinlock ordering, and G-stage/PMP isolation.
