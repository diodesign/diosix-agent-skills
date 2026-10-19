# Dual Audiences & Document Map

Diosix documentation serves two primary audiences. Tailor each document (or
clearly separated sections within a document) to its intended reader.

## 1. Audience 1: Users (Beginners & Experts)

### Beginner Users
- Need frictionless onboarding, clear defaults, and copy-pasteable commands.
- Emphasize:
  - Interactive guided tour (`dsx tour` and the Welcome window in `diosix-gui`).
  - Simple imperative workflows (`dsx list`, `dsx run <image>`, `dsx ssh <vm>`,
    `dsx host info`, `dsx host install --mode basic --disk /dev/vda`).
  - Pre-configured guest OS catalog (`dsx image catalog`, `dsx image pull`).

### Expert Users & Operators
- Need deterministic automation, declarative configuration, and security
  auditing.
- Emphasize:
  - Declarative configuration files in `/etc/diosix/` (`system.toml`,
    `charter.toml`, `install.toml`, `catalog.toml`, `diosix.toml`) and their
    required file permissions.
  - Machine-readable `--json` output flags across `dsx` subcommands.
  - Capability attenuation (`dsx manifest prune`), air-gapped domains,
    zero-password FIDO2/TPM2/UART passkey authentication (`dsx auth`), and
    Constitutional Charter quorum operations (`dsx charter`).

## 2. Audience 2: Developers & Maintainers

- Need exact invariants, ABI contracts, build/test commands, and code standards.
- Emphasize:
  - Privilege separation between M-mode (`hypervisor/`), S-mode `dynarec`
    (`hypervisor/hardware/emulation/`), and U-mode (`tools/`).
  - Exact SBI Extension ID (`0x0a000008` / `"DIOS"`), Function IDs (`FIDs`
    `0..24`), and explicit fixed-width C struct layouts (`uint64_t`, `uint32_t`,
    `uint16_t`, `uint8_t`) in `docs/interface.md`.
  - Memory ownership (`defer`/`errdefer`, dynamic allocation once an allocator
    is available), 80-column line limits, and 500-line file limits in
    `docs/development.md` and `docs/style-guide.md`.
  - Testing structure: unit tests co-located inside the `.zig` source files
    they test (`./scripts/build.sh test --summary all`), and integration tests
    placed in dedicated `tests/` folders (and
    `./scripts/test_manifest_integration.py`).

## 3. Code-to-Documentation Synchronization Matrix

When modifying code, inspect and update the corresponding documentation files:

| Code Area Modified | Documentation Files to Update |
| :--- | :--- |
| `tools/diosix-ctl/` commands, flags, or `/etc/diosix/*.toml` | `docs/cli.md`, `docs/run.md`, `docs/security.md` |
| `hypervisor/interface/`, `hypervisor/core/sbi.zig`, or `tools/driver/` | `docs/interface.md`, `docs/architecture.md`, `docs/security.md` |
| `tools/diosix-gui/` modules, seat profiles, or OSK | `docs/gui-architecture.md`, `docs/cli.md`, `docs/release-architecture.md` |
| `build.zig`, `scripts/build*.sh`, or test layout | `docs/build.md`, `docs/development.md`, `README.md` |
| Coding conventions, file limits, or testing policy | `docs/style-guide.md`, `docs/development.md` |
