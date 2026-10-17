# Pre-Commit & Pre-Merge Gates

Before committing or merging any branch into `devel` or `stable`, verify that
every gate below passes with zero errors.

## Gate 1: Completeness, Formatting & Structural Limits

1. **No Stubs or Partial Implementations**:
   - Confirm the feature or fix works end-to-end as designed with zero `TODO`
     placeholders, dead code, or mocked production paths.
2. **Zig Formatter**:
   ```bash
   zig fmt --check hypervisor/ tools/ build.zig
   ```
3. **Line Width & File Length Audit**:
   - Confirm modified lines are **<= 80 characters**.
   - Confirm new or refactored source files are **<= 500 lines** (or split into
     cohesive submodules if exceeding 500 lines).
   - Confirm filenames use concise `snake_case` without redundant `diosix_`
     prefixes.

## Gate 2: Co-located Unit Tests & Leak Detection

1. **Test Placement**:
   - Unit tests must live inside the `.zig` source file they test, using
     `std.testing.refAllDecls(@This())` to force semantic analysis of all
     public declarations.
   - Allocator-using unit tests must use `std.heap.SafeAllocator` (or verify
     `heap.inuse_size == 0` for `HeapAllocator`) to guarantee zero memory leaks.
2. **Execute Host-Native Unit Test Suite**:
   ```bash
   ./scripts/build.sh test --summary all
   ```
   All test targets (`diosix-unit-tests`, `diosix-ctl-unit-tests`,
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
   Confirm `build.zig` (`addFileInput`) and `scripts/build_rootvm.sh` include
   all new or renamed source and configuration files.
3. **QEMU Integration Suite (when touching SBI, `/dev/diosix`, manifests, or CLI)**:
   ```bash
   ./scripts/test_manifest_integration.py
   ```

## Gate 4: Documentation, English Prose & Style Guide Synchronization

1. Inspect `docs/style-guide.md`, `docs/development.md`, `docs/cli.md`,
   `docs/architecture.md`, `docs/interface.md`, and `docs/security.md`.
2. Verify all modified CLI commands, TOML configuration keys, SBI hypercalls,
   and testing conventions are accurately documented in clear, grammatically
   correct English with verified spelling, wrapped at <= 80 characters, using
   active voice, present tense, sentence-case headings, and first-use
   abbreviation expansion.
