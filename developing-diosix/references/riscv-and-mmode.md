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
2. **Double-Fetch (TOCTOU) Prevention on Guest Memory**:
   - Guest RAM can be mutated concurrently by other virtual cores of the same
     guest. When M-mode or a parent VM reads structured data from guest memory
     (such as hypercall buffers, manifests, Whisper frames, or VirtIO rings),
     copy the bytes into private hypervisor/parent memory **once** before
     validating and acting on them. Never validate in-place and re-read from
     guest memory.
3. **Deterministic Resource Reclamation & Page Zeroization**:
   - Every physical page allocated to a guest VM (`physmem`) must be scrubbed
     (zeroed) before allocation or upon reclamation so residual guest data or
     cryptographic material never leaks across VM lifecycles.
   - Zeroize ephemeral cryptographic buffers and challenge nonces using
     `std.crypto.secureZero`.
   - When a guest terminates, recursively terminate all descendant VMs (zero
     orphans) and reclaim 100% of its physical pages, page tables, and vCores.
4. **IRQ-Safe Locking & SMP Memory Ordering**:
   - Spinlocks acquired in paths reachable from `xint_handler` must disable
     machine interrupts (`const flags = lock.lock(); defer lock.unlock(flags);`).
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
