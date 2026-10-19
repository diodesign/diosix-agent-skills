# Architecture, Modularity & TigerStyle

Distilled from Diosix engineering standards, enterprise Type-1 hypervisor
engineering, and TigerStyle (**Safety > Performance > Developer Experience**).

## 1. Complete, Working, Buildable Implementation (Table Stakes)

- **No Partial or Broken Code**: Every feature or bug fix must be implemented
  completely end-to-end and must compile and pass all tests before sign-off.
- **No Stubs or Silent No-Ops**: Never commit `TODO` placeholders, unimplemented
  branches that silently succeed, or hardcoded mock returns in production code
  paths.
- **English Prose in Comments**: Write all comments and docstrings in clear,
  grammatically correct English with verified spelling. Start every comment
  sentence with a capital letter and end with a period. Explain *why* a design
  decision or invariant exists, not *what* the syntax already says.

## 2. File Size, Subfolder Decomposition & Naming

- **<= 80 Columns**: Wrap all source code, comments, and multiline strings at
  80 characters. Always run `zig fmt` before committing.
- **<= 500 Lines per File (Componentize into Subfolders, Never Truncate)**:
  Keep files focused on a single domain concept and under 500 lines unless
  there is a compelling architectural reason. When refactoring a file over 500
  lines, never truncate its features or unit tests—decompose it into smaller
  building blocks inside a dedicated subfolder.
- **Subfolders Over `prefix_subname.zig` Flat Files**:
  Never create clusters of flat files sharing a `prefix_` in the same directory
  (such as `vm_info.zig`, `vm_run.zig`, `panes_modal.zig`, or `tests_gui.zig`).
  Instead, create a subfolder named after the domain (`vm/`, `panes/`, `gui/`,
  `icon/`, `window/`, `tests/`) and give each file inside the subfolder a
  concise, unprefixed name:
  - **Good**:
    - `tools/diosix-ctl/src/vm.zig` + `tools/diosix-ctl/src/vm/{info,install,run,stage}.zig`
    - `tools/diosix-gui/src/panes.zig` + `tools/diosix-gui/src/panes/{catalog,modal,vm}.zig`
    - `tools/diosix-gui/src/tests.zig` + `tools/diosix-gui/src/tests/{gui,nav,panes,visual}.zig`
    - `hypervisor/core/sbi.zig` + `hypervisor/core/sbi/{charter,diosix,ipc,standard}.zig`
  - **Bad**:
    - `vm_info.zig`, `vm_install.zig`, `vm_run.zig`, `vm_stage.zig`
    - `panes_catalog.zig`, `panes_modal.zig`, `panes_vm.zig`
    - `tests_gui.zig`, `tests_nav.zig`, `tests_panes.zig`, `tests_visual.zig`
- **How to Split a Module into a Subfolder Cleanly in Zig**:
  1. Keep the primary facade/coordinator in `foo.zig` (or `foo/mod.zig`) and
     place cohesive submodules in `foo/bar.zig` and `foo/baz.zig`.
  2. Inside `foo/bar.zig`, reference parent-directory modules via
     `@import("../other.zig")` and sibling submodules via `@import("baz.zig")`.
  3. Re-export public helpers in `foo.zig` (`pub const doBar = bar.doBar;`) so
     external callers simply `@import("foo.zig")`.
  4. **Wire Submodule Unit Tests**: Because `zig build test` only executes
     `test` blocks in files transitively referenced from the root test file,
     always reference every submodule in the parent module's `test` block:
     ```zig
     test {
         std.testing.refAllDecls(@This());
         _ = @import("foo/bar.zig");
         _ = @import("foo/baz.zig");
     }
     ```
- **Zero Copy-Paste Primitives**: Never duplicate foundational types or helpers
  across files:
  - Import `Spinlock` from `hypervisor/core/atomic.zig` (never redefine a local
    spinlock without M-mode interrupt save/restore).
  - Import `PAGE_SIZE`, `alignUp`, `alignDown`, `elf`, `riscv`, and `sbi` from
    `hypervisor/interface/`.
  - Place shared CLI/GUI validators (such as `isValidSafeName` or JSON
    escaping) in `common.zig`.
