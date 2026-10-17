# Borrow-Checker & Pointer Lifetime Audit

Zig does not have a compile-time borrow checker. When reviewing Zig and
assembly code in Diosix, manually emulate Rust's borrow checker by tracing the
**provenance**, **lifetime**, and **aliasing mutability** of every pointer and
slice.

## 1. Lifetime & Provenance Tracing (`'a`)

For every pointer (`*T`, `[*]T`) and slice (`[]T`, `[:0]T`) introduced in a
function:

1. **Identify the Owner**:
   - Is it owned by a heap `Allocator` (`alloc`, `create`, `dupe`, `print`)?
   - Is it backed by an `mmap` region (`readBinaryFile`)?
   - Is it borrowing a local stack variable (`var buf: [N]u8`)?
   - Is it borrowing a field inside a parent struct (`&self.field`)?
2. **Verify Lifetime Outlives All Uses**:
   - **Stack Escape Bug**: Confirm no slice or pointer to a local stack
     variable is returned from the function, stored in a longer-lived struct,
     or passed to an asynchronous/child thread context.
   - **Use-After-Free (UAF)**: Confirm `defer allocator.free(x)` or
     `defer unmapBinaryFile(x)` executes **after** the last read/write of `x`
     and any sub-slices (`x[start..end]`) derived from `x`.
   - **Loop Scope Trap**: Remember that `defer` inside a `while`/`for` block
     runs at the end of **each iteration**, not the end of the function. Any
     slice allocated and `defer`-freed inside a loop body becomes a dangling
     pointer if stored in an outer variable!

## 2. Aliasing Mutability & Container Reallocation

 Emulate Rust's rule: *either one mutable reference (`&mut T`) OR any number of
immutable references (`&T`), never both simultaneously across a mutation.*

1. **Slice Invalidation on Growth**:
   - Whenever `ArrayList.append`, `ArrayList.ensureTotalCapacity`,
     `allocator.resize`, or `allocator.remap` is called, assume the underlying
     backing array may be relocated in memory.
   - **Check**: Are any pointers or slices (`&list.items[i]`, `const slice =
     list.items`) held across an `append` or reallocation? If so, flag as a
     critical UAF bug; store integer indices (`usize`) instead of pointers!
2. **Iterator Invalidation**:
   - Never insert into or remove from an intrusive `LinkedList`
     (`hypervisor/core/dsa.zig`) or array while iterating over it without
     capturing `const next = node.next;` **before** mutating or freeing `node`.
3. **Overlapping Memory Copies**:
   - In Zig, `@memcpy(dst, src)` asserts that `dst` and `src` do not overlap.
     If source and destination slices can alias, use `std.mem.copyForwards` or
     `std.mem.copyBackwards`.

## 3. Partial Initialization & `errdefer` Audit

Trace every `return err` or `try` failure point between the start and end of a
function:

```zig
pub fn createResource(allocator: Allocator) !*Resource {
    const res = try allocator.create(Resource);
    errdefer allocator.destroy(res); // Frees `res` if `buf` allocation fails

    res.buf = try allocator.alloc(u8, 1024);
    errdefer allocator.free(res.buf); // Frees `res.buf` if later step fails

    try initializeHardware(res);
    return res;
}
```

- [ ] If step $K$ fails with an error, are resources allocated in steps
      $1 \dots K-1$ all released in reverse order without leaking or
      double-freeing?
- [ ] Once ownership is transferred to a global table or returned to the
      caller, is there any fallible `try` remaining that could trigger an
      `errdefer` and leave a dangling pointer in the global table?

## 4. Cross-Privilege & Alignment Pointer Safety

- [ ] **Alignment**: Does every `@ptrCast` to a multi-byte pointer type either
      prove alignment via `@alignCast` or use an explicitly unaligned pointer
      (`*align(1) const T` / `std.mem.readInt`)?
- [ ] **MMIO Volatility**: Are all pointers to hardware MMIO registers (UART,
      CLINT, PLIC, VirtIO) marked `volatile` (`*volatile u32`) so the compiler
      cannot elide or reorder hardware reads/writes?
- [ ] **Foreign Guest Mapping**: When `diosix-ctl` maps child guest RAM via
      `MAP_CHILD_MEM`, is the mapped slice unmapped in a `defer` block before
      `START_GUEST` or function return?
