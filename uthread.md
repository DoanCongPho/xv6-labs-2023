# Part 1: Uthread — switching between threads

A user-level threading package for xv6. The kernel sees one process; inside it, several
"threads" cooperate by voluntarily yielding the CPU. Switching between them is just
saving 14 RISC-V registers into one struct and loading 14 from another.

## What was missing

Three holes in the starter code:

1. `struct thread` had nowhere to park a thread's registers.
2. `thread_create()` set the new thread's state to `RUNNABLE` but didn't arrange for
   the first switch into it to actually start `func` on its own stack.
3. `thread_schedule()` picked a `next_thread` but never called `thread_switch`.
4. `thread_switch` (assembly) was a single `ret` that did nothing.

## The implementation

### A `struct context` to hold saved registers

```c
struct context {
  uint64 ra;
  uint64 sp;
  uint64 s0;  uint64 s1;  uint64 s2;  uint64 s3;
  uint64 s4;  uint64 s5;  uint64 s6;  uint64 s7;
  uint64 s8;  uint64 s9;  uint64 s10; uint64 s11;
};

struct thread {
  char           stack[STACK_SIZE];
  int            state;
  struct context context;
};
```

14 fields, laid out so the offsets `0, 8, 16, …, 104` match the `sd`/`ld` immediates
in the assembly. Same shape as the kernel's `struct context` in
[kernel/proc.h](kernel/proc.h#L2).

### `thread_create` — fabricate a startup context

```c
t->state = RUNNABLE;
t->context.ra = (uint64)func;
t->context.sp = (uint64)(t->stack + STACK_SIZE);
```

The trick: a brand-new thread has never run, but we make its context *look like* a
thread that previously called `thread_switch`.

- `ra = func`: the `ret` at the end of `thread_switch` will jump straight into the
  thread's start function on the very first switch into it.
- `sp = stack + STACK_SIZE`: RISC-V stacks grow downward, so the initial pointer
  must be the high end of the stack buffer.
- The 12 s-registers are left as garbage — `func` doesn't read them before writing
  its own values.

### `thread_schedule` — invoke the switch

```c
if (current_thread != next_thread) {
  next_thread->state = RUNNING;
  t = current_thread;
  current_thread = next_thread;
  thread_switch((uint64)&t->context, (uint64)&next_thread->context);
}
```

First arg = the *outgoing* context (where to save into); second arg = the *incoming*
context (where to load from). The order matches the assembly's use of `a0` for
stores and `a1` for loads.

### `thread_switch` (assembly)

```asm
.globl thread_switch
thread_switch:
    sd ra, 0(a0)
    sd sp, 8(a0)
    sd s0, 16(a0)
    …
    sd s11, 104(a0)

    ld ra, 0(a1)
    ld sp, 8(a1)
    ld s0, 16(a1)
    …
    ld s11, 104(a1)

    ret
```

Same shape as the kernel's [swtch.S](kernel/swtch.S). 14 stores into `*a0`, 14 loads
from `*a1`, then `ret` jumps to whatever was just loaded into `ra` — landing either
at `func` (first entry) or at the instruction after the previous `thread_switch(...)`
call (a resumed thread).

## Why these specific 14 registers

```
 1 × ra              return address
 1 × sp              stack pointer
12 × s0–s11          callee-saved registers
─────
14 total
```

This is the smallest set that captures *where to resume*, *on which stack*, and
*with which live local variables*.

### Why we *don't* save `t0–t6` and `a0–a7`

The RISC-V ABI splits registers into roles:

| Class | Registers | Who preserves them across a call |
|-------|-----------|----------------------------------|
| Hardwired | `zero` | n/a |
| Caller-saved | `ra`, `t0–t6`, `a0–a7` | The **caller** spills before the call if it cares |
| Callee-saved | `sp`, `s0–s11` | The **callee** must restore them before returning |
| Per-process | `gp`, `tp` | Nobody — same for every thread |

`thread_switch` is reached via a normal C call. The compiler in `thread_schedule`
*already* spilled any caller-saved register it needed alive across the call —
that's the ABI promise. So `t`/`a` registers hold dead values at switch time;
saving them would be wasted work.

### Why `ra` is saved even though it's caller-saved

`ra` is "caller-saved" in the sense that the compiler's normal prologue won't
preserve it across a call. But in `thread_switch` we're *swapping which call we're
returning from*. We must capture this thread's `ra` so that when control returns
here later, the `ret` jumps back to *its* caller (not to whoever's call site we
happen to load).

### Why the s-registers are saved

The compiler uses `s0–s11` to hold local C variables that need to survive function
calls (e.g., the loop variable `i` in `thread_a`). If `thread_switch` clobbered
them while loading the new thread's state without first saving the old, the
outgoing thread's locals would be corrupted on resume.

### Why we don't save FP, `gp`, `tp`, or memory

- **Floating-point**: xv6 user code doesn't use FP. If it did, we'd need
  `f0–f31` + `fcsr` (33 more saves).
- **`gp`, `tp`**: process-wide; identical for every thread.
- **`zero`**: hardwired to 0.
- **Memory** (globals, heap, stack contents): already in memory; not lost when a
  thread sleeps. The stack in particular sits untouched in `t->stack[]` while
  another thread runs on a different `sp`.

## How `a0` and `a1` get the right values

The RISC-V calling convention places the first 8 integer/pointer arguments in
`a0`–`a7` in left-to-right order. So for:

```c
void thread_switch(uint64 old, uint64 new);
```

…the compiler emits code at the call site that loads `old` into `a0` and `new`
into `a1`, then `jal thread_switch`. On entry to the assembly, the registers are
already correct — the asm doesn't have to look anything up.

## How `i` (and other locals) survive a switch

When `thread_a` is mid-loop and calls `thread_yield`:

```
thread_a   (i held in some s-register, say s1)
  └─ thread_yield()
       └─ thread_schedule()
            └─ thread_switch()
```

One of two things has happened by the time `thread_switch` runs:

- **Case A — nobody clobbered s1.** `i` is still live in `s1`. `thread_switch`
  saves it directly into `t->context.s1`.
- **Case B — `thread_yield`/`thread_schedule` reused `s1`.** Whoever did so
  spilled the old `i` to its own stack frame first (per ABI), then used `s1` for
  its own work. `thread_switch` saves whatever transient is in `s1` now; the real
  `i` is sitting on `thread_a`'s stack inside that intermediate frame.

Either way, on resume the cascade unwinds in mirror order:
- `thread_switch` loads its `s1` from `t->context.s1`.
- `thread_schedule`'s epilogue loads `s1` from its stack frame (overwriting in
  Case B with the original `i`).
