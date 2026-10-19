# Diosix agent skills

This repository provides Large Language Model (LLM) agent skills for developing,
documenting, and reviewing the Diosix hypervisor. These skills give coding
agents the project-specific context they need to follow Diosix architecture,
code, and documentation conventions.

Applying these skills helps your agent write memory-safe, idiomatic Zig and
RISC-V assembly code that matches the existing codebase and passes pre-merge
review with minimal rework.

## What's included

This repository includes three complementary skills that cover the full Diosix
contribution workflow: authoring code, updating documentation, and auditing
changes before you commit or merge them.

### developing-diosix

The [`developing-diosix`](developing-diosix/SKILL.md) skill defines the
engineering standards, Zig 0.17 idioms, and RISC-V assembly conventions for
writing Machine-mode (M-mode) hypervisor code, Supervisor-mode (S-mode)
emulation logic, and User-mode (U-mode) system utilities. It enforces a strict
priority order—safety, security, correctness, and maintainability—so that
generated code is self-documenting, memory-safe, and ready to pass review on
the first attempt.

*   **Complete, buildable implementations**: Require every change to compile
    and pass both the Root Virtual Machine (VM) hypervisor build and the host
    unit test suite (`./scripts/build.sh` and `./scripts/build.sh test`)
    without `TODO` stubs or mocked production paths.
*   **Modular file organization**: Cap source files at 500 lines and wrap lines
    at 80 columns, decomposing larger files into cohesive subdirectories
    (`vm/run.zig`, `panes/vm.zig`) with concise `snake_case` names rather than
    flat `prefix_subname.zig` files or truncated functionality.
*   **Idiomatic Zig 0.17 dispatch and types**: Parse commands and configuration
    keys with `std.meta.stringToEnum` and `switch`, use explicit fixed-width
    types (`u64`, `u32`, `u16`, `u8`) in Application Binary Interface (ABI)
    structs, write hexadecimal literals in lowercase, and replace magic
    bitmasks with named constants and helpers.
*   **Disciplined memory management**: Restrict static fixed-size buffers to
    pre-heap M-mode boot code, allocate buffers dynamically once an allocator
    is available, pair every allocation with `defer` or `errdefer`, and zeroize
    freed guest pages and cryptographic secrets.
*   **Enterprise hypervisor isolation**: Never panic in M-mode due to guest VM
    input or state; return a Supervisor Binary Interface (SBI) error or
    terminate the offending guest subtree, and copy shared guest memory once
    before validation to prevent double-fetch Time-of-Check to Time-of-Use
    (TOCTOU) vulnerabilities.
*   **Mechanism over hardcoded policy**: Keep tunable settings in build-time
    options (`build.zig`, `boot/hypervisor.conf`) or run-time configuration
    files (`/etc/diosix/*.toml`), and compile shims and media images at build
    time rather than synthesizing binaries on the fly in Zig.
*   **Clean atomic migrations**: Preserve external ABI and CLI backwards
    compatibility while updating co-versioned in-tree producers, consumers, and
    tests atomically without accumulating legacy fallback arrays.

### documenting-diosix

The [`documenting-diosix`](documenting-diosix/SKILL.md) skill governs the
creation and maintenance of technical documentation across `docs/` and
`README.md`. It ensures that documentation stays 100% synchronized with the
implementation, serves both system operators and core developers, and complies
with the Diosix style guide.

*   **Lockstep code and documentation updates**: Update `docs/` in the same
    change whenever code modifications alter SBI function IDs, `ioctl` numbers,
    C struct layouts, CLI flags, modularized file paths, or configuration
    schemas.
*   **Dual-audience design**: Tailor documentation explicitly for two distinct
    audiences: beginner and expert users launching and configuring guest VMs
    with `dsx`, and developers building, debugging, and extending the M-mode
    hypervisor, S-mode dynamic recompiler (`dynarec`), and U-mode tools.
*   **Canonical hardware and virtualization terminology**: Distinguish strictly
    between the host build machine, physical host hardware, target platform,
    Guest Physical Address (GPA), Host Physical Address (HPA), Guest Virtual
    Address (GVA), and tree-relative Context ID (CID).
*   **Strict prose and formatting standards**: Write in American English using
    active voice, present tense, sentence-case headings, and an 80-column line
    limit.
*   **Abbreviation and list discipline**: Spell out non-universal abbreviations
    on their first use per page, reserve numbered lists strictly for sequential
    multi-step procedures, and use bulleted lists for unordered items.

### reviewing-diosix

