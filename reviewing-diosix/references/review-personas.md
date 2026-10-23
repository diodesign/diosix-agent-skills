# The 4 Review Personas Checklist

Evaluate every diff and its surrounding module context through all four
personas. Flag and fix every finding before completing the task.

## 1. The Skeptical Veteran Engineer
*Hunts incomplete features, code smells, shortcuts, non-idiomatic Zig, and
structural bloat.*

- [ ] **Complete & Working Implementation**: Does the code build cleanly and
      implement 100% of the design without `TODO` stubs, fake fallbacks, or
      half-wired flags?
- [ ] **Surgical Dead-Code & Orphaned-Test Removal (The Kept-Alive-by-Tests
      Smell)**:
      - Zig lazily analyzes top-level declarations (`fn`, `pub fn`, `var`,
        `const`) and does **not** emit compiler warnings when they are unused.
        Even worse, `std.testing.refAllDecls(@This())` or a leftover `test`
        block will keep an obsolete function or global variable compiling
        cleanly long after its last production caller was removed.
      - Whenever a change replaces a helper, refactors a workflow, or modifies a
        module, grep across `hypervisor/`, `tools/`, `build.zig`, and `scripts/`
        for every touched or superseded function, global variable (`var` /
        `const`), struct field, error tag, and type.
      - Verify that each symbol has at least one **non-test production call
        site or reference** (excluding documented external ABI definitions in
        `hypervisor/interface/` and `tools/driver/diosix.h`).
      - If a symbol has zero production callers—or is referenced **only** inside
        `test` blocks—**delete the dead declaration and its unit tests
        immediately**. Do not retain unused functions, globals, or commented-out
        code "just in case."
- [ ] **No `std.mem.eql` Chains**: Are all command, flag, and key dispatches
      using `enum` + `std.meta.stringToEnum` + `switch`?
- [ ] **Zero Copy-Paste Primitives**: Are shared primitives (`Spinlock`,
      `PAGE_SIZE`, `alignUp`/`alignDown`, `isValidSafeName`, JSON escaping)
      imported from `hypervisor/core/atomic.zig`, `hypervisor/interface/`, or
      `common.zig` rather than duplicated in leaf files?
- [ ] **Fixed-Width ABI Types & `@min` Type Annotations**: Do `extern struct` /
      `packed struct` ABI types and 64-bit guest registers use explicit `u64` /
      `u32` / `u16` / `u8` instead of `usize`? Are `@min` / `@max` calls with
      comptime literals explicitly typed
      (`const row_w: usize = @min(w, 1280);`) so Zig does not infer a narrow
      integer type (`u11`)?
- [ ] **No Magic Numbers, Unnamed Bitmasks, or Uppercase Hex**:
      - **Explanatory-Comment Smell**: Does any numeric literal need a comment
        to explain what it means (`return 15; // Store page fault.`,
        `flags & (1 << 4) // User`)? Replace it with a named constant or `enum`
        tag (`@backingInt(riscv.Cause.store_page_fault)`, `PTE.U`).
      - **Pre-Computed Mask Smell**: Does the code use pre-computed hex masks or
        thresholds like `(vaddr & 0xfff) > 0xffc` or `(pte & 0xe) == 0`?
        Replace them with named constants (`PAGE_OFFSET_MASK`,
        `PAGE_SIZE - size`, `PTE.RWX`) and self-documenting inline helpers
        (`crossesPageBoundary(vaddr, @sizeOf(u32))`, `PTE.isPointer(pte)`).
      - **Lowercase Hexadecimal**: Are all hexadecimal literals written in
        lowercase (`0xfff`, `0xffff_ffff`, never `0xFFF`)?
- [ ] **No Static Buffer Guessing**: Are fixed stack buffers (`var buf:
      [4096]u8`) used where an allocator is available? Replace them with
      dynamic allocations (`allocator.print`, `allocator.alloc`, `ArrayList`).
