# RISC-V Assembly & Enterprise Hypervisor Rules

## 1. Enterprise Type-1 Hypervisor Safety & Isolation

1. **Blast-Radius Containment (Never Panic on Guest Actions)**:
   - M-mode `panic` is reserved strictly for unrecoverable host hardware faults
     or hypervisor internal invariant corruption.
   - Untrusted or misbehaving guest VMs must **never** be able to crash the
     host. Malformed hypercalls, invalid GPAs, unaligned addresses, or quota
     exhaustions must return deterministic SBI error codes (`SBI_ERR_DENIED`,
     `SBI_ERR_INVALID_ADDRESS`, `SBI_ERR_INVALID_PARAM`, `SBI_ERR_FAILED`) or
     cleanly terminate only the offending guest subtree (`guest.terminate`).
2. **Double-Fetch (TOCTOU) Prevention & GPA Validation Pattern**:
   - Guest RAM can be mutated concurrently by other virtual cores of the same
     guest. When M-mode or a parent VM reads structured data from guest memory
     (such as `CharterArgs`, `WhisperArgs`, manifests, or VirtIO rings),
     validate the GPA range and copy the bytes into private hypervisor memory
     **once** before validating and acting on them:
     ```zig
     // 1. Bound size and check arithmetic overflow before touching memory.
     if (len == 0 or len > MAX_PAYLOAD_SIZE) return sbi.SBI_ERR_INVALID_PARAM;
     const end_gpa = std.math.add(u64, gpa, len) catch {
         return sbi.SBI_ERR_INVALID_ADDRESS;
     };
     _ = end_gpa;

     // 2. Copy page-by-page via guest.gpaToHpa into a private buffer once.
     var local_buf: [MAX_PAYLOAD_SIZE]u8 = undefined;
     if (!guest.copyFromGuest(local_buf[0..len], gpa)) {
         return sbi.SBI_ERR_INVALID_ADDRESS;
     }

     // 3. Parse/validate exclusively from `local_buf[0..len]`.
     ```
   - Never dereference a guest-supplied pointer directly as a host pointer, and
     always verify `!physmem.isHypervisorMemory(hpa)` when mapping memory.
3. **Deterministic Resource Reclamation & Page Zeroization**:
   - Every physical page allocated to a guest VM (`physmem`) must be scrubbed
     (zeroed) before allocation or upon reclamation so residual guest data or
     cryptographic material never leaks across VM lifecycles.
   - Zeroize ephemeral cryptographic buffers and challenge nonces using
     `std.crypto.secureZero`.
   - When a guest terminates, recursively terminate all descendant VMs (zero
     orphans) and reclaim 100% of its physical pages, page tables, and vCores.
4. **IRQ-Safe Locking & SMP Memory Ordering**:
   - Always use the canonical `Spinlock` in `hypervisor/core/atomic.zig` which
     saves, disables, and restores machine interrupts (`MSTATUS_MIE`) so a trap
     handler on the same hart cannot deadlock on a held lock:
     ```zig
     const flags = lock.lock();
     defer lock.unlock(flags);
     ```
   - Follow a strict lock acquisition order (`global_guest_lock` $\rightarrow$
     parent `guest.lock` $\rightarrow$ child `guest.lock` $\rightarrow$
     `physmem` / `pcore` locks) and never hold a spinlock across `hw_run_vcore`
     or `wfi`.
   - Use explicit `.acquire` / `.release` atomic orderings on cross-hart state.

## 2. RISC-V Register & Stack Conventions in M-Mode

- **`tp` (Thread Pointer, `x4`)**: Reserved in M-mode to point to the current
  physical CPU's `*CpuContext`. Never clobber `tp` in assembly or inline `asm`.
- **`mscratch` CSR**: Always holds the top address of the current physical
  CPU's M-mode exception/interrupt (`xint`) stack while executing guest or
  hypervisor code.
- **Trap Entry (`xint.s`)**:
  1. Atomically swap `sp` and `mscratch` (`csrrw sp, mscratch, sp`).
  2. Allocate `XINT_REGISTER_FRAME_SIZE` (`32 * 8` bytes) and save `x1` and
     `x3..x31`.
  3. Read original `sp` from `mscratch`, store it at `(2 * 8)(sp)`, and
     immediately restore `mscratch` (`addi t0, sp, XINT_REGISTER_FRAME_SIZE;
     csrw mscratch, t0; mv tp, t0`).
- **Alignment**: Keep all stacks 16-byte aligned (`.align 16`) and trap vector
  handlers at least 8-byte aligned (`.align 8`).

## 3. Host-Testable Hardware Abstractions (`if (is_test)`)

All hypervisor unit tests (`./scripts/build.sh test`) compile and run natively
on the **host build machine** without QEMU.
- Never execute raw privileged CSR instructions (`csrr`, `csrw`, `hfence.gvma`,
  `hlv.*`, `wfi`) or raw physical MMIO dereferences unconditionally.
- Wrap every hardware primitive in `hypervisor/hardware/native/cpu/riscv64/`
  with `if (is_test)` (or `if (comptime is_test)`) and read/write mock CSR
  variables (`test_mstatus`, `test_hgatp`, `test_cpu_ctx`, `test_heap`).
- Export mock symbols for external assembly routines inside
  `comptime { if (is_test) { @export(&hw_foo_mock, ...); } }`.

## 4. Address Space & Isolation Invariants

Never confuse address spaces; include the address space in variable names
(`dtb_hpa`, `entry_gpa`, `buf_gva`):
- **HPA**: Host Physical Address (DRAM or MMIO on host hardware).
- **GPA**: Guest Physical Address (translated via G-stage `sv39x4` or PMP).
- **GVA**: Guest Virtual Address (translated via VS-stage in guest OS).
- Always issue `hfence.gvma` when `gstage_dirty` is set before `mret`, and
  `fence.i` after emitting JIT translation blocks in `dynarec`.
- **H-Extension CSR Banking Caveat (`senvcfg` vs. `henvcfg`)**: While most
  S-mode CSRs (`0x100..0x180`) have VS-mode shadow CSRs at `0x200..0x280`
  (e.g., `vsstatus` at `0x200`, `vstimecmp` at `0x24d`), **CSR `0x20a` does not
  exist** in the RISC-V Privileged Architecture. Hypervisor environment
  configuration is in `henvcfg` (`0x60a`), and guest supervisor environment
  configuration (`readVsenvcfg` / `writeVsenvcfg`) is accessed via `senvcfg`
  (`0x10a`). Executing `csrr`/`csrw 0x20a` in M-mode triggers an immediate
  illegal-instruction fatal exception (`MTVAL: 0x20a02573`).
