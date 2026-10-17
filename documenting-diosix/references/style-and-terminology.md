# Style, Voice & Canonical Terminology

All Diosix documentation must strictly follow `docs/style-guide.md`.

## 1. Language, Voice, Tone & Line Wrapping

- **Clear, Correct English**: Write all documentation and comments in English
  with verified spelling and sound grammar. Be concise, but never compress
  wording to the point where technical meaning becomes ambiguous.
- **Direct & Technical**: Avoid marketing hype, superlatives, and filler words.
  Every sentence must convey concrete technical information.
- **Active Voice & Present Tense**:
  - Good: "The Root VM prunes the system manifest before launching a child VM."
  - Bad: "The system manifest is pruned by the Root VM when a child VM is
    going to be launched."
- **80-Column Line Wrap**: Wrap all markdown paragraphs and list items at
  approximately 80 characters so files remain readable in standard terminal
  splits and `git diff`.
- **Sentence-Case Headings**: Capitalize only the first word and proper nouns
  in all headings (`#`, `##`, `###`, `####`).
- **List Discipline**:
  - **Numbered lists (`1.`, `2.`)**: Use *only* for ordered, step-by-step
    procedures or lifecycle sequences.
  - **Bulleted lists (`*` or `-`)**: Use for unordered features, properties, or
    definitions.

## 2. Abbreviations

- On the **first mention on a given page**, spell out the term followed by its
  abbreviation in parentheses:
  - `Virtual Machine (VM)`
  - `Supervisor Binary Interface (SBI)`
  - `Guest Physical Address (GPA)`
  - `Host Physical Address (HPA)`
  - `Physical Memory Protection (PMP)`
  - `Trusted Computing Base (TCB)`
  - `Context ID (CID)`
  - `Executable and Linkable Format (ELF)`
  - `Flattened Device Tree (FDT)`
- On **subsequent mentions on the same page**, use only the abbreviation (`VM`,
  `SBI`, `GPA`).
- **Exceptions**: Universal industry terms (`CPU`, `RAM`, `ROM`, `OS`, `GUI`,
  `CLI`, `SSH`, `JSON`, `TOML`, `YAML`, `QEMU`, `GPT`, `UEFI`, `BIOS`) do not
  need expansion.

## 3. Canonical Terminology Definitions

Never conflate the build host, physical host hardware, and guest VMs:

| Term | Exact Meaning |
| :--- | :--- |
| **Host build machine** (or **build machine**) | The computer where source code is compiled and images are built. |
| **Host hardware** (or **host system**) | The physical machine, board, or SoC running the compiled hypervisor. |
| **Target platform** (or **target system**) | The board, CPU architecture, or emulator machine (`qemu-system-riscv64`) targeted by the build. |
| **Guest VM** | Any virtual machine running atop the hypervisor. |
| **Root VM** | The initial privileged guest VM (`CID 1` at boot) acting as the system progenitor (`init`-equivalent). |
| **Steward VM** | A hardware-protected Tier-1 governance VM (`steward-sec`, `steward-ops`, `steward-gov`) enforcing the Constitutional Charter. |
| **Standby Root VM** | A pre-staged warm failover Root VM promoted by M-mode if the primary Root VM fails or is voted out. |
| **Host Physical Address (HPA)** | A physical memory or MMIO address on the host hardware. |
| **Guest Physical Address (GPA)** | A physical memory address from a guest VM's perspective (translated via G-stage `sv39x4` or PMP). |
| **Guest Virtual Address (GVA)** | A virtual address managed by a guest VM's supervisor OS (VS-stage). |
| **Relative Context ID (CID)** | Tree-relative VM identifier: `CID 0` (immediate parent), `CID 1` (self), `CID 2..N` (direct children). |