- [ ] **File & Line Limits (Componentize into Subfolders, Never Truncate)**: Is
      every file **<= 500 lines** and every line **<= 80 columns**? When a file
      exceeds 500 lines, never truncate its functionality or unit tests—refactor
      it into a dedicated subfolder (`vm/run.zig`, `panes/modal.zig`,
      `tests/gui.zig`) rather than flat `prefix_subname.zig` files, and verify
      every new submodule is imported inside the parent module's `test` block
      (`_ = @import("vm/run.zig");`).
- [ ] **Subfolders Over Prefix-Underscore Filenames**: Are related submodules
      organized into subdirectories (`vm/`, `panes/`, `gui/`, `icon/`, `tests/`)
      with concise filenames (`info.zig`, `modal.zig`, `api.zig`, `auth.zig`,
      `main.c`) rather than flat `prefix_subname.zig` (`vm_info.zig`,
      `panes_modal.zig`, `auth_cmd.zig`) or `diosix_`-prefixed files?
- [ ] **Zig 0.17 Compliance**: Are `@backingInt`, `@fromBackingInt`, `@splat`,
      `allocator.print`, `list.last()`, and `SafeAllocator` used consistently?
- [ ] **Mechanism vs. Policy and Overlay Purity**: Are policy-level settings
      and tunables defined in build-time (`build.zig` options /
      `boot/hypervisor.conf`) or
      run-time (`/etc/diosix/*.toml`, with single-line `key = "value"` pairs
      <= 80 cols) configuration files rather than hardcoded in Zig? Is
      `tools/overlay-common/` free of generated binaries or build artifacts?
      (Paths and file inventories are covered by the Hard-Coded-Path Smell
      below.)
- [ ] **No On-the-Fly Binary or Image Synthesis in Zig**: Does the code hand-
      craft ELF binaries, machine-code shims, or disk/media images word-by-word
      at runtime in Zig (e.g., `synthesizeMinimalRiscv64Elf`)? Replace runtime
      binary synthesis with real assembly/source files compiled at build time
      or dedicated image-building scripts.
- [ ] **No Legacy Candidate-List Accumulation (The Candidate-List Smell)**:
      Does the code introduce or expand multi-path search arrays (such as
      `DEFAULT_GUEST_CANDIDATES` or `GUEST_SHIM_CANDIDATES`) to avoid updating
      co-versioned internal scripts or modules? Reject historical path probing
      for in-tree artifacts: update all in-tree producers, consumers, and tests
      atomically to use a single canonical path, handle persistent disk layout
      migrations once at the boot/install boundary, and require a
      `// DEPRECATED(26.x): remove in 27.0` comment on any genuine external
      compatibility alias.
- [ ] **No Hard-Coded Paths or Build Inventory (The Hard-Coded-Path Smell)**:
      Is configuration embedded in code? Flag every one of these in
      `build.zig`, `scripts/*`, tests, and tools:
      - String literals naming repository paths
        (`"tools/guest-shim/src/main.zig"`, `"tools/guest-shim/linker.ld"`,
        `"hypervisor/core/metadata.zig"`, `"scripts/build_rootvm.sh"`) rather
        than values derived from one layout definition.
      - Hand-maintained arrays of source files or cache inputs
        (`ctl_sources`, `gui_sources`, `shim_sources`, `extra_rootvm_inputs`).
        These must be discovered from the directory tree or read from a
        declarative manifest, so adding, renaming, or splitting a file never
        needs a `build.zig` edit.
      - The same path or binary name repeated across `build.zig` and several
        scripts, or runtime absolute paths (`/etc/diosix`, `/var/lib/diosix`,
        `/run/diosix`, `/usr/share/diosix`) repeated as literals across tool
        modules instead of one `sysconfig.zig` accessor or constant.
      - **Acceptance test**: add an empty-but-valid `.zig` file to a tool
        source directory. If the build, cache hash, or test list needs any
        edit to notice it, report the hard-coded inventory as a finding.
