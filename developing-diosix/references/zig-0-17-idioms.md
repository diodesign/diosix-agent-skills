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

## 3. Explicit Integer Division & Overflow Safety

- Always specify rounding intent on signed or inexact division:
  - `@divTrunc(a, b)` (toward zero)
  - `@divFloor(a, b)` (toward $-\infty$)
  - `@divCeil(a, b)` (toward $+\infty$)
  - `@divExact(a, b)` (asserts zero remainder)
- Use checked arithmetic (`std.math.add`, `std.math.mul`, `std.math.sub`) when
  computing buffer sizes, offsets, GPA/HPA ranges, or ELF segment bounds from
  untrusted inputs.

## 4. Reflection & Build System Hygiene

- `@hasDecl(T, "name")` returns `true` **only** for `pub` declarations, even
  within the same file.
- Struct/union/enum type reflection uses Struct-of-Arrays (SoA) fields
  (`info.field_names`, `info.field_types`, `info.field_attrs`).
- In `build.zig`, keep the configuration cache pure:
  - Use `run_cmd.addPassthruArgs()` instead of inspecting `b.args`.
  - Use `b.findProgramLazy(...)` instead of `b.findProgram(...)`.
  - Register all file inputs to system commands via `addFileInput(b.path(...))`.
