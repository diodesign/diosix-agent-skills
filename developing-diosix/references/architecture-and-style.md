# Architecture, Modularity & TigerStyle

Distilled from Diosix engineering standards, enterprise Type-1 hypervisor
engineering, and TigerStyle (**Safety > Performance > Developer Experience**).

## 1. Complete, Working, Buildable Implementation (Table Stakes)

- **No Partial or Broken Code**: Every feature or bug fix must be implemented
  completely end-to-end and must compile and pass all tests before sign-off.
- **No Stubs or Silent No-Ops**: Never commit `TODO` placeholders, unimplemented
  branches that silently succeed, or hardcoded mock returns in production code
  paths.
- **Surgically Remove Dead Code & Orphaned Unit Tests**: Zig does not warn on
  unused top-level declarations (`fn`, `pub fn`, `var`, `const`), and
  `std.testing.refAllDecls(@This())` or an old `test` block will keep an unused
  function or global variable compiling even after its last production caller is
  gone. Whenever you replace a helper, refactor a module, or simplify a flow,
  search across `hypervisor/`, `tools/`, `build.zig`, and `scripts/` to verify
  whether the old functions, globals, constants, or types still have any
  non-test callers. If not, delete them and their unit tests immediately.
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

## 4. Eliminating Magic Numbers & Unnamed Bitmasks

Never scatter unnamed numeric or hexadecimal literals (`0xfff`, `0xffc`, `0xe`,
`13`, `15`, `1 << 18`) in expressions. Apply these two smell tests and
category-specific replacements:

1. **The "Explanatory Comment" Smell**: If a literal requires an inline comment
   to explain what it means (`return 15; // Store page fault.` or
   `flags & (1 << 4) // User`), the literal is a magic number. Replace it with a
   named constant or `enum` tag (`TRAP_STORE_PAGE_FAULT`, `PTE.U`).
2. **The "Pre-Computed / Derived Mask" Smell**: Never write pre-computed derived
   hex masks like `if ((vaddr & 0xfff) > 0xffc)` or `if ((pte & 0xe) == 0)`.
   Pre-computing `4096 - 4` as `0xffc` erases the relationship between the page
   size and the access width and makes a page-boundary straddle check look like
   a range check on the first 4 KiB of memory. Express derived values via named
   constants and `@sizeOf` (`PAGE_SIZE - @sizeOf(u32)`, `PTE.R | PTE.W | PTE.X`)
   and encapsulate multi-part bit checks in inline predicate helpers
   (`crossesPageBoundary(vaddr, @sizeOf(u32))`, `PTE.isValid(pte)`).
3. **How to Replace Magic Numbers by Category**:
   - **Page Geometry & Alignment (`0xfff`, `0xffc`, `12`)**: Use `PAGE_SHIFT`,
     `PAGE_SIZE = 1 << PAGE_SHIFT`, `PAGE_OFFSET_MASK = PAGE_SIZE - 1`, and
     inline helpers (`crossesPageBoundary`, `alignUp`, `alignDown`).
   - **Hardware Bitfields & PTE Flags (`1 << 18`, `0xe`, `0x1f`)**: Group into
     namespace structs (`MSTATUS.SUM`, `PTE.V`, `PTE.RWX`) or `packed struct`
     types with inline predicates (`PTE.isPointer(pte)`).
   - **Exception Causes, CSRs & Protocol IDs (`13`, `15`, `0x14d`)**: Use typed
     `enum` values from `hypervisor/interface/`
     (`@backingInt(riscv.Cause.load_page_fault)`, `riscv.CSR.STIMECMP`).
   - **Sentinel Values (`0xffff_ffff`)**: Define named sentinels
     (`INVALID_VADDR_PAGE`, `TIMER_INFINITY`) or use Zig optionals (`?u32`).
   - **Lowercase Hexadecimal**: Always write hex literals in lowercase (`0xfff`,
     `0x8000_0000`, `0xffff_ffff`, never `0xFFF`).

## 5. Configuration Externalization & Overlay Hygiene

- **Mechanism in Code, Policy in Build/Run-Time Config**: Code must focus on
  implementation (mechanism); policy-level settings and tunables belong in
  build-time (`build.zig` options / `boot/hypervisor.conf`) or run-time
  (`/etc/diosix/*.toml`: `diosix.toml`, `catalog.toml`, `charter.toml`,
  `install.toml`, `system.toml`) configuration files.
