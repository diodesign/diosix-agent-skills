---
name: documenting-diosix
description: >-
  Rules, style standards, and audience guidelines for writing and updating
  Diosix documentation in docs/ and README.md. Activate this skill whenever
  writing new documentation, updating CLI or architecture docs after code
  changes, or auditing documentation against docs/style-guide.md.
---

# Documenting Diosix

Every documentation page in `docs/` and `README.md` must stay 100% synchronized
with the implementation in `hypervisor/`, `tools/`, and `scripts/`, and must
comply with `docs/style-guide.md` so documentation passes `reviewing-diosix` on
the first pass.

## Core Mandates

1. **Clear, Correct English Prose (Table Stakes)**: All documentation must be
   written in American English with verified spelling, sound grammar, and
   concise wording that never sacrifices technical clarity or precision.
2. **Truth in Documentation**: Never leave stale CLI flags, SBI function IDs,
   ioctl numbers, C struct layouts (`uint64_t` vs `unsigned long`), file paths,
   or configuration keys in `docs/`. Every code change that alters behavior,
   CLI options, modular file paths, or config schemas must update `docs/` in
   the same change.
3. **Dual-Audience Design**: Cater explicitly to two distinct audiences:
   - **Users (Beginners & Experts)**: Operators booting Diosix, running the
     guided tour, launching guest VMs with `dsx`, or configuring declarative
     system/charter manifests.
   - **Developers & Maintainers**: Engineers building, testing, debugging, and
     extending the M-mode hypervisor, S-mode `dynarec`, and U-mode tools.
4. **Strict Style Compliance (`docs/style-guide.md`)**:
   - Wrap prose, list items, and code blocks at **<= 80 characters** per line
     (excluding markdown table rows and long URLs). Use file basenames in
     markdown link text (`[`S50dropbear`](../tools/.../S50dropbear)`) so link
     lines wrap cleanly within 80 columns.
   - Use **active voice** and **present tense** ("The hypervisor validates the
     GPA," never "The GPA will be validated by the hypervisor").
   - Use **sentence-case headings** (`## Hierarchical guest model`, never
     `## Hierarchical Guest Model`).
   - Spell out abbreviations on **first use per page** (`Supervisor Binary
     Interface (SBI)`), then use only the abbreviation (`SBI`) thereafter.
   - Use numbered lists **strictly** for sequential multi-step procedures and
     bulleted lists for non-sequential items.
5. **When Unsure, Stop and Ask the Lead Maintainer**: If intended behavior,
   terminology, or audience scope is ambiguous, stop and ask the lead
   maintainer for clarification rather than guessing or inventing details.

## Focused Reference Guides

Read the relevant reference file before drafting or editing documentation:

- **[Style & Canonical Terminology](./references/style-and-terminology.md)**:
  English grammar/spelling standards, 80-column wrapping, abbreviation
  expansion rules, and canonical hardware/virtualization definitions
  (`Host build machine`, `Host hardware`, `Target platform`, `HPA`, `GPA`,
  `GVA`, `CID`).
- **[Dual Audiences & Document Map](./references/audiences-and-structure.md)**:
  Tailoring content for beginner/expert users vs. maintainers, mapping code
  changes to specific `docs/*.md` files, and documenting testing conventions.