The [`reviewing-diosix`](reviewing-diosix/SKILL.md) skill provides a rigorous
pre-commit and pre-merge audit methodology that mirrors `developing-diosix` and
`documenting-diosix`. It prevents incomplete, insecure, or unverified changes
from entering the repository by enforcing a four-step verification workflow
before any change is committed or merged.

*   **Build, completeness, and test gate**: Verify that features are 100%
    implemented without `TODO` placeholders, confirm unit tests are co-located
    in source files while integration tests reside in `tests/`, and run
    `./scripts/build.sh` and `./scripts/build.sh test --summary all` with zero
    warnings or leaks.
*   **Four-persona deep review**: Audit every modified file through the lenses
    of a Skeptical Veteran Engineer (idioms, subdirectory modularity, and build
    hygiene), a Paranoid Security Expert (blast-radius containment, TOCTOU
    prevention, and secret zeroization), a Diosix Maintainer (readability, ABI
    assertions, and test coverage), and a Beginner/Expert User (ergonomics and
    clear error reporting).
*   **Rust-style borrow-checker and pointer lifetime audit**: Manually trace
    every pointer, slice, and allocation across all control-flow paths to
    guarantee zero Use-After-Free (UAF) bugs, double frees, pointer
    invalidation across resizes, or dangling stack references.
*   **Documentation and style gate**: Confirm that `docs/` matches all code
    changes, complies with the project style guide, and passes
    `zig fmt --check build.zig hypervisor tools scripts`.
*   **Escalation over assumption**: Stop and ask the lead maintainer for
    direction whenever an audit uncovers an architectural trade-off, ambiguous
    requirement, or missing subsystem primitive.

## Installing

You can install these skills in Google Antigravity or in any other coding agent
environment that supports the `SKILL.md` format.

### Google Antigravity

Google Antigravity finds skills in two ways. It discovers any skill directory
containing a `SKILL.md` file under `.agents/skills/` (or `.agent/skills/`) in
your Diosix workspace, and it loads any directory that you register in
`.agents/skills.json`. Either way, the Antigravity IDE, the `agy` CLI, and the
Antigravity desktop GUI pick up the skills. Antigravity scans one directory
level deep, so each skill directory must sit directly inside the scanned
directory.

Use one of the following methods to install the skills.

#### Clone into the skills directory (recommended)

Clone this repository directly into `.agents/skills/` at the root of your
Diosix workspace. Each skill directory in the repository root then sits
directly under `.agents/skills/`:

```shell
mkdir -p .agents
git clone https://github.com/diodesign/diosix-agent-skills.git .agents/skills
```

To update the skills, run `git -C .agents/skills pull`.

#### Clone elsewhere and symlink

Clone this repository outside your workspace, then symlink each skill into
`.agents/skills/`. Use this method to share one checkout of the skills across
several workspaces:

```shell
git clone https://github.com/diodesign/diosix-agent-skills.git \
    ~/src/diosix-agent-skills
mkdir -p .agents/skills
for skill in developing-diosix documenting-diosix reviewing-diosix; do
    ln -s ~/src/diosix-agent-skills/"$skill" .agents/skills/"$skill"
done
```

#### Clone elsewhere and register with skills.json

Instead of symlinking, clone this repository outside your workspace and
register its path in `.agents/skills.json` at the root of your Diosix
workspace. Antigravity scans the registered directory one level deep, so it
finds each skill directory in the repository root. Use this method to share one
checkout of the skills across several workspaces without creating symlinks:

```shell
git clone https://github.com/diodesign/diosix-agent-skills.git \
    ~/src/diosix-agent-skills
mkdir -p .agents
```

Then create `.agents/skills.json` with the absolute path to your clone:

```json
{
  "entries": [
    { "path": "/home/your-user/src/diosix-agent-skills" }
  ]
}
```

#### Verify the installation

Reload your skills, then confirm that Antigravity reports the three Diosix
skills. In the `agy` CLI, run `/skills reload`. The output lists
`developing-diosix`, `documenting-diosix`, and `reviewing-diosix` as added.

In the Antigravity GUI and IDE, you can also inspect active skills in the
**Skills & Customizations** panel. For more information on skills, see the
[Antigravity documentation](https://antigravity.google/docs) and the
[Antigravity agent skills guide](https://antigravity.google/docs/skills).

### Other agent environments

These skills follow the open `SKILL.md` directory format (`SKILL.md` with YAML
frontmatter and supporting files under `references/`). If you use another
coding agent environment, consult its documentation for instructions on where
to install or register custom skill directories.

## More information

For usage, copyright, and distribution of this repository's materials, see
[LICENSE](LICENSE).

For more information about Diosix, see the project's
[diosix.org website](https://diosix.org) and
[source code repository](https://github.com/diodesign/diosix).