- **Configuration Lives Outside the Code (Build Graph Included)**: The build
  script is code, so it must be the *mechanism* that turns a description of
  the repository into build steps. It must not *be* that description.
  - **The Hard-Coded-Path Smell**: A string literal naming a repository path
    (`"tools/guest-shim/linker.ld"`, `"hypervisor/core/metadata.zig"`,
    `"scripts/build_rootvm.sh"`), a binary or install name, or an array of
    source files (`ctl_sources`, `gui_sources`, `extra_rootvm_inputs`) inside
    `build.zig`, a shell script, or a Python test is configuration embedded in
    code. It rots silently: every refactor (such as splitting a 500-line file)
    forces a matching edit, and a missed edit produces a stale cache hash or a
    missing input rather than a compile error.
  - **Test for the smell**: Create a new `.zig` file inside
    `tools/diosix-ctl/src/` and run `zig build test`. If `build.zig`, a script,
    or a test list must change for the new file to be built, hashed, or
    tested, the layout is hard-coded.
  - **Fix pattern**:
    1. Define each repository location once, as data: a small layout file
       (for example `build/layout.zon`) or one `Layout` struct at the top of
       `build.zig` that names every root (`hypervisor_root`, `tools_root`,
       `guest_shim_dir`, `linker_script`), and derive every other path from
       those names with `b.path` / `std.fs.path.join`.
    2. Derive file lists by directory discovery (walk the source root at
       configure time, filter by extension, sort for determinism) instead of
       enumerating files. Feed the same discovered list to `addFileInput` so
       the Root VM cache hash tracks real inputs automatically.
    3. Pass tunables and per-target choices as `b.option` values or read them
       from `boot/*.conf` / `/etc/diosix/*.toml`.
    4. Keep scripts parameterized by arguments or environment instead of
       repeating the same path in several scripts.
  - **Runtime paths follow the same rule**: Tools resolve `/etc/diosix`,
    `/var/lib/diosix`, `/run/diosix`, `/usr/share/diosix`, and device nodes
    through `sysconfig.zig` accessors or one named constants block. They are
    never repeated as string literals across modules.
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

## 6. Backwards Compatibility vs. Legacy Accumulation

Maintain strict backwards compatibility for external contracts without letting
defensive fallback lists (`DEFAULT_GUEST_CANDIDATES`, `GUEST_SHIM_CANDIDATES`)
accumulate across internal modules:

1. **External Contracts vs. Co-Versioned Internal Plumbing**:
   - **External Contracts (Protect Compatibility)**: The SBI hypercall ABI
     (`hypervisor/interface/sbi.zig`), Linux driver ioctls
     (`tools/driver/diosix.h`), documented `dsx` CLI commands/flags/JSON
     schemas, and user-authored configuration files (`diosix.toml`,
     `manifest.toml`).
   - **Co-Versioned Internal Plumbing (Update Atomically, Never Accumulate)**:
     Internal Root VM filesystem paths, build scripts, init scripts, and test
     harnesses (`scripts/build_rootvm.sh`, `S50diosix`, `loader.zig`,
     `scripts/test_manifest_integration.py`) that are built and shipped
     together from the same repository. When changing an internal path or
     symbol, grep and update all in-tree producers, consumers, and tests in the
     same commit instead of appending to a fallback search list.
2. **Migrate Once at the Boundary, Never Probe Everywhere at Runtime**:
   - Runtime code must consult a single canonical path (resolved via
     `sysconfig.zig` or a single constant) rather than probing historical
     directories (`/boot`, `/var/lib/diosix`, `/usr/share/diosix`).
   - If persistent user storage (`/var/lib/diosix` on `/dev/vda`) may contain
     state from an older release, migrate or symlink that state once at the
     boot/upgrade boundary (`S50diosix` or `dsx host install`).
3. **The "Candidate-List Smell"**:
   - Multi-entry candidate arrays are only valid when selecting between
     distinct architectural tiers with different semantics (for example,
     user-installed `default.elf` falling back to the built-in recovery
     `micro-guest.elf`). Never use candidate arrays to paper over inconsistent
     internal paths.
4. **Sunset Tags for External Compatibility Shims**:
   - When an external compatibility alias is required (such as a deprecated CLI
     flag or TOML key), isolate it in the parser layer, translate it
     immediately to the canonical internal representation, and annotate it with
     `// DEPRECATED(26.x): remove in 27.0 — use <canonical>.`

## 7. Testing Architecture

- **Inline Unit Tests**: Every `.zig` source file must include `test` blocks at
  the bottom of the same file testing its pure functions, parsers, formatters,
  and edge cases.
- **Force Full Semantic Analysis**: Include `std.testing.refAllDecls(@This());`
  in module test blocks so the Zig compiler type-checks every public function
  even if not directly called by a test case.
- **Separate Integration Tests**: Place multi-component and end-to-end tests in
  dedicated `tests/` folders (and `scripts/test_manifest_integration.py`).