- Control returns to `thread_a` with `s1` holding `i` again.

The general invariant: **save the 14 registers, leave the stack memory alone**.
Anything else of value is reachable through `sp` and the chain of saved frames.

## What `sd` and `ld` do

| Instruction | Meaning |
|-------------|---------|
| `sd rs2, off(rs1)` | Store doubleword: write the 64-bit value in `rs2` to memory at `rs1+off` |
| `ld rd, off(rs1)`  | Load doubleword: read 64 bits from `rs1+off` into `rd` |

"Doubleword" = 64 bits in RISC-V (a "word" is fixed at 32 bits, inherited from
MIPS). Each `struct context` field is `uint64`, so doubleword is the right size,
and the offsets `0, 8, 16, …, 104` are simply `8 * field_index`.

## The first switch out of `main`

`thread_init()` enrolls `main` itself as `all_thread[0]` with state `RUNNING`.
This matters because the very first `thread_switch` needs *somewhere* to save the
currently running registers — without thread 0, the initial `sd` instructions
would have nowhere valid to write.

`main` then creates the three test threads, marks itself `FREE`, and calls
`thread_schedule()` — handing the CPU to one of `a/b/c` and abandoning its own
stack forever. The `exit(0)` after `thread_schedule()` is unreachable; the
process actually exits when the scheduler later runs out of `RUNNABLE` threads
and prints `thread_schedule: no runnable threads`.

## Cooperative scheduling

This package is **cooperative**: a thread runs until it explicitly calls
`thread_yield()`. There's no timer interrupt, no preemption. That's why we get
away with saving only 14 registers — the switch only ever happens at a function
call boundary, where the ABI has already cleaned up caller-saved state.

A preemptive scheduler (interrupt-driven) would have to save all 32 integer
registers, because preemption can land between any two instructions when caller-
saved registers may still hold live values.

## Testing

```bash
make qemu
$ uthread
```

Expected: `thread_a/b/c started`, then 100 interleaved lines per thread, then
each thread's `exit after 100`, then `thread_schedule: no runnable threads`.

Or just:

```bash
make grade
```

Should print `uthread: OK`.
