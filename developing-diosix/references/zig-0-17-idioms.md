# Zig 0.17.0 Idioms & Builtins

Always target **Zig 0.17.0** (stable). Do not use deprecated 0.15/0.16 APIs.

## 1. Enum Dispatch with `std.meta.stringToEnum`

Never write `if (std.mem.eql(u8, s, ...)) else if (...)` chains. Define a
type-safe `enum` and dispatch with `switch`:

```zig
const Command = enum {
    host,
    info,
    run,
    create,
    @"install-guest",
    @"--help",
    @"-h",
};

if (std.meta.stringToEnum(Command, arg)) |cmd| {
    switch (cmd) {
        .host => try host.cmdHost(&client, rest),
        .info => try vm.cmdInfo(&client, rest),
        .run, .create => try vm.cmdRun(&client, rest),
        .@"install-guest" => try vm.cmdInstallGuest(&client, rest),
        .@"--help", .@"-h" => try printUsage(allocator),
    }
} else {
    return error.UnknownCommand;
}
```

Quote hyphenated or flag identifiers in both enum declarations and `switch`
prongs using `@"..."` (e.g., `.@"--ram"`, `.@"-m"`).

## 2. Zig 0.17.0 Mandatory Replacements

| Deprecated / Removed | Zig 0.17.0 Idiom |
| :--- | :--- |
| `@intFromEnum(x)` | `@backingInt(x)` (works on enums, bitpacks, tagged unions) |
| `@enumFromInt(x)` | `@fromBackingInt(x)` (destination type inferred) |
| `[1]u8{0} ** N` | `@splat(0)` (e.g., `var buf: [64]u8 = @splat(0);`) |
| `@bitCast(extern_struct)` | `@as(*align(1) const T, @ptrCast(&val)).*` or `extern union` |
| `std.fmt.allocPrint(a, ...)` | `a.print(fmt, args)` (method on `std.mem.Allocator`) |
| `list.getLastOrNull()` | `list.last()` (and `list.last().?` for `getLast()`) |
| `std.heap.DebugAllocator` | `std.heap.SafeAllocator` |
| `std.heap.StackFallbackAllocator` | `std.heap.BufferFirstAllocator.init(buf, fallback)` |
| `void{}` | `{}` |
| `errdefer \|err\|` | Move into helper function and handle via `catch \|err\|` |
| `@import("builtin").cpu` | `@import("builtin").target.cpu` (`.os`, `.abi`, `.ofmt`) |
| `std.builtin.OptimizeMode` | `std.lang.Optimize` (`.debug`, `.safe`, `.fast`, `.small`) |

## 3. Explicit Integer Widths, Division & Overflow Safety

- **Fixed-Width Types in ABI & Architectural State**: Never use `usize` or
  `isize` in `extern struct` / `packed struct` ABI definitions (`sbi.zig`,
  `api.zig`, ELF/FDT headers) or 64-bit guest register files. Always use
  explicit fixed-width types (`u64`, `u32`, `u16`, `u8`) so layouts are
  identical across 64-bit and 32-bit targets.
- **Explicit Type Annotation on `@min` / `@max` with Literals**: In Zig,
  `@min(runtime_usize, 1280)` narrows the result type to `u11` (the smallest
  unsigned integer fitting `1280`)! Multiplying `y * row_w` will then panic
  with integer overflow. Always annotate the destination type explicitly:
  ```zig
  // GOOD: Explicitly typed as usize so `y * row_w` cannot overflow u11.
  const row_w: usize = @min(surface.width, 1280);

  // BAD: Infers `row_w` as `u11`, panicking when `y * row_w >= 2048`!
  const row_w = @min(surface.width, 1280);
  ```
- **Rounding Intent on Division**: Always specify rounding intent on signed or
  inexact division:
  - `@divTrunc(a, b)` (toward zero)
  - `@divFloor(a, b)` (toward $-\infty$)
  - `@divCeil(a, b)` (toward $+\infty$)
  - `@divExact(a, b)` (asserts zero remainder)
- **Checked Arithmetic**: Use `std.math.add`, `std.math.mul`, and
  `std.math.sub` when computing buffer sizes, offsets, GPA/HPA ranges, or ELF
  segment bounds from untrusted inputs.
- **Lowercase Hexadecimal & Named Bitmask/Page-Offset Helpers**: Always write
  hexadecimal literals in lowercase (`0xfff`, `0x8000_0000`, `0xffff_ffff`,
  never `0xFFF` or `0xFFFFFFFF`). Never write raw magic bitmasks like
  `if ((vaddr & 0xfff) > 0xffc)` — which at first glance looks like a check on
  the first 4 KiB of memory rather than a 4 KiB page-boundary straddle check.
  Define named constants and inline predicates instead:
  ```zig
  pub const PAGE_SHIFT = 12;
  pub const PAGE_SIZE: u32 = 1 << PAGE_SHIFT;
  pub const PAGE_OFFSET_MASK: u32 = PAGE_SIZE - 1; // 0xfff

  inline fn crossesPageBoundary(vaddr: u32, size: u32) bool {
      return (vaddr & PAGE_OFFSET_MASK) > (PAGE_SIZE - size);
  }
  ```

## 4. Guiding `zig fmt` to <= 80 Columns

`zig fmt` preserves single-line calls and initializers unless you give it a
syntactic line-break hint:
- **Trailing Commas**: Add a trailing comma after the last argument of a
  function call, struct literal, or switch prong argument list so `zig fmt`
  wraps it cleanly across multiple lines <= 80 columns.
- **String Concatenation & Multiline Strings**: Split long format strings
  across lines with `++` or `\\` multiline string literals rather than letting
  strings exceed column 80.
- **Module-Level Aliases**: Bind deeply nested imports at the top of the file
  (`const sbi = @import("interface").sbi;`) to keep type signatures concise.

## 5. Reflection & Build System Hygiene

- `@hasDecl(T, "name")` returns `true` **only** for `pub` declarations, even
  within the same file.
- Struct/union/enum type reflection uses Struct-of-Arrays (SoA) fields
  (`info.field_names`, `info.field_types`, `info.field_attrs`).
- In `build.zig`, keep the configuration cache pure:
  - Use `run_cmd.addPassthruArgs()` instead of inspecting `b.args`.
  - Use `b.findProgramLazy(...)` instead of `b.findProgram(...)`.
  - Register all file inputs to system commands via `addFileInput(b.path(...))`.
  - Keep module flags (`--dep interface -Mroot=... -Minterface=...`) in
    `scripts/build_rootvm.sh` 100% synchronized with `build.zig`.
