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
never sign off on broken, incomplete, or unverified code.

## Mandatory 4-Step Verification Workflow

1. **Build, Completeness & Test Gate (Table Stakes)**:
   - Verify the feature is 100% implemented as designed with zero `TODO` stubs,
     dead code, or mocked production paths.
   - Verify unit tests are co-located in the `.zig` source files they test and
     integration tests reside in separate `tests/` directories.
   - Run `./scripts/build.sh test --summary all` and `./scripts/build.sh` until
     100% of compilation and test steps pass with zero warnings or leaks.
2. **4-Persona Deep Review**:
   - Audit every modified file and its surrounding context through four
     distinct lenses:
     1. **Skeptical Veteran Engineer** (completeness, idioms, smells,
        500-line/80-col limits, dynamic vs. static buffers)
     2. **Paranoid Security Expert** (hypervisor blast-radius isolation,
        double-fetch/TOCTOU, IRQ-safe locking, crypto/page zeroization, bounds)
     3. **Diosix Maintainer** (modularity, readability, `defer`/`errdefer`,
        ABI assertions, test coverage)
     4. **Beginner & Expert User** (zero-toil UX, clear English errors, config
        flexibility, performance)
3. **Rust-Style Borrow-Checker & Pointer Lifetime Audit**:
   - Manually trace every pointer, slice, and allocation across all control
     flow paths to guarantee zero Use-After-Free (UAF), double-free, pointer
     invalidation on resize/remap, or dangling stack references.
4. **Documentation, Spelling & Style Gate**:
   - Verify `docs/` matches all code changes, is written in clear,
     grammatically correct English with verified spelling, and complies with
     `docs/style-guide.md` and `docs/development.md`.
   - Re-run `zig fmt --check` and `./scripts/build.sh test --summary all` after
     any review fixes.
5. **When Unsure, Stop and Ask the Lead Maintainer**: If a review uncovers an
   architectural trade-off or missing primitive (for instance, whether to add a
   dynamic formatting helper to M-mode's allocator or use a different mechanism),
   stop and ask the lead maintainer for direction before making assumptions.

## Focused Reference Guides

Consult these reference checklists during every review:

- **[The 4 Review Personas Checklist](./references/review-personas.md)**:
  Detailed inspection criteria for the Skeptical Veteran Engineer, Paranoid
  Security Expert, Diosix Maintainer, and Beginner/Expert User.
- **[Borrow-Checker & Pointer Lifetime Audit](./references/borrow-checker-and-memory-audit.md)**:
  Step-by-step methodology for tracing pointer provenance, lifetimes, aliasing
  mutability, and cross-privilege memory safety in Zig.
- **[Pre-Commit & Pre-Merge Gates](./references/pre-commit-and-merge-gates.md)**:
  Concrete verification commands and sign-off criteria required before
  committing or merging to `devel` or `stable`.
