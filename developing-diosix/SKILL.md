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
and strictly bounded so that every change passes `reviewing-diosix` on the
first pass without pre-commit rework.

## Core Mandates

1. **Working, Complete & Buildable Code (Table Stakes)**: Every change must
   compile cleanly across both the Root VM/hypervisor build and the host unit
   test suite (`./scripts/build.sh` and `./scripts/build.sh test`). Never leave
   broken code, stubbed `TODO` placeholders, mocked production paths, or
   partially implemented features.
2. **Line & File Limits (Componentize into Subfolders, Never Truncate)**: Wrap
   all code and comments at **<= 80 columns**. Keep source files **<= 500
   lines**. When a file grows past 500 lines, never truncate its functionality
   or tests; decompose it into a dedicated subfolder (`panes/vm.zig`,
   `vm/run.zig`, `tests/gui.zig`) rather than proliferating flat
   `prefix_subname.zig` files (`panes_vm.zig`, `vm_run.zig`, `tests_gui.zig`),
   and wire every submodule into the parent's `test` block
   (`_ = @import("vm/run.zig");`).
3. **Subfolders Over Prefix-Underscore Filenames & Zero Copy-Paste**: Use
   concise `snake_case` filenames without redundant project, directory, or
   sibling prefixes (use `api.zig`, `main.c`, `vm/info.zig`, and
   `tests/nav.zig`, never `diosix_api.zig`, `diosix-alsa-init.c`,
   `vm_info.zig`, or `tests_nav.zig`). Never duplicate shared primitives (such
   as `Spinlock`, page-alignment math, path-traversal validators, or ABI
   constants) across modules; import them from `hypervisor/interface/`,
   `hypervisor/core/atomic.zig`, or `common.zig`.
4. **Idiomatic Dispatch**: Never write `if (std.mem.eql(...)) else if (...)`
   chains for commands, flags, or config keys. Define an `enum`, parse with
   `std.meta.stringToEnum(EnumType, slice)`, and dispatch with `switch`.
5. **Explicit Fixed-Width ABI Types, Lowercase Hex & Named Bitmasks**: Use
   explicit fixed-width types (`u64`, `u32`, `u16`, `u8`) in all ABI/hardware
   structs (`extern struct`, `packed struct`) and 64-bit guest register state —
   never `usize` or C `unsigned long`. Always annotate local variables
   initialized from `@min`/`@max` with a comptime literal
   (`const w: usize = @min(n, 1280);`) so Zig does not infer a narrow integer
   type (`u11`) that overflows later. Write all hexadecimal literals in
   lowercase (`0xfff`, `0xffff_ffff`, never `0xFFF`), and replace magic
   bitmasks/offsets (`(vaddr & 0xfff) > 0xffc`) with named constants and
   self-documenting helpers (`PAGE_OFFSET_MASK`,
   `crossesPageBoundary(vaddr, @sizeOf(u32))`).
6. **Dynamic Allocation Over Static Guessing**: Only early M-mode boot code
   executing before heap initialization may use fixed static buffers. Once an
   allocator is available, allocate and grow buffers dynamically rather than
   guessing fixed stack buffer sizes (e.g., avoid `var buf: [4096]u8`).
7. **Memory Ownership & Cleanup**: Pair every allocation (`alloc`, `create`,
   `dupe`, `print`, `mmap`) immediately with `defer` or `errdefer`. Zeroize
   freed guest pages and cryptographic secrets (`std.crypto.secureZero`).
8. **Enterprise Hypervisor Isolation**: Never panic in M-mode due to guest
   input or state; return an SBI error or terminate the offending guest
   subtree. Copy shared guest memory once (`@memcpy`) into hypervisor-owned
   memory before validation to prevent double-fetch (TOCTOU) vulnerabilities.
9. **Mechanism in Code, Policy in Build/Run-Time Config**: Never hardcode
   policy-level settings or tunable defaults in Zig code. Move tunables and
   settings to build-time configuration (`build.zig` options /
   `boot/hypervisor.conf`) or run-time configuration files
   (`/etc/diosix/*.toml`, keeping `key = "value"` on single lines <= 80 cols).
10. **Build Binaries & Images at Build Time, Not on the Fly in Zig**: Avoid
    assembling ELF binaries, machine-code shims, or media images word-by-word at
    runtime in Zig. Keep shims in assembly or Zig source files compiled during
    build time, use dedicated scripts to build media images, and never place
    generated build outputs inside `tools/overlay-common/` (use
    `.zig-cache/overlay-dynamic/`).
11. **Co-located Unit Tests & Separate Integration Tests**: Place unit `test`
    blocks inside the source file they test (calling
    `std.testing.refAllDecls(@This())`). Place multi-component integration
    tests in dedicated `tests/` directories.
12. **Backwards Compatibility Without Legacy Fallback Accumulation**: Preserve
    compatibility for external contracts (SBI hypercall ABI, `/dev/diosix`
    ioctls, documented `dsx` CLI commands/JSON schemas, and user-authored
    `/etc/diosix/*.toml` files), but never accumulate multi-path fallback
    arrays (`DEFAULT_GUEST_CANDIDATES`) for co-versioned internal plumbing.
    When producer and consumer ship together in this repository, update all
    in-tree producers, consumers, and tests atomically to use a single
    canonical path or symbol, and handle persistent disk layout upgrades once
    at the boot/install boundary rather than probing historical paths forever.
13. **When Unsure, Stop and Ask the Lead Maintainer**: Never guess on
    architectural decisions, ambiguous requirements, or missing subsystem
    capabilities. Stop and ask the lead maintainer before proceeding.

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
  `SafeAllocator`, `stringToEnum`, `@ptrCast` for `extern struct`, fixed-width
  ABI types, `@min` type annotations, 80-column formatting with trailing commas,
  and build cache purity).
- **[Memory Management & Dynamic Buffers](./references/memory-and-buffers.md)**:
  M-mode vs. U-mode allocator patterns, dynamic buffer growth, `defer` /
  `errdefer` discipline, secret zeroization, and out-pointer initialization.
- **[Architecture & TigerStyle](./references/architecture-and-style.md)**:
  Complete feature implementation, modular file splitting without losing unit
  tests, zero copy-paste duplication, bounded loops, assertion pairing, and
  overlay/configuration hygiene.
- **[RISC-V & Enterprise Hypervisor](./references/riscv-and-mmode.md)**:
  RV64 assembly conventions, CSR mocking, blast-radius containment, double-fetch
  prevention pattern, IRQ-safe spinlock ordering, and G-stage/PMP isolation.
