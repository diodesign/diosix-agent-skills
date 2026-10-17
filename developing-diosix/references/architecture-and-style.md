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

## 2. File Size, Line Length & Naming

- **<= 80 Columns**: Wrap all source code, comments, and multiline strings at
  80 characters. Always run `zig fmt` before committing.
- **<= 500 Lines per File**: Keep files focused on a single domain concept and
  under 500 lines unless there is a compelling architectural reason. Split
  growing modules proactively along clean domain boundaries.
- **Non-Redundant Filenames**: Use concise `snake_case` names. Never repeat
  `diosix` or parent folder names in filenames:
  - Good: `tools/diosix-ctl/src/api.zig`, `tools/alsa-init/main.c`
  - Bad: `tools/diosix-ctl/src/diosix_api.zig`,
    `tools/alsa-init/diosix-alsa-init.c`
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

## 4. Configuration Externalization (Zero Magic Values)

- Code provides **mechanism**; configuration files provide **policy/details**.
- Never hardwire tunable defaults (default RAM, vCPU counts, partition sizes,
  catalog URLs, SHA-256 digests, quorum thresholds, guided tour text) inside
  Zig source code when they belong in `/etc/diosix/*.toml` (`diosix.toml`,
  `catalog.toml`, `charter.toml`, `install.toml`, `system.toml`).
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