- [ ] **No Silent Format Overflow or Guessed Buffer Sizes**: Is any
      `bufPrint(...) catch return`, `catch ""`, or `catch "--"` hiding a
      too-small buffer? Is every remaining fixed buffer a named constant with a
      comment justifying its size? Console output should use
      `common.printFmt` and stored strings `allocator.print`.
- [ ] **No Swallowed Errors**: Are all error unions handled explicitly via
      `try`, `catch`, or `errdefer` (no silent `_ = foo() catch {};`)?

## 2. The Paranoid Security Expert
*Assumes untrusted guests and malformed inputs are actively trying to break
isolation, corrupt memory, or cause denial of service.*

- [ ] **Blast-Radius Containment (Zero Guest-Induced Panics)**: Can any guest
      input, malformed hypercall, or invalid GPA trigger an M-mode `panic` or
      infinite loop? (Must return an SBI error or terminate only the offending
      guest.)
- [ ] **No Double-Fetch (TOCTOU) on Guest Memory**: Is data in guest-writable
      memory copied via `guest.copyFromGuest` into private hypervisor/parent
      memory **once** before validation and use?
- [ ] **M-Mode / Guest Privilege Boundary**: Does every SBI hypercall validate
      that GPAs and sizes lie strictly within the calling guest's owned memory
      (and `!physmem.isHypervisorMemory(hpa)`) before converting GPA to HPA?
- [ ] **Checked Arithmetic**: Are all size/offset calculations (`offset + len`,
      `count * stride`) using `std.math.add` / `std.math.mul` to prevent
      integer overflow wrap-around?
- [ ] **Secret & Page Zeroization**: Are freed guest physical pages zeroed and
      ephemeral keys/nonces wiped via `std.crypto.secureZero`?
- [ ] **IRQ-Safe Locking & Lock Ordering**: Do all spinlocks use the canonical
      IRQ-safe `atomic.Spinlock` (`const f = lock.lock();` /
      `defer lock.unlock(f);`), and is the global lock hierarchy respected
      without holding locks across `hw_run_vcore` or `wfi`?
- [ ] **Constitutional & Steward Invariants**: Can Root VM (`CID 1`) bypass a
      sealed Constitutional Charter, forge a Steward vote, kill a `STEWARD` VM,
      or map a `CONFIDENTIAL` guest's memory? (Must be impossible.)

## 3. The Diosix Maintainer
*Cares about long-term clarity, testability, and zero-friction debugging.*

- [ ] **Single Responsibility**: Does each file and function have one clear
      purpose, with branching pushed up to coordinators and loops pushed down
      to pure helpers?
- [ ] **Impeccable English Comments & Docs**: Are all comments and docs written
      in clear, grammatically correct American English with verified spelling,
      full sentences ending in a period, and explaining *why* rather than
      *what*?
- [ ] **Test Placement & Submodule Coverage**: Are unit tests co-located at the
      bottom of the `.zig` file they test (with
      `std.testing.refAllDecls(@This())` and `_ = @import("sub.zig");` for all
      submodules), and are multi-component integration tests in `tests/`?

## 4. The Beginner & Expert User
*Expects a responsive, intuitive, reliable system with zero toil.*

- [ ] **Actionable Error Messages**: Do CLI and GUI errors clearly state in
      plain English what went wrong and how to fix it (using
      `common.getExeName()` rather than a hardcoded binary name)?
- [ ] **Beginner + Expert Ergonomics**: Do interactive defaults work out of the
      box for beginners while supporting `--json` and `/etc/diosix/*.toml`
      overrides for experts?
- [ ] **Performance & Responsiveness**: Does the change avoid unnecessary
      allocations in hot loops, redundant MMIO writes, or frame-rate drops in
      `diosix-gui`?
