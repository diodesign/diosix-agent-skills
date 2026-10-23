# Memory Management & Dynamic Buffers

## 1. Dynamic Allocation First (Stop Guessing Static Buffer Sizes)

Do not allocate arbitrary fixed-size stack buffers (such as
`var buf: [4096]u8 = undefined`) to format help text, serialize JSON/TOML, or
accumulate I/O when an allocator is available:
- Fixed stack buffers risk silent truncation (`error.NoSpaceLeft` swallowed by
  `catch return`), waste stack frames, or fail on legitimate large inputs.
- **Rule**: As soon as an allocator is initialized, use dynamic allocation
  (`allocator.print`, `allocator.alloc`, `std.ArrayList(u8)`) and grow buffers
  as needed.

### Example: Formatting Output Dynamically (Zig 0.17)

```zig
// PREFERRED: Dynamic allocation sized to exact content
pub fn printUsage(allocator: std.mem.Allocator) !void {
    const exe = common.getExeName();
    const msg = try allocator.print(
        "{s}: Diosix Type-1 Hypervisor Guest Management CLI\n...",
        .{exe},
    );
    defer allocator.free(msg);
    common.printStr(msg);
}

// AVOID: Guessing a magic 4096-byte stack buffer and swallowing errors
fn printUsageBad() void {
    var buf: [4096]u8 = undefined;
    const msg = std.fmt.bufPrint(&buf, "...", .{}) catch return;
    common.printStr(msg);
}
```

### Never Swallow a Formatting Failure

`std.fmt.bufPrint(...) catch return`, `catch ""`, and `catch "--"` turn a
too-small buffer into silent missing output. Prefer, in order:

1. A print helper that cannot fail silently: `common.printFmt(fmt, args)`
   formats into a small scratch buffer and spills to the heap, so nothing is
   truncated or dropped. Use it for all console output.
2. `allocator.print(...)` with `defer allocator.free(...)` for strings that are
   stored, serialized, or sent elsewhere (JSON, files, command lines).
3. For text drawn into a fixed-capacity widget (the GUI `Icon.text_buf`), a
   stack scratch buffer is acceptable only when its size is one named
   constant tied to that capacity and overflow truncates deliberately to the
   longest prefix that fits (`label.fmtLabel`) rather than blanking the text.

Every remaining fixed-size buffer needs a named constant and a one-line
comment explaining why that size is correct (`PATH_MAX`, one `getdents64`
record, a hex digest length, a derived `comptime` maximum). Bare `[128]u8`,
`[256]u8`, and `[512]u8` guesses are findings.

## 2. When Static Resources Are Permitted

Static buffers or pre-allocated slabs are permitted **only** in:
1. **Early M-mode boot (`entry.s` and pre-heap `main.zig`)**: Before
   `cpu_ctx.allocator.init(...)` completes on a physical core, cores `0..31`
   use static slabs and cores `32+` use `dyn_boot_stack`.
2. **M-mode panic / crash handlers (`debug.printf`, `panic`)**: Where heap
   locks may be held or corrupted.
3. **Hardware-mandated fixed ABI structures**: Fixed-size page tables
   (`4096`-byte pages), SBI shared rings, or kernel ioctl structs (`api.zig`).

Everywhere else — including M-mode post-heap initialization (`boot.zig`,
`guest.zig`, `dt.zig`, `sbi.zig`) and all U-mode tools (`diosix-ctl`,
`diosix-gui`, `diosix-config`, `guest-shim`) — pass and use `std.mem.Allocator`.

## 3. Allocator Selection by Layer & Asking When Unsure

- **M-mode Hypervisor**: Obtain the current physical core's `std.mem.Allocator`
  interface via `pcore.this().allocator.allocator()`, or use the guest/system
  allocator passed down from `bootCpuInit`.
  - **Important — Ask When Unsure**: Custom M-mode structures like
    `HeapAllocator` (`hypervisor/core/alloc.zig`) only implement the core
    allocation primitives (`alloc`, `resize`, `remap`, `free`) and do not
    directly define convenience methods like `print` on `HeapAllocator` itself
    (whereas `std.mem.Allocator` provides `print` in the standard library, and
    M-mode console logging uses `debug.printf`). If you encounter a place where
    a required helper is missing or you are unsure whether to extend a
    hypervisor allocator or use `std.mem.Allocator` / `debug.printf`, **stop
    and ask the lead maintainer for direction rather than guessing**.
- **U-mode CLI Tools (`diosix-ctl`, `diosix-config`)**:
  - Use `std.heap.ArenaAllocator` for short-lived command execution where all
    command memory can be freed at once on exit, or pass `gpa` /
    `page_allocator` with explicit `defer allocator.free(...)`.
- **U-mode Long-Running Daemons (`diosix-gui`, `guest-shim`)**:
  - Pass an explicit `std.mem.Allocator` into `init` / `deinit` methods and
    verify zero leaks in unit tests using `std.heap.SafeAllocator`.

## 4. Ownership, `defer`, `errdefer`, and In-Place Initialization

1. **Immediate Cleanup Registration**: Place `defer` or `errdefer` on the line
   immediately following every allocation or `mmap`:
   ```zig
   const buf = try allocator.alloc(u8, size);
   errdefer allocator.free(buf);
   ```
2. **In-Place Out-Pointer Initialization**: For large structs or structs
   containing self-referential pointers/locks (`Guest`, `VirtualCore`,
   `CpuContext`), initialize in-place via an out-pointer (`fn init(target: *T)`)
   rather than returning by value to avoid stack copies and dangling pointers.
3. **Shared Process State**: Store process-wide read-mostly metadata (such as
   `common.setExeName(argv[0])` / `common.getExeName()`) in a single cohesive
   module rather than threading baggage parameters through every function call.
