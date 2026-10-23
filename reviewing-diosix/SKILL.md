---
name: reviewing-diosix
description: >-
  Pre-commit and pre-merge code and documentation review methodology for
  Diosix. Activate this skill before committing, merging, or signing off on
  any feature, refactoring, bug fix, or documentation update, or when the user
  asks for a code/security/architectural review.
---

# Reviewing Diosix

Every code and documentation change in Diosix must pass a rigorous,
multi-perspective audit before commit or merge. Never rubber-stamp changes, and
never sign off on broken, incomplete, or unverified code. The review criteria
below mirror `developing-diosix` and `documenting-diosix` 1:1 so that applying
the authoring skills eliminates pre-commit and pre-merge rework.

## Mandatory 4-Step Verification Workflow

1. **Build, Completeness & Test Gate (Table Stakes)**:
   - Verify the feature is 100% implemented as designed with zero `TODO` stubs,
     dead code, or mocked production paths.
   - **Surgically identify and remove dead code and its unit tests**: Zig does
     not warn on unused top-level functions, globals (`var`/`const`), or types,
     and `std.testing.refAllDecls(@This())` or an old `test` block can make
     unused production code appear referenced. For every replaced, refactored,
     or suspicious symbol in touched modules, search across `hypervisor/`,
     `tools/`, `build.zig`, and `scripts/` to confirm it has at least one
     non-test production caller. If a function, global variable, constant, or
     type is unused in production (or kept alive solely by its unit tests),
     delete it and its corresponding unit tests immediately.
   - When refactoring files over the 500-line limit, **never** truncate
     live functionality or unit tests; always componentize oversized files into
     cohesive subfolders (`vm/run.zig`, `panes/modal.zig`, `tests/gui.zig`)
     rather than flat `prefix_subname.zig` files, and verify every submodule is
     referenced in its parent's `test` block (`_ = @import("vm/run.zig");`) so
     `zig build test` executes its unit tests.
   - Verify unit tests are co-located in the `.zig` source files they test and
     integration tests reside in separate `tests/` directories.
   - Scope file-format and line/column checks to tracked source directories
     (`build.zig`, `hypervisor`, `tools`, `scripts`) rather than running
     unscoped recursive globs across `.zig-cache/` or `zig-out/`.
   - Run `./scripts/build.sh test --summary all` and `./scripts/build.sh` until
     100% of compilation and test steps pass with zero warnings or leaks.
2. **4-Persona Deep Review**:
   - Audit every modified file and its surrounding context through four
     distinct lenses:
     1. **Skeptical Veteran Engineer** (completeness, idioms, surgical removal
        of unused functions/globals/constants and their orphaned unit tests,
        subfolder hierarchy instead of flat `prefix_subname.zig` files,
        lowercase hex literals `0xfff`, named constants/helpers like
        `PAGE_OFFSET_MASK` and `crossesPageBoundary` instead of magic bitmasks,
        zero copy-paste duplication of `Spinlock`/alignment/path validators, no
        historical candidate-list fallback arrays like
        `DEFAULT_GUEST_CANDIDATES` for co-versioned internal paths, explicit
        fixed-width `u64` ABI types instead of `usize`, explicit `usize` type
        annotations on `@min`/`@max` with literals, componentizing >500-line
        files without truncating live code, <=80-col limit, dynamic vs. static
        buffers, externalizing policy to build/run-time config, no hard-coded
        repository paths, binary names, or hand-maintained source-file arrays
        in `build.zig`/scripts (the Hard-Coded-Path Smell: code is mechanism,
        configuration lives outside it), no on-the-fly binary/ELF/image
        synthesis in Zig, no generated files inside `tools/overlay-common/`)
     2. **Paranoid Security Expert** (hypervisor blast-radius isolation,
        single-copy double-fetch/TOCTOU prevention on guest memory, IRQ-safe
        locking via `atomic.Spinlock`, crypto/page zeroization, checked
        arithmetic bounds)
     3. **Diosix Maintainer** (modularity, readability, `defer`/`errdefer`,
        ABI `comptime` size/offset assertions, external-contract backwards
        compatibility vs. atomic in-tree migration, submodule test coverage)
     4. **Beginner & Expert User** (zero-toil UX, clear English errors, config
        flexibility, performance)
3. **Rust-Style Borrow-Checker & Pointer Lifetime Audit**:
   - Manually trace every pointer, slice, and allocation across all control
     flow paths to guarantee zero Use-After-Free (UAF), double-free, pointer
     invalidation on resize/remap, or dangling stack references.
4. **Documentation, Spelling & Style Gate**:
   - Verify `docs/` matches all code changes (including SBI FIDs, C struct
     types, CLI flags, and modularized file paths), is written in clear,
     grammatically correct American English with verified spelling, and
     complies with `docs/style-guide.md` and `docs/development.md`.
   - Re-run `zig fmt --check build.zig hypervisor tools scripts` and
     `./scripts/build.sh test --summary all` after any review fixes.
5. **When Unsure, Stop and Ask the Lead Maintainer**: If a review uncovers an
   architectural trade-off or missing primitive, stop and ask the lead
   maintainer for direction before making assumptions.

## Focused Reference Guides

Consult these reference checklists during every review:

- **[The 4 Review Personas Checklist](./references/review-personas.md)**:
  Detailed inspection criteria for the Skeptical Veteran Engineer, Paranoid
  Security Expert, Diosix Maintainer, and Beginner/Expert User.
- **[Borrow-Checker Audit](./references/borrow-checker-and-memory-audit.md)**:
  Step-by-step methodology for tracing pointer provenance, lifetimes, aliasing
  mutability, and cross-privilege memory safety in Zig.
- **[Pre-Commit Gates](./references/pre-commit-and-merge-gates.md)**:
  Concrete verification commands and sign-off criteria required before
  committing or merging to `devel` or `stable`.
