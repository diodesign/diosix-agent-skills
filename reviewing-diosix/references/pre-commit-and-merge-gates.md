# Pre-Commit & Pre-Merge Gates

Before committing or merging any branch into `devel` or `stable`, verify that
every gate below passes with zero errors.

## Gate 1: Completeness, Formatting & Structural Limits

1. **No Stubs, Partial Implementations, or Dead Code**:
   - Confirm the feature or fix works end-to-end as designed with zero `TODO`
     placeholders, dead code, or mocked production paths.
   - **Dead-Code & Orphaned-Test Audit**: For every function, global variable
     (`var`/`const`), type, or struct field that was superseded, refactored, or
     left behind in modified files, verify it has at least one **non-test**
     reference across `hypervisor/`, `tools/`, `build.zig`, and `scripts/`.
     Because Zig does not warn on unused top-level declarations and
     `std.testing.refAllDecls(@This())` masks unused `pub` items, explicitly
     check that no symbol is kept alive solely by its own `test` block. Delete
     any unused function, global variable, or constant along with its unit
     tests.
2. **Zig Formatter & Line/File Audit (Scoped to Tracked Directories)**:
   > **Warning**: Never run unscoped `zig fmt .` or Python
   > `glob.glob("**/*.zig", recursive=True)` from the repository root, because
   > `.zig-cache/` and `zig-out/buildroot-*` contain hundreds of thousands of
   > files and symlink loops. Always scope to
   > `build.zig hypervisor tools scripts`:
   ```bash
   zig fmt --check build.zig hypervisor tools scripts && python3 -c '
   import subprocess
   dirs = ["build.zig", "hypervisor", "tools", "scripts"]
   cmd = ["find", *dirs, "-name", "*.zig"]
   files = sorted(subprocess.check_output(cmd).decode().splitlines())
   over_500, over_80 = [], []
   for f in files:
       lines = open(f).read().splitlines()
       if len(lines) > 500: over_500.append((f, len(lines)))
       for i, l in enumerate(lines, 1):
           if len(l) > 80: over_80.append((f, i, len(l)))
   print(f"Checked {len(files)}: {len(over_500)} >500L, {len(over_80)} >80C")
   assert not over_500 and not over_80
   '
   ```
3. **Subfolder Hierarchy, Filename & Overlay Audit**:
   - Confirm related submodules are organized in subdirectories (`vm/run.zig`,
     `panes/modal.zig`, `tests/gui.zig`) rather than flat `prefix_subname.zig`
     files (`vm_run.zig`, `panes_modal.zig`, `tests_gui.zig`), and that
     filenames use concise `snake_case` without redundant `diosix_` or `_cmd`
     affixes.
   - Confirm `tools/overlay-common/` contains only static configuration files
     (`etc/`) and no generated `boot/`, `usr/`, or `root/` build outputs.

## Gate 2: Co-located Unit Tests & Leak Detection

1. **Test Placement & Submodule Wiring**:
   - Unit tests must live inside the `.zig` source file they test, using
     `std.testing.refAllDecls(@This())` and `_ = @import("sub.zig");` for any
     extracted submodules to force semantic analysis and test execution across
     all declarations.
   - Allocator-using unit tests must use `std.heap.SafeAllocator` (or verify
     `heap.inuse_size == 0` for `HeapAllocator`) to guarantee zero memory leaks.
2. **Execute Host-Native Unit Test Suite**:
   ```bash
   ./scripts/build.sh test --summary all
   ```
   All test targets (`diosix-unit-tests`, `diosix-ctl-unit-tests`,
   `diosix-interface-unit-tests`, `diosix-emulation-unit-tests`,
   `diosix-gui-unit-tests`, `diosix-config-unit-tests`,
   `diosix-guest-shim-unit-tests`) must pass 100%.

## Gate 3: Full-Stack Build & Integration Tests

1. **Integration Test Placement**:
   - Multi-component and end-to-end integration tests must reside in dedicated
     `tests/` directories (and `scripts/test_manifest_integration.py`).
2. **Full Root VM & Hypervisor Build**:
   ```bash
   ./scripts/build.sh
   ```
   Confirm adding, renaming, or splitting a source file required **no edit** to
   `build.zig` or `scripts/build_rootvm.sh`: sources and cache inputs are
   discovered or read from a layout manifest, never from hand-maintained
   arrays. A new file that must be listed by hand in `build.zig`, a script, or
   a test list is a failing finding (the Hard-Coded-Path Smell). Confirm the
   Root VM cache hash still covers the new file's directory, and that `--dep`
   and `-M` module flags still match. Run this audit and expect no matches
   outside the single layout definition:
   ```bash
   grep -nE '"(hypervisor|tools|scripts|boot|docs)/[^"]+"' build.zig
   grep -nE '(_sources|_inputs) *= *\[' build.zig
   ```
3. **QEMU Integration Suite (for SBI, `/dev/diosix`, manifests, or CLI)**:
   ```bash
   ./scripts/test_manifest_integration.py
   ```

## Gate 4: Documentation, English Prose & Style Guide Synchronization

1. Inspect `README.md`, `docs/style-guide.md`, `docs/development.md`,
   `docs/cli.md`, `docs/architecture.md`, `docs/gui-architecture.md`,
   `docs/interface.md`, `docs/release-architecture.md`, `docs/run.md`, and
   `docs/security.md`.
2. Verify all modified CLI commands, TOML configuration keys, SBI hypercalls,
   modularized file paths, and testing conventions are accurately documented in
   clear, grammatically correct American English with verified spelling,
   wrapped at <= 80 characters, using active voice, present tense,
   sentence-case headings, and first-use abbreviation expansion.