- **Non-Redundant Filenames**: Use concise `snake_case` names. Never repeat
  `diosix`, parent folder names, or redundant `_cmd` suffixes in filenames:
  - Good: `tools/diosix-ctl/src/api.zig`, `tools/diosix-ctl/src/auth.zig`,
    `tools/alsa-init/main.c`
  - Bad: `tools/diosix-ctl/src/diosix_api.zig`,
    `tools/diosix-ctl/src/auth_cmd.zig`, `tools/alsa-init/diosix-alsa-init.c`
- **Identifier Casing**:
  - Types, structs, enums, unions, type-returning functions: `PascalCase`
  - Functions and methods: `camelCase`
  - Local variables and struct fields: `snake_case`
  - Constants: `SCREAMING_SNAKE_CASE` or `snake_case`
  - Append units/qualifiers in descending order of significance:
    `ram_bytes_max`, `latency_ms_min`, `timeout_ms`.

## 3. Control Flow & Assertions

- **Push `if`s Up and `for`s Down**: Centralize branching (`switch` / `if`) in
  parent coordinator functions; keep leaf helper functions pure and branch-free.
- **Positive Invariants**: State conditions positively (`if (idx < len)` rather
  than `if (idx >= len)`).
- **Bounded Loops**: Every loop and queue must have an explicit fixed upper
  bound (`MAX_GUESTS`, `MAX_MANIFEST_SIZE`, `MAX_VCPUS`) to prevent infinite
  loops or denial-of-service in M-mode or U-mode.
- **Split & Pair Assertions**:
  - Assert preconditions at function entry and postconditions before return.
  - Write `assert(a); assert(b);` instead of `assert(a and b);`.
  - Use `comptime assert(@sizeOf(T) == ...)` and `@offsetOf` on ABI, ELF, FDT,
    and ioctl structs.
- **Never Swallow Errors**: Never write `_ = fallibleCall() catch {};` without
  explicit justification. Propagate errors with `try` or handle every error tag
  in an explicit `switch (err)`.

## 4. Configuration Externalization & Overlay Hygiene

- **Mechanism in Code, Policy in Build/Run-Time Config**: Code must focus on
  implementation (mechanism); policy-level settings and tunables belong in
  build-time (`build.zig` options / `boot/hypervisor.conf`) or run-time
  (`/etc/diosix/*.toml`: `diosix.toml`, `catalog.toml`, `charter.toml`,
  `install.toml`, `system.toml`) configuration files.
- **Single-Line TOML Key-Value Pairs**: Lightweight U-mode TOML parsers
  (`sysconfig.zig`, `charter.zig`) parse `key = "value"` line-by-line. In
  `/etc/diosix/*.toml`, wrap `#` comments at <= 80 columns and keep
  `key = "value"` strings concise on a single line <= 80 columns rather than
  using multiline `"""` strings.
- **Never Synthesize Binaries or Media Images on the Fly in Zig**: Do not craft
  ELF files, machine-code shims, or media/disk images word-by-word in Zig at
  runtime (e.g., `synthesizeMinimalRiscv64Elf`). Maintain shims as real
  assembly/source files built during `zig build`, and use dedicated scripts for
  building media images.
- **Keep `tools/overlay-common/` Pure Static Config**: Buildroot's
  `BR2_ROOTFS_OVERLAY` copies every file inside `tools/overlay-common/` into the
  Root VM rootfs regardless of `.gitignore`. Never output compiled binaries,
  generated `.S` files, or temporary artifacts into `tools/overlay-common/`;
  stage generated files exclusively in `.zig-cache/overlay-dynamic/`.
- Ensure shipped configuration files in `tools/overlay-common/etc/diosix/`
  carry least-privilege file permissions (`0644` for public read-only configs,
  `0600` for sensitive key/state files).

## 5. Testing Architecture

- **Inline Unit Tests**: Every `.zig` source file must include `test` blocks at
  the bottom of the same file testing its pure functions, parsers, formatters,
  and edge cases.
- **Force Full Semantic Analysis**: Include `std.testing.refAllDecls(@This());`
  in module test blocks so the Zig compiler type-checks every public function
  even if not directly called by a test case.
- **Separate Integration Tests**: Place multi-component and end-to-end tests in
  dedicated `tests/` folders (and `scripts/test_manifest_integration.py`).
