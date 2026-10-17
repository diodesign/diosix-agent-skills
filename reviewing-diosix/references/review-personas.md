# The 4 Review Personas Checklist

Evaluate every diff and its surrounding module context through all four
personas. Flag and fix every finding before completing the task.

## 1. The Skeptical Veteran Engineer
*Hunts incomplete features, code smells, shortcuts, non-idiomatic Zig, and
structural bloat.*

- [ ] **Complete & Working Implementation**: Does the code build cleanly and
      implement 100% of the design without `TODO` stubs, fake fallbacks, or
      half-wired flags?
- [ ] **No `std.mem.eql` Chains**: Are all command, flag, and key dispatches
      using `enum` + `std.meta.stringToEnum` + `switch`?
- [ ] **No Static Buffer Guessing**: Are fixed stack buffers (`var buf:
      [4096]u8`) used where an allocator is available? Replace them with
      dynamic allocations (`allocator.print`, `allocator.alloc`, `ArrayList`).
- [ ] **File & Line Limits**: Is every modified file **<= 500 lines** (unless
      architecturally unavoidable) and every line **<= 80 columns**?
- [ ] **No Redundant Filenames**: Are filenames concise (`api.zig`, `main.c`)
      without redundant `diosix_` prefixes?
- [ ] **Zig 0.17 Compliance**: Are `@backingInt`, `@fromBackingInt`, `@splat`,
      `allocator.print`, `list.last()`, and `SafeAllocator` used consistently?
- [ ] **No Magic Values**: Are tunable parameters externalized to
      `/etc/diosix/*.toml` rather than hardcoded in Zig?
- [ ] **No Swallowed Errors**: Are all error unions handled explicitly via
      `try`, `catch`, or `errdefer` (no silent `_ = foo() catch {};`)?

## 2. The Paranoid Security Expert
*Assumes untrusted guests and malformed inputs are actively trying to break
isolation, corrupt memory, or cause denial of service.*

- [ ] **Blast-Radius Containment (Zero Guest-Induced Panics)**: Can any guest
      input, malformed hypercall, or invalid GPA trigger an M-mode `panic` or
      infinite loop? (Must return an SBI error or terminate only the offending
      guest.)
- [ ] **No Double-Fetch (TOCTOU) on Guest Memory**: Is data in guest-writable
      memory copied into private hypervisor/parent memory **once** before
      validation and use?
- [ ] **M-Mode / Guest Privilege Boundary**: Does every SBI hypercall validate
      that GPAs and sizes lie strictly within the calling guest's owned memory
      before converting GPA to HPA?
- [ ] **Checked Arithmetic**: Are all size/offset calculations (`offset + len`,
      `count * stride`) using `std.math.add` / `std.math.mul` to prevent
      integer overflow wrap-around?
- [ ] **Secret & Page Zeroization**: Are freed guest physical pages zeroed and
      ephemeral keys/nonces wiped via `std.crypto.secureZero`?
- [ ] **IRQ-Safe Locking & Lock Ordering**: Do trap-reachable spinlocks save/
      restore IRQ flags (`const f = lock.lock(); defer lock.unlock(f);`), and
      is the global lock hierarchy respected without holding locks across
      `hw_run_vcore` or `wfi`?
- [ ] **Constitutional & Steward Invariants**: Can Root VM (`CID 1`) bypass a
      sealed Constitutional Charter, forge a Steward vote, kill a `STEWARD` VM,
      or map a `CONFIDENTIAL` guest's memory? (Must be impossible.)

## 3. The Diosix Maintainer
*Cares about long-term clarity, testability, and zero-friction debugging.*

- [ ] **Single Responsibility**: Does each file and function have one clear
      purpose, with branching pushed up to coordinators and loops pushed down
      to pure helpers?
- [ ] **Impeccable English Comments & Docs**: Are all comments and docs written
      in clear, grammatically correct English with verified spelling, full
      sentences ending in a period, and explaining *why* rather than *what*?
- [ ] **Test Placement & Coverage**: Are unit tests co-located at the bottom of
      the `.zig` file they test (with `std.testing.refAllDecls(@This())`), and
      are multi-component integration tests in `tests/`?

## 4. The Beginner & Expert User
*Expects a responsive, intuitive, reliable system with zero toil.*

- [ ] **Actionable Error Messages**: Do CLI and GUI errors clearly state in
      plain English what went wrong and how to fix it (using
      `common.getExeName()` rather than a hardcoded binary name)?
- [ ] **Beginner + Expert Ergonomics**: Do interactive defaults work out of the
      box for beginners while supporting `--json` and `/etc/diosix/*.toml`
      overrides for experts?
- [ ] **Performance & Responsiveness**: Does the change avoid unnecessary
      allocations in hot loops, redundant MMIO writes, or frame-rate drops in
      `diosix-gui`?
