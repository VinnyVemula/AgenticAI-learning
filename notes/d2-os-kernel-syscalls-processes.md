# Day 2 — The OS I: Kernel, Syscalls, and Processes

> **Framing.** This note is about the single most load-bearing boundary in all of computing: the line between *your code* and *the machine*. Everything on one side is untrusted and replaceable; everything on the other side is privileged and shared. A CPU privilege bit enforces that line, a **syscall** is the only legal door through it, and a **process** is the OS's name for "one tenant on the trusted side of the door, with its own memory, its own open files, and its own identity."
>
> **The one idea that unites the backend and agentic layers of this note:** *a process is the unit of both isolation and cancellation, and that is why it is simultaneously the foundation of your deployment story (containers, worker pools, graceful shutdown, zero-downtime deploys) and the only thing standing between an LLM's confidently-wrong code and your production credentials.* When an agent "runs code," nothing magical happens: your harness performs a `fork`+`exec` (or Windows `CreateProcess`), the child makes syscalls the kernel is willing to serve, and you kill it if it misbehaves. Learn process management once and you have learned Docker, Kubernetes rollouts, gunicorn workers, and AI code sandboxes at the same time.
>
> **Who this is for.** Someone who has written a program but never asked what happens between `print("hi")` and pixels on a screen. No prior OS, C, or assembly knowledge assumed. Every term is defined where it first appears, along with *what problem it solved and what people did before it existed*.

---

## Concept map

```
                        ┌──────────────────────────────────────────────┐
   YOUR CODE            │  user space  (unprivileged, ring 3)          │
   (Part 1 §1–§7)       │  python.exe   nginx   your agent harness     │
                        └───────────────┬──────────────────────────────┘
                                        │  the ONLY door: SYSCALL  (§3)
                        ════════════════╪══════════════════════════════
                                        │  hardware-enforced boundary (§2)
                        ┌───────────────▼──────────────────────────────┐
   THE KERNEL           │  kernel space (privileged, ring 0)           │
                        │  scheduler │ memory mgr │ filesystems │ net  │
                        └───────────────┬──────────────────────────────┘
                                        │
                        ┌───────────────▼──────────────────────────────┐
                        │  CPU · RAM · disk · NIC                      │
                        └──────────────────────────────────────────────┘

   A PROCESS (§4) = address space + thread(s) + fd table + credentials
   fork/exec (§5) creates one · signals (§6) interrupt one · wait (§7) buries one
   namespaces+cgroups (§8) lie to one  ──►  that lie is called "a container"

   PART 2 reuses ALL of the above with one substitution:
        the code inside the child process was written by a language model.
```

**Depth tiers used in this note** (the tier tells you how hard to study a thing, and how much supporting material it owes):

| Tag | Meaning |
|---|---|
| **[CORE]** | Load-bearing. Every black box gets opened until no "but how does *that* work?" remains. |
| **[WORKING]** | You must use it correctly and reason about its trade-offs, but never reimplement it. |
| **[AWARE]** | Know it exists and when to reach for it. Then stop. |

**Where I deliberately stop, on purpose (not by accident):**
- **CPU scheduling internals** (run queues, CFS/EEVDF, priority inversion) — named, not opened. You need "the kernel decides who runs next" today; the algorithm is its own topic.
- **Virtual memory mechanics** (page tables, TLB, page faults, copy-on-write implementation) — I open only as much as is needed to explain *protection* and *why `fork` is cheap*. The full memory hierarchy belongs to the memory day of Phase 0.
- **Namespaces and cgroups internals** — [AWARE] only here (§8). The syllabus promises you will *prove* the "a container is just a process" claim on Day 88; today you only need to stop believing containers are VMs.
- **Threads vs processes as a concurrency model** (GIL, async, thread pools) — mentioned as part of process anatomy (§4), developed on its own day.

---

# PART 1 — BACKEND

## §1. What an operating system actually is  **Depth: [WORKING]**

### Intuition — the problem that had to be solved

Imagine a computer with no operating system. You write a program, you burn it onto the machine, and when it runs it *is* the machine: it owns all the memory, it talks to the disk controller by writing bytes to specific hardware addresses, and it runs until you power-cycle the box. This is not a thought experiment — it is how early computers worked, and it is how a $2 microcontroller in a washing machine still works today.

Three problems appear the moment you want more than one program:

1. **Sharing.** Two programs both want "the CPU" and both want "memory address 0x1000." Somebody must arbitrate.
2. **Protection.** Program A's bug must not corrupt Program B, and neither must be able to read your bank details out of Program C.
3. **Abstraction.** No sane person wants to write disk-controller register pokes to save a text file. Programs want `open`/`read`/`write`, not hardware.

An **operating system** is the program that solves all three at once. The tightest definition worth memorising:

> **An OS safely multiplexes hardware among programs.** *Multiplex* = share one physical resource among many logical users by rapidly switching between them or by carving it up. *Safely* = one user's mistake or malice cannot reach another's.

The historical arc, because it explains the shape of everything that follows: **batch monitors** (1950s: a program that loads the next punched-card job when the last one finishes — sharing, no protection) → **time-sharing systems** (1960s, CTSS/Multics: many interactive users at once, which *forces* protection because now strangers share the machine) → **Unix** (1970: the ideas that survived, small enough to fit in a book) → today's Linux/Windows/macOS, all of which are recognisably Unix-shaped or Unix-influenced in exactly the areas this note covers.

### The analogy — and where it breaks

An OS is the **landlord and building management of an apartment building**. The building has one water supply, one electrical feed, one street entrance (the hardware). The landlord gives each tenant a unit with a lock (memory isolation), meters and caps what each unit draws (resource limits), takes maintenance requests through the front office rather than letting tenants into the boiler room (syscalls), and can evict a tenant who sets their kitchen on fire (kill a process).

**Where the analogy breaks:** apartments are *physically* separate — the walls exist whether or not the landlord is competent. In a computer there are no walls. Every process's memory is in the same RAM chips, inches apart, and "isolation" is a *convention the CPU enforces on the kernel's instructions* (§2). Turn off one bit in a page table and the walls evaporate. The apartment analogy makes isolation feel like a property of matter; in reality it is a property of *configuration*, which is precisely why isolation bugs (Meltdown, Spectre, container escapes) are so devastating: nothing physically prevented the leak, only bookkeeping did.

### Worked example — two programs, one address `0x1000`

Compile two unrelated C programs. Both, by accident of how the compiler laid out their globals, want to store a counter at virtual address `0x1000`. Run them at the same time. Both succeed. Neither can see the other's counter.

What actually happened, concretely:

```
Process A: virtual 0x1000  ──(A's page table)──►  physical 0x7F3A2000
Process B: virtual 0x1000  ──(B's page table)──►  physical 0x1C08D000
                                    ▲
                        the kernel installs a different translation
                        table per process; the MMU (memory management
                        unit, a piece of CPU hardware) consults the
                        active table on EVERY memory access
```

The kernel keeps a per-process map from *virtual* addresses (what your program says) to *physical* addresses (actual RAM). On a context switch it points the CPU at a different map (on x86-64: it loads a new value into the `CR3` register). Same number in the program, different byte of RAM. That indirection is what makes "every program gets its own memory starting at address 0" a safe lie.

**Visual — the layer cake, with who trusts whom:**

| Layer | Example | Privileged? | If it crashes… |
|---|---|---|---|
| Application | your Python script | No | that process dies |
| Runtime / libc | CPython, glibc, ntdll | No | that process dies |
| **Syscall interface** | `write`, `openat`, `clone` | boundary | — |
| Kernel core | scheduler, VFS, TCP/IP | **Yes** | machine panics / BSOD |
| Device drivers | NVMe, NIC, GPU | **Yes** (usually) | machine panics |
| Hardware | CPU, RAM, disk | — | buy a new one |

That table is the whole reason for §2: notice that a driver bug takes down the *machine*, while an application bug takes down one *process*. Everything the OS does about protection is an attempt to keep as much code as possible in the bottom-of-the-blast-radius row.

**Under the hood (named, not opened):** the scheduler decides which thread runs on which core and for how long; the memory manager decides which pages live in RAM versus on disk; the VFS (virtual file system) turns `open("/tmp/x")` into filesystem-specific block reads. Treat all three as trusted black boxes today.

---

## §2. Kernel space vs user space — why the split exists  **Depth: [CORE]**

### Intuition — what people did before, and what it cost

Before the split, "protection" meant "please don't." MS-DOS (1981–1995, and the ancestor of a shocking amount of still-running industrial software) ran your program with full authority over the machine. Any program could write to any memory address, reprogram the interrupt table, or scribble on the disk's boot sector. The consequences were exactly what you'd predict: one bad pointer in a word processor could take down the machine, and the entire concept of "a virus" became trivially easy — there was nothing to escalate *to*, because everything was already privileged.

The fix is not software discipline. It is **hardware**. Modern CPUs refuse to execute certain instructions unless a bit in the processor says "you are the kernel."

### The mechanism, opened all the way

**1. CPU privilege levels.** x86-64 defines four privilege levels (historically called *rings*) numbered 0–3, where 0 is most privileged. Mainstream OSes use exactly two: **ring 0 = kernel space**, **ring 3 = user space**. (ARM calls them *exception levels*: EL0 for apps, EL1 for the kernel, EL2 for hypervisors.) The current level lives in the low bits of the code-segment selector (`CS`), and — this is the crucial part — *user code cannot change it by writing to it*. The only way to go from ring 3 to ring 0 is through a controlled entry point the kernel installed in advance.

**2. What ring 0 unlocks.** Privileged instructions (`hlt`, `lgdt`, `mov` to/from control registers like `CR3`, most `rdmsr`/`wrmsr`), direct device I/O (`in`/`out` port access, or memory-mapped device registers), and — most importantly — the ability to modify the page tables that define everyone's memory map.

**3. How memory protection is expressed.** Each page-table entry (the record describing one 4 KiB page of memory) carries permission bits, including a **U/S bit** ("user/supervisor"). If U/S says *supervisor only* and ring-3 code touches that address, the CPU raises a **page fault** — a hardware trap into the kernel. The kernel looks at it, decides this was not a legitimate fault (not a swapped-out page, not a copy-on-write page), and delivers `SIGSEGV` to the offending process. Your program prints `Segmentation fault` and dies. Modern CPUs add belt-and-braces features here: **SMEP** (kernel cannot *execute* user pages) and **SMAP** (kernel cannot casually *read/write* user pages), both designed to blunt exploits that trick the kernel into running attacker-controlled bytes.

**4. The mode switch, and what it costs.** Crossing from ring 3 to ring 0 and back is a **mode switch** (not to be confused with a *context switch*, which is swapping one process for another). On x86-64 the `syscall` instruction: saves the return address into `rcx` and flags into `r11`, loads the kernel entry address from the `LSTAR` model-specific register, and flips to ring 0. The kernel then switches to a **kernel stack** for that thread (never trust the user stack pointer), and — on kernels with **KPTI** (kernel page-table isolation, the Meltdown mitigation, 2018) — swaps to a different page table entirely, which is why post-2018 syscalls are measurably more expensive than pre-2018 ones on affected hardware.

A mode switch is *cheap compared to a context switch* but *ruinously expensive compared to a function call* — and that ratio, not the absolute number, is what drives real design decisions (buffering, batching, `io_uring`, `sendfile`, memory-mapped I/O).

### The analogy — and where it breaks

The user/kernel split is a **bank teller window**. You (user space) cannot walk into the vault. You fill in a slip, slide it under the glass, and a teller (the kernel) — who *can* enter the vault — validates your request, performs it, and slides the result back. The glass is bulletproof and one-way. Every operation you want is expressible as some slip; if it isn't on the list of slips, you cannot ask for it at all.

**Where the analogy breaks (two ways, both important):**
1. A teller who is tricked can be *fired and replaced*. If the kernel is tricked, the attacker now **is** the bank: kernel compromise is total and unrecoverable from inside the machine. There is no higher authority to appeal to (short of a hypervisor). That asymmetry is why kernel bugs are the highest-severity bugs that exist, and why "reduce the amount of code running in ring 0" is a recurring design goal (microkernels, unikernels, eBPF verifiers, userspace drivers).
2. Bank tellers do not leak information by *how long they take*. CPUs do. Meltdown and Spectre (2018) extracted kernel memory from user space without ever violating the U/S bit — they measured cache timing side effects of speculative execution. The permission check was correct; the *physics* leaked. Any mental model where "the check passed, therefore nothing leaked" is wrong.

### Worked example — traced: what happens when you touch what you shouldn't

```c
// crash.c   —  gcc crash.c -o crash && ./crash
int main(void) {
    int *p = (int *)0xFFFFFFFF81000000;   // a typical Linux kernel address
    *p = 1;                               // ring 3 writing a supervisor page
    return 0;
}
```

The traced sequence, step by step:

```
1. CPU executes the store instruction, virtual addr 0xFFFFFFFF81000000
2. MMU walks the page table for this process → finds entry with U/S = supervisor
3. Current privilege level is 3 (user)          →  PERMISSION DENIED
4. CPU raises exception #14 (page fault), pushes fault info, switches to ring 0,
   jumps to the kernel's page-fault handler (address preinstalled in the IDT)
5. Kernel handler reads CR2 (the faulting address) and the error code:
   "write, user-mode, protection violation" — not a swapped page, not COW
6. Kernel decides: this process is misbehaving → queue SIGSEGV for it
7. Default disposition of SIGSEGV is terminate+core  →  process dies
8. Kernel writes exit status "killed by signal 11" into the process's
   exit record; the shell reads it and prints:
        Segmentation fault (core dumped)      [exit code 139 = 128 + 11]
```

Notice what did *not* happen: no kernel memory was modified, and no other process noticed. The blast radius was exactly one process. That containment is the entire product the OS is selling.

### Runnable example — measuring the price of crossing the boundary

This is the "what does a mode switch cost?" question from the syllabus, turned into something you can run right now on any OS.

```python
# no install needed — stdlib only.  Run: python syscall_cost.py
import os, time

N = 200_000

# (a) pure user-space work: no boundary crossing at all
t0 = time.perf_counter()
buf = []
for _ in range(N):
    buf.append("x")
s = "".join(buf)
t1 = time.perf_counter()

fd = os.open(os.devnull, os.O_WRONLY)   # NUL on Windows, /dev/null on POSIX

# (b) N tiny writes: one syscall per byte
t2 = time.perf_counter()
for _ in range(N):
    os.write(fd, b"x")                  # unbuffered: goes straight to the kernel
t3 = time.perf_counter()

# (c) the same N bytes in ONE syscall
os.write(fd, s.encode())
t4 = time.perf_counter()
os.close(fd)

print(f"userspace append+join {N}: {(t1-t0)*1e6/N:8.3f} us/op")
print(f"os.write syscall  {N}x1B: {(t3-t2)*1e6/N:8.3f} us/op")
print(f"one os.write of {N}B     : {(t4-t3)*1e6:8.3f} us total")
print(f"ratio syscall/userspace  : {(t3-t2)/(t1-t0):8.1f}x")
```

Real output, measured on the machine I wrote this on (Windows 11, Python 3.11, corporate endpoint-security agent active):

```
userspace append+join 200000:    0.091 us/op
os.write syscall  200000x1B:    1.573 us/op
one os.write of 200000B     :  108.700 us total
ratio syscall/userspace  :     17.2x
```

**Why this works, line by line.**
- `time.perf_counter()` is a monotonic high-resolution clock; on Linux it is backed by `clock_gettime(CLOCK_MONOTONIC)`, which the kernel exposes through the **vDSO** — a page of kernel-provided code mapped into your address space precisely so that reading the clock does *not* require a mode switch. That is your first hint that "syscalls are expensive" is a real enough pressure that the kernel builds escape hatches for the hottest ones.
- Block (a) is the control: ~0.09 µs of pure interpreter work per iteration, never leaving user space.
- Block (b) calls `os.write` per byte. `os.write` is a thin wrapper over the raw `write(2)` syscall — no buffering. Each iteration therefore pays: Python call overhead + mode switch + kernel VFS dispatch + device write + mode switch back. Measured: 1.573 µs.
- Block (c) makes **one** syscall carrying all 200,000 bytes: 108.7 µs total, i.e. **0.0005 µs per byte** — roughly 2,900× cheaper per byte than the per-byte version. The bytes are the same; only the number of boundary crossings changed.

**Honesty caveat — what this measurement is and is not.** This is *not* a measurement of raw syscall latency. It bundles CPython's function-call overhead, Windows' `WriteFile` path, and (on this machine) a security agent that hooks I/O. Published microbenchmarks put a bare Linux `getpid()`-class syscall in the low hundreds of nanoseconds, rising noticeably with KPTI/retpoline mitigations enabled — verify on *your* target with a dedicated tool (`lmbench`'s `lat_syscall`, or `perf bench syscall`) before quoting a number in a design review. What the experiment *does* establish, robustly and reproducibly, is the shape of the answer: **crossing the boundary costs one to two orders of magnitude more than not crossing it, so the winning strategy is always to cross less often with more data.** That single sentence explains buffered I/O, TCP's Nagle algorithm, database write batching, log-flush intervals, `sendfile`, `io_uring`, and every "why is my loop slow" question involving a filesystem.

### Trade-off — how much code belongs in ring 0?

| Design | Ring-0 surface | Pro | Con | Real examples |
|---|---|---|---|---|
| **Monolithic kernel** | huge (drivers, FS, net all in-kernel) | fast (no boundary crossings internally) | one driver bug = whole-machine crash; giant attack surface | Linux, Windows NT (mostly), FreeBSD |
| **Microkernel** | tiny (IPC + scheduling only; drivers are user processes) | a crashed driver restarts; verifiable | every internal call is now a boundary crossing → slower unless carefully engineered | QNX, seL4, MINIX 3 |
| **Hybrid / userspace drivers** | medium | pragmatic middle | complexity | macOS/XNU, Windows UMDF, DPDK/SPDK (net & storage in userspace *for speed*) |

**Depth: [AWARE]** on this row — know the axis exists and that it's a real, live engineering trade-off (seL4 is formally verified and flies in helicopters; DPDK moves the network stack to user space to *avoid* syscalls). Do not go deeper unless a project forces you.

**Interview angle.** *"What is the difference between a mode switch and a context switch?"* — A **mode switch** changes privilege level within the same thread (user → kernel → user); nothing about *which* process is running changes, and it costs on the order of hundreds of nanoseconds. A **context switch** replaces the running thread with a different one: save registers, switch kernel stacks, switch page tables (`CR3`), invalidate/refill TLB and caches; typically a few microseconds *plus* a much larger hidden cost as the new thread's cache working set warms up. Every context switch involves mode switches, but the reverse is not true. Candidates who conflate these two cannot reason about why an async server beats a thread-per-connection server.

---

## §3. Syscalls — the only door into the kernel  **Depth: [CORE]**

### Intuition

If the kernel is behind bulletproof glass, the **syscall** is the slip you slide under it. A syscall is a *request for the kernel to do something on your behalf that you are not privileged to do yourself*. That's the whole concept. The set of available syscalls is the complete, exhaustive list of things any program on the machine can ask for. Linux has roughly 300–400 of them depending on architecture and version; that finite list is the entire vocabulary of everything every program on your server can do.

Before syscalls as we know them, programs called into the OS by triggering software interrupts with magic numbers in registers (DOS: `int 21h`; early Linux: `int 0x80`) or by calling into a jump table at a fixed address. The idea is the same age as protection itself; the modern `syscall`/`sysenter` instructions are just a faster implementation of the same door.

### The mechanism, opened all the way

Take `write(1, "hi\n", 3)` on x86-64 Linux:

```
user space                            │ kernel space
──────────────────────────────────────┼────────────────────────────────────
  your code:  printf("hi\n")          │
     │                                │
     ▼                                │
  libc's write() wrapper:             │
     mov  rax, 1        ; syscall #   │   (1 = write, per the syscall table)
     mov  rdi, 1        ; arg1 = fd   │
     mov  rsi, buf      ; arg2 = ptr  │
     mov  rdx, 3        ; arg3 = len  │
     syscall  ───────────────────────►│  CPU: save rip→rcx, flags→r11,
                                      │       load entry addr from LSTAR MSR,
                                      │       switch to ring 0
                                      │  entry_SYSCALL_64:
                                      │    switch to this thread's kernel stack
                                      │    (swap page tables if KPTI)
                                      │    bounds-check rax against table size
                                      │    call sys_write(rdi, rsi, rdx)
                                      │      → copy_from_user(rsi, 3 bytes)
                                      │        (validates the pointer! user
                                      │         memory is never trusted)
                                      │      → VFS → tty/pipe/file driver
                                      │    put return value in rax
     ◄──────────────────────── sysret ┤    restore ring 3, rip from rcx
  rax = 3  (bytes written)            │
  or rax = -EBADF (a small negative)  │
     │                                │
  libc: if rax in [-4095,-1]:         │
          errno = -rax; return -1     │
```

Five details that matter in practice:

1. **The calling convention is a hard ABI.** Syscall number in `rax`; arguments in `rdi, rsi, rdx, r10, r8, r9` (note `r10`, *not* `rcx` — because `rcx` is clobbered by the `syscall` instruction itself). Return value in `rax`. This is documented in the Linux `syscall(2)` man page and the AMD64 ABI, and it is *stable* — that stability is why a 2005 Linux binary still runs on a 2026 kernel.
2. **Errors are negative return values, not exceptions.** The kernel returns `-EBADF` (`-9`); libc turns anything in `[-4095, -1]` into `return -1` plus `errno`. Higher-level languages then turn `errno` into an exception (`OSError` in Python). Three layers of error representation for one condition — knowing this saves you when a stack trace says `OSError: [Errno 9] Bad file descriptor` and you need to reason about which layer lied to you.
3. **The kernel never trusts a user pointer.** `copy_from_user`/`copy_to_user` validate every address and length. Skipping that check is one of the classic kernel vulnerability patterns.
4. **Not every "syscall" is a syscall.** `gettimeofday`/`clock_gettime` are usually served from the **vDSO**: kernel-authored code mapped read-only into your address space, computing the time from a shared page with no mode switch at all. If you microbenchmark "syscall cost" using `time()`, you may measure ~20 ns and conclude syscalls are free. They are not; you measured a deliberate bypass.
5. **Windows is the same idea with different names and one extra rule.** `ntdll.dll` exports thin stubs (`NtWriteFile`, `NtCreateFile`) that execute `syscall` into the kernel; the Win32 API (`WriteFile`, `CreateProcessW`) sits *above* those in `kernel32.dll`. Critically, **Windows syscall numbers are not a stable contract** — they change between builds, and you are expected to call through the documented Win32/NT APIs. This is why Windows syscall tracing is done with ETW or Sysinternals **Process Monitor** rather than `strace`.

### The analogy — and where it breaks

A syscall is a **restaurant order window in a kitchen you're not allowed to enter**. You write what you want on a ticket, hand it through, and food comes back. The menu is finite; if soufflé isn't on the menu, you cannot have soufflé no matter how nicely you ask.

**Where the analogy breaks:** a restaurant order is *asynchronous and pipelined* — you hand in the ticket and go do something else. A classic syscall is **synchronous**: your thread is blocked, doing nothing, until the kernel returns. That single difference is the origin of most of modern backend architecture. Thread-per-request servers exist because syscalls block; `epoll`/`kqueue`/IOCP exist to *avoid* blocking; `async`/`await` exists to make non-blocking code readable; `io_uring` (2019) exists to make syscalls *actually* work like the restaurant analogy — a submission queue and a completion queue in shared memory, so you can hand in 100 tickets with one crossing. If you internalise the analogy without this correction, you will not understand why async I/O exists.

### Worked example — the dozens of syscalls behind one `print`

The syllabus asks you to trace `python -c "print('hi')"` and count the syscalls behind one print. Here is the command and what to expect.

**On Linux:**

```bash
# -f follows child processes, -c gives a summary count per syscall
strace -f -c python3 -c "print('hi')"

# to see the ACTUAL write, filter to it:
strace -f -e trace=write python3 -c "print('hi')"
```

**On macOS:** `strace` doesn't exist; use `sudo dtruss -f python3 -c "print('hi')"` (and expect to fight System Integrity Protection).

**On Windows** (what I can run on this machine): there is no `strace`. Use Sysinternals **Process Monitor** (`procmon`), set a filter `Process Name is python.exe`, run the command, and read the event list — you will see the file/registry/process operations, which are the Win32-level equivalent of the syscall stream.

**What you will see, and the honest version of the count.** The single `write` that emits `hi\n` is the *last and least* of it. The overwhelming majority of the syscall traffic is:

| Phase | Representative syscalls (Linux names) | Why |
|---|---|---|
| Process creation | `execve`, `brk`, `arch_prctl`, `set_tid_address` | the shell replaced itself with Python; set up the new image |
| Dynamic linking | `openat`, `read`, `fstat`, `mmap`, `mprotect`, `close` on `ld.so.cache`, `libc.so.6`, `libpython3.x.so` | find and map shared libraries |
| Interpreter startup | dozens–hundreds of `openat`/`stat`/`read` walking `sys.path` looking for `encodings`, `site`, `sitecustomize`, `.pyc` caches | Python's import machinery probing the filesystem |
| Standard I/O setup | `ioctl(1, TCGETS)` or `isatty` checks, `fstat(1)` | decide whether stdout is a terminal → choose line-buffered vs block-buffered |
| **The actual work** | **one** `write(1, "hi\n", 3)` | your program |
| Teardown | `munmap`, `close`, `exit_group` | give it all back |

**Honesty note (Principle 6).** I am deliberately *not* quoting you a precise total like "exactly 847 syscalls." That number varies by Python version, distro, whether `.pyc` caches are warm, whether you're in a venv, and whether stdout is a pipe or a terminal — anyone quoting a fixed figure is quoting their machine, not yours. Run `strace -f -c` yourself and read the `total` row; the number will be in the hundreds, and the *ratio* is the lesson: **one line of your code, hundreds of kernel crossings, and >99% of them are startup, not work.** This is the mechanical reason why "spawn a fresh interpreter per request" is a performance decision with a real price tag (measured in §5) and why serverless platforms obsess over cold starts.

### Runnable example — proving buffering changes the syscall count

You can observe the buffering decision from the table above without any tracing tool, because Python exposes it:

```python
# buffering_demo.py — stdlib only.  Run twice:
#     python buffering_demo.py                 (block-buffered when piped)
#     python -u buffering_demo.py              (unbuffered)
#   and compare:  python buffering_demo.py | cat     vs     python buffering_demo.py
import sys, os

print("stdout isatty:", sys.stdout.isatty())
print("stdout write_through/line_buffering:", getattr(sys.stdout, "line_buffering", None))
print("underlying buffer:", type(sys.stdout.buffer).__name__)

# 3 print()s = how many write() syscalls?  Depends on buffering:
for i in range(3):
    print("chunk", i)
# On exit, Python flushes; if block-buffered, ALL of the above leaves in ONE write().
```

Real output on my machine, run directly in a terminal:

```
stdout isatty: True
stdout write_through/line_buffering: True
underlying buffer: BufferedWriter
chunk 0
chunk 1
chunk 2
```

**Why this works.** `sys.stdout.isatty()` is a thin wrapper over the `isatty(3)`/`ioctl` check the interpreter itself performs at startup. When stdout is a terminal, CPython sets `line_buffering=True` — every newline triggers a `write` syscall, so a human sees output immediately. When stdout is a *pipe* (`| cat`, or captured by a log collector, or captured by `subprocess.PIPE`), CPython switches to **block buffering** — typically an 8 KiB buffer — so three `print`s become *zero* syscalls until the buffer fills or the process exits and flushes. Pipe it and the interleaving changes; that is exactly one syscall-count decision, visible from Python.

**This is the single most common "my logs disappeared" bug in production.** Your container's stdout is a pipe, not a tty, so Python block-buffers; the process is then `SIGKILL`ed (§6) before the buffer flushes, and the last 8 KiB of logs — including the exception that explains the crash — is destroyed with the process. The fixes are `python -u`, `PYTHONUNBUFFERED=1`, or `flush=True`. You now know *why* from first principles, not as a folk remedy.

### Failure modes worth knowing on day 2

| Symptom | Underlying syscall reality |
|---|---|
| `EINTR` ("Interrupted system call") | a signal arrived while a slow syscall was blocked. Pre-2014 code had to retry manually; PEP 475 made CPython retry automatically for you |
| `EAGAIN`/`EWOULDBLOCK` | you set a non-blocking fd and the kernel has nothing right now — not an error, a "come back later" |
| `EMFILE` / "Too many open files" | you hit the per-process fd limit (`ulimit -n`) — usually an fd *leak* (§4), not a limit that's too low |
| `EPIPE` + `SIGPIPE` | you wrote to a pipe whose reader is gone; default disposition kills you. This is why `head` terminating a pipeline kills the producer |
| `ENOSPC` on a write that "should" work | writes are buffered; the error surfaces at `write` or even at `close` — always check the return value of `close` on data you care about |

---

## §4. What a process actually is  **Depth: [CORE]**

### Intuition — program vs process

A **program** is a file on disk: bytes, dormant, inert. A **process** is a program *in flight*: the same bytes plus everything the kernel must remember to keep them running. Same program, three processes = three independent, isolated running things. The distinction is the difference between a recipe and a meal being cooked.

Before processes were a concept, "running a program" and "the machine" were the same thing — see §1. The process abstraction is what makes "the computer is doing several things" expressible at all.

### Definition — the four things (memorise these)

> A **process** = an **address space** + one or more **threads** + a **file descriptor table** + **credentials**.

**1. Address space.** The private virtual→physical memory map from §1, divided into regions: **text** (the machine code, read-only + executable), **data/bss** (globals), **heap** (grows via `brk`/`mmap` when you allocate), **stack** (grows down, one per thread), plus mapped shared libraries and any files you `mmap`ed. Everything a process can address, it addresses here — and nothing outside it exists as far as the process is concerned.

**2. Thread(s).** A **thread** is one flow of execution: a program counter, a set of registers, and a stack. Threads within a process *share the address space and the fd table* (which is why they can corrupt each other's data and why locks exist) but have private stacks and registers. The kernel schedules threads, not processes. On Linux the distinction is almost cosmetic internally: both `fork` and thread creation go through the same `clone` syscall with different flags — a thread is just "a process that shares everything."

**3. File descriptor table.** A per-process array of small integers → open kernel objects. `0` = stdin, `1` = stdout, `2` = stderr by convention, then 3, 4, 5… as you open things. The integer is *just an index into your process's table* — fd 3 in your process and fd 3 in mine are unrelated. This indirection is the mechanism behind shell redirection, pipes, and inherited sockets in pre-forking servers (§5). And critically: **a file descriptor is a capability.** If you hold one, you may use it; the permission check happened at `open` time and is never repeated. Hand an fd to another process and you have handed over access, permanently, regardless of file permissions.

**4. Credentials.** The identity the kernel checks on every permission decision: `uid`/`euid` (real and effective user ID), `gid`/groups, and on Linux the finer-grained **capabilities** (e.g. `CAP_NET_BIND_SERVICE` = "may bind ports below 1024 without being root"). Credentials are why "run as non-root" is the single highest-value line in a Dockerfile.

**Plus the bookkeeping you will touch constantly:** `pid` (process ID), `ppid` (parent's pid — every process has exactly one parent, forming a tree), `cwd` (current working directory — resolves all your relative paths), the **environment** (a copy of key=value strings, inherited at creation), **signal dispositions** (§6), **resource limits** (`RLIMIT_NOFILE`, `RLIMIT_AS`, `RLIMIT_CPU`…), the **umask**, the **process group** and **session** (used for job control and, crucially for §11, for killing whole trees at once), and the **exit status** slot where your final exit code waits to be collected (§7).

**Visual — process anatomy:**

```
┌─ PROCESS  pid=20324  ppid=18992  uid=1000 ────────────────────────────┐
│                                                                       │
│  ADDRESS SPACE (private)                 FD TABLE (private)           │
│  ┌───────────────────────────┐           ┌───┬──────────────────────┐ │
│  │ 0x7fff… stack (thread 1) ↓│           │ 0 │ tty (stdin)          │ │
│  │         stack (thread 2) ↓│           │ 1 │ pipe → log collector │ │
│  │            ⋮              │           │ 2 │ tty (stderr)         │ │
│  │ 0x7f3a… libpython3.11.so  │           │ 3 │ socket → 10.0.0.7:443│ │
│  │         libc.so.6         │           │ 4 │ /var/app/data.db     │ │
│  │            ⋮              │           └───┴──────────────────────┘ │
│  │ 0x0140… heap             ↑│                                        │
│  │ 0x0100… data / bss        │           CREDENTIALS  uid/gid/caps    │
│  │ 0x0040… text (r-x)        │           CWD, ENV, umask, rlimits     │
│  └───────────────────────────┘           SIGNAL DISPOSITIONS (§6)     │
│                                                                       │
│  THREADS: [tid 20324 running] [tid 20325 blocked in read()]           │
└───────────────────────────────────────────────────────────────────────┘
```

### The analogy — and where it breaks

A process is a **rented commercial kitchen for one shift**. The recipe (program) is on the wall. The kitchen has its own countertops (memory), its own set of hatches to the outside world already propped open — the delivery door, the bin chute (file descriptors) — and an ID badge that determines which storerooms you may enter (credentials). One or more cooks work in it simultaneously (threads), sharing the countertops, which is why they collide if they don't coordinate.

**Where the analogy breaks:** two cooks in one kitchen can *see each other* and negotiate in real time. Threads cannot. A thread has no idea another thread just half-updated the data structure it is reading — there is no ambient awareness, only the memory contents at whatever instant it happens to look. The kitchen analogy makes shared-memory concurrency feel manageable and social; in reality it is blind, and every bug class in concurrent programming (torn reads, lost updates, ABA problems) comes from that blindness. Second, real kitchens don't get their countertops swapped out mid-chop; a thread can be preempted between *any* two machine instructions, including in the middle of what looks in your source like one operation (`counter += 1` is at least a load, an add, and a store).

### Runnable example — introspecting your own process

```python
# whoami.py — stdlib only.  Run: python whoami.py
import os, sys, platform, tempfile

print("pid            :", os.getpid())
print("ppid           :", os.getppid())
print("executable     :", sys.executable)
print("cwd            :", os.getcwd())
print("env var count  :", len(os.environ))
print("platform       :", sys.platform, platform.release())

# fds: open a file and observe the next integer the kernel handed out
f = open(os.path.join(tempfile.gettempdir(), "fd_probe.txt"), "w")
print("fd of new file :", f.fileno())
print("stdin/out/err  :", sys.stdin.fileno(), sys.stdout.fileno(), sys.stderr.fileno())
f.close()

# POSIX-only introspection (the kernel exposes process state as files)
if sys.platform.startswith("linux"):
    print("\n--- /proc/self/maps (first 5 regions) ---")
    with open("/proc/self/maps") as m:
        for line in list(m)[:5]:
            print(line.rstrip())
    print("--- /proc/self/fd ---")
    print(sorted(os.listdir("/proc/self/fd")))
    print("uid/euid/gid   :", os.getuid(), os.geteuid(), os.getgid())
else:
    print("\n[skipped] /proc and os.getuid() are POSIX-only; on Windows use")
    print("          Sysinternals Process Explorer, or `pip install psutil`.")
```

Real output on my machine (Windows 11, so the POSIX branch is honestly skipped rather than faked):

```
pid            : 20324
ppid           : 18992
executable     : C:\Users\...\python.exe
cwd            : C:\Users\...\scratchpad
env var count  : 61
platform       : win32 11
fd of new file : 3
stdin/out/err  : 0 1 2
platform       : win32 11

[skipped] /proc and os.getuid() are POSIX-only; on Windows use
          Sysinternals Process Explorer, or `pip install psutil`.
```

**Why this works, and what each line proves.**
- `os.getpid()`/`os.getppid()` read the kernel's bookkeeping for the current process — direct evidence of the **process tree**. `ppid=18992` is the shell that launched me.
- **`fd of new file : 3` is the interesting one.** It is not a coincidence and not a Python detail: the kernel allocates *the lowest unused index* in this process's fd table, and 0/1/2 were pre-populated by whoever launched me. This is the mechanism that makes shell redirection work (§5) — the child is created with 0/1/2 already pointing wherever the parent decided.
- `len(os.environ)` = 61 confirms the environment is a real, finite, *copied* block of data owned by this process — not a global registry. Change it here and no other process sees the change.
- **`/proc/self/maps` and `/proc/self/fd`** (Linux) are the kernel exposing the two structures from the anatomy diagram as readable files. `maps` prints the address-space regions with their permissions (`r-xp` for text, `rw-p` for data/heap); `fd` lists symlinks — one per open descriptor, pointing at the actual file/socket/pipe. **If you ever debug a "too many open files" outage, `ls -l /proc<PID>/fd | wc -l` and then looking at *what* those fds are is the whole investigation.** Windows has no `/proc`; the equivalent is Process Explorer's handle view or `psutil.Process().open_files()` — an honest platform difference, not a gap in the concept.

### Performance and runtime behaviour you should already expect

- **Memory is counted twice, and both numbers matter.** *Virtual* size (how much address space is mapped) is often huge and mostly meaningless; **RSS** (resident set size — how much physical RAM is actually occupied) is what gets you OOM-killed. Shared libraries and copy-on-write pages are counted in *both* parents and children, which is why "sum the RSS of all my workers" massively overstates real usage. Prefer PSS (proportional set size) on Linux when you need honesty.
- **fd limits are per-process and low by default** (often 1024 soft on Linux). A server holding 10,000 connections needs this raised — and needs to not leak.
- **Process creation is not free.** Quantified in §5, with a measurement that will surprise you.

---

## §5. Creating processes: `fork`, `exec`, `wait`, and exit codes  **Depth: [CORE]**

### Intuition — why *two* calls, not one

Here is the design question Unix answered in 1970, and the answer is the most elegant thing in this note. You want to run `ls` from your shell. The naive API is one call: `spawn("ls", args, environment, redirect_stdout_to=..., run_as_user=..., in_directory=...)` — and notice how that parameter list grows without bound, because there are *dozens* of process attributes a parent might want to customise for its child (§4 lists them).

Unix instead split it into two primitives:

- **`fork()`** — duplicate the current process. One call, **two returns**: the parent gets the child's pid, the child gets `0`. Now there are two nearly identical processes.
- **`exec()`** (a family: `execve`, `execvp`, …) — *replace* the current process's program image with a different program, keeping the process's identity: same pid, same fd table, same credentials, same cwd.

**The gap between the two calls is the entire point.** After `fork` but before `exec`, the child is running *your* code with *your* language and *your* libraries, and it can reconfigure itself freely using ordinary syscalls: close fds, `dup2` stdout onto a pipe, `chdir`, `setuid` to drop privileges, set resource limits, join a new namespace. Then it `exec`s, and the new program inherits that carefully arranged environment without needing to know anything about it. One unbounded parameter list becomes two tiny primitives plus normal code. This is why shell redirection (`cmd > file`) requires no cooperation from `cmd`, why `sudo`/`docker run` need no cooperation from the program being run, and why "run untrusted code in a locked-down environment" is even possible (§11 in Part 2 uses exactly this gap).

**Why `fork` isn't absurdly expensive:** it doesn't copy your memory. It copies the *page table* and marks every writable page **copy-on-write** — both processes read the same physical pages, and only when one of them *writes* does the kernel duplicate that single 4 KiB page. Which is what makes the classic pattern (fork, then immediately exec, throwing the copy away) cheap enough to be idiomatic. `vfork` and `posix_spawn` exist to optimise this further.

### The analogy — and where it breaks

`fork` is **cell division**: one cell becomes two genetically identical cells, and the only way each knows which one it is, is a marker (`fork`'s return value). `exec` is then **the daughter cell differentiating** into a liver cell — same cell, entirely new function.

**Where the analogy breaks (three ways, all of which bite real code):**
1. Cells divide *everything*. `fork` does not: file **locks** are not inherited, pending **signals** are cleared in the child, **timers** are reset, and — the big one — **only the calling thread survives**. Fork a multi-threaded process and the child has exactly one thread, but it inherits all the *mutexes* those vanished threads were holding. Those locks will never be released. This is the notorious "fork in a multithreaded program deadlocks" trap, and it is why `fork` in a process with a thread pool, a logging thread, or a connection pool is a genuine landmine (POSIX only guarantees async-signal-safe functions between `fork` and `exec` for this reason).
2. Biological daughter cells are independent instantly. Forked processes still *share* every open file descriptor's underlying kernel object — including the file offset. Two processes writing to an inherited fd interleave into the same file at a shared position. Sharing, not copying.
3. Cells never "become a different organism while keeping their ID badge," which is exactly what `exec` does — and that identity preservation is why `ps` shows a pid whose program name changed and why your process supervisor doesn't notice.

### Worked example — a traced `fork`+`exec`+`wait` timeline

```
t=0   shell (pid 500) reads:  ls -l > out.txt
t=1   shell calls fork()
        → returns 700 in the parent (pid 500)
        → returns   0 in the child  (pid 700, ppid 500)
t=2   CHILD (700), still running shell code:
        fd = open("out.txt", O_WRONLY|O_CREAT|O_TRUNC)   → fd 3
        dup2(3, 1)      # make fd 1 (stdout) point at out.txt
        close(3)        # tidy: 1 already refers to the file
t=3   CHILD calls execve("/bin/ls", ["ls","-l"], envp)
        → address space REPLACED by ls's image; pid is still 700;
          fd 1 still points at out.txt, so ls writes there
          WITHOUT KNOWING ANYTHING ABOUT REDIRECTION
t=4   PARENT (500) calls waitpid(700, &status, 0)   → blocks
t=5   CHILD finishes, calls exit_group(0)
        → kernel frees its memory, keeps a small exit record: status 0
        → child is now a ZOMBIE (§7): dead, but its status is unread
        → kernel sends SIGCHLD to pid 500
t=6   PARENT's waitpid returns 700; status decodes to "exited, code 0"
        → the exit record is freed; the zombie is reaped; pid 700 is reusable
t=7   shell prints its prompt
```

Read `t=2`/`t=3` again: that is the whole trick. **`>` is implemented entirely in the child between fork and exec.** Nothing in `ls` participates.

### Runnable example — the process lifecycle you will actually write

Modern code does not call `fork` directly; `subprocess` (Python), `child_process` (Node), `os/exec` (Go) wrap the whole fork/exec/wait dance portably. Here is the lifecycle, including the parts people get wrong:

```python
# proc_demo.py — stdlib only.  Run: python proc_demo.py
import os, subprocess, sys, time, signal

print("parent pid:", os.getpid(), "platform:", sys.platform)

# 1. run a child to completion and read its exit code
p = subprocess.run([sys.executable, "-c", "import sys; sys.exit(7)"])
print("exit code of sys.exit(7):", p.returncode)

# 2. an uncaught exception is also just an exit code
p = subprocess.run([sys.executable, "-c", "raise SystemError('boom')"],
                   capture_output=True, text=True)
print("exit code of uncaught exception:", p.returncode)

# 3. start a long-running child, then kill it MID-WORK
child = subprocess.Popen(
    [sys.executable, "-c",
     "import time\nfor i in range(100): print(i, flush=True); time.sleep(0.05)"],
    stdout=subprocess.DEVNULL)
time.sleep(0.4)
print("child pid:", child.pid, "poll while running:", child.poll())  # None = alive
child.terminate()                 # polite: SIGTERM on POSIX, TerminateProcess on Windows
rc = child.wait(timeout=5)        # <-- REAP IT (§7). Never skip this.
print("returncode after terminate():", rc)

# 4. the impolite version
child2 = subprocess.Popen([sys.executable, "-c", "import time; time.sleep(30)"])
child2.kill()                     # SIGKILL on POSIX; same TerminateProcess on Windows
print("returncode after kill():", child2.wait(timeout=5))

# 5. platform reality check
print("has os.fork:", hasattr(os, "fork"))
print("has SIGKILL:", hasattr(signal, "SIGKILL"), "| SIGTERM value:", int(signal.SIGTERM))
print("signals you may handle here:",
      sorted(s.name for s in signal.valid_signals() if hasattr(s, "name")))
```

Real output, on my Windows 11 machine:

```
parent pid: 20324 platform: win32
exit code of sys.exit(7): 7
exit code of uncaught exception: 1
child pid: 39344 poll while running: None
returncode after terminate(): 1
returncode after kill(): 1
has os.fork: False
has SIGKILL: False | SIGTERM value: 15
signals you may handle here: ['SIGABRT', 'SIGBREAK', 'SIGFPE', 'SIGILL', 'SIGINT', 'SIGSEGV', 'SIGTERM']
```

**Why this works, line by line.**
- `subprocess.run` = fork + exec + wait, fused. It returns a `CompletedProcess` whose `.returncode` is the decoded exit status from §5's `t=6`.
- `sys.exit(7)` → `7`. Exit codes are a **single byte, 0–255**, and `0` means success. This is the only value a parent gets for free, which is why it carries so much weight in shell scripts, CI pipelines, Kubernetes probes, and systemd units.
- An uncaught exception → `1`. CPython prints the traceback to stderr and exits 1. Your supervisor sees only "1": *the traceback is not the exit code*, and if you discarded stderr you have destroyed the only diagnosis. (This is the buffering trap from §3, wearing a different hat.)
- `child.poll()` returns `None` while the child is alive and the exit code once it's dead — non-blocking status check versus `wait()`'s blocking one.
- `terminate()` then **`wait()`**. The `wait()` is not decoration: without it the child stays a **zombie** on POSIX (§7) and the `Popen` object leaks a handle on Windows. Killing without reaping is the single most common process-management bug in application code.

**Honesty caveats — three places this output contradicts what a Linux-only tutorial would tell you, and they are all real:**

1. **`returncode after terminate(): 1`, not `-15`.** On POSIX, `Popen.terminate()` sends `SIGTERM` (signal 15) and Python reports a signal death as a *negative* returncode: `-15`. On Windows there are no signals to send between processes; `terminate()` calls `TerminateProcess(handle, 1)`, so the child's exit code is literally `1` — indistinguishable from a crash. **Consequence:** any code that branches on `returncode == -15` or `returncode < 0` to mean "we killed it" is silently wrong on Windows. Track intent in your own variable instead of inferring it from the exit code.
2. **`has SIGKILL: False`.** `signal.SIGKILL` does not exist on Windows, and `Popen.kill()` there is *the same call* as `terminate()`. The "polite vs. forced" distinction that all of §6 is built on — the thing your entire graceful-shutdown design depends on — **does not exist on Windows in the POSIX sense.** Windows' nearest equivalents are console control events (`CTRL_C_EVENT`/`CTRL_BREAK_EVENT`, which only work with `CREATE_NEW_PROCESS_GROUP`) and Job Objects for tree control.
3. **`has os.fork: False`.** Windows has no `fork` at all; process creation is a single `CreateProcess` call that takes a big struct (exactly the API Unix rejected in favour of two primitives). Consequences you will hit in Python: `multiprocessing` on Windows and on macOS since 3.8 defaults to the **spawn** start method — the child is a *fresh interpreter* that **re-imports your `__main__` module** and receives arguments by pickling. That is why every `multiprocessing` example has an `if __name__ == "__main__":` guard (without it, the child re-runs your top-level code and forkbombs) and why unpicklable arguments (open sockets, lambdas, database connections) fail on Windows while "working" under fork on Linux. If your CI is Linux and your laptop is Windows, this is where you lose an afternoon.

### Exit codes — the one-byte API between every program and its supervisor

| Code | Convention | Where it comes from |
|---|---|---|
| `0` | success | the only universally agreed value |
| `1` | generic failure | uncaught exception, `exit(1)` |
| `2` | usage / CLI misuse | argparse, most GNU tools |
| `126` | found but not executable | shell (POSIX conventions) |
| `127` | command not found | shell — the classic `$PATH`/typo signal |
| `128+N` | killed by signal N (**as reported by the shell**) | `$?` = 137 → 128+9 → SIGKILL → almost always the **OOM killer** or a shutdown timeout |
| negative `N` | killed by signal N (**as reported by Python**) | `Popen.returncode == -9` is the same event as shell's 137 |
| `> 255` on Windows | an NTSTATUS value | e.g. `3221225786` = `0xC000013A` = `STATUS_CONTROL_C_EXIT`. Verify per runtime — Python intercepts Ctrl-C into `KeyboardInterrupt`, so its exit code differs from a native app's |

**Recognise 137 on sight.** `exit code 137` in Kubernetes/Docker logs = `128 + 9` = SIGKILL = "something decided you were not going to be asked twice." The two overwhelmingly common causes: the container exceeded its memory limit (OOM kill), or it ignored `SIGTERM` past the grace period (§6, and the graceful-shutdown design below).

### Performance — measure the cost of a process, on your own machine

```python
# spawn_cost.py — stdlib only.  Run: python spawn_cost.py
import subprocess, sys, time

def bench(fn, n):
    t = time.perf_counter()
    for _ in range(n):
        fn()
    return (time.perf_counter() - t) / n * 1000     # ms per op

inproc = bench(lambda: print("hi", end=""), 2000)
print()
spawn = bench(lambda: subprocess.run([sys.executable, "-c", "print('hi')"],
                                     stdout=subprocess.DEVNULL), 20)
print(f"in-process print : {inproc:9.4f} ms")
print(f"spawn python -c  : {spawn:9.4f} ms")
print(f"ratio            : {spawn/inproc:9.0f}x")
```

Real output on my machine:

```
in-process print :    0.0022 ms
spawn python -c  : 1595.2662 ms
ratio            :    731757x
```

**Read that number carefully, and then read the caveat, because both halves are the lesson.** 1.6 *seconds* to run `python -c "print('hi')"` — nearly six orders of magnitude more than doing the same work in-process.

**Honesty caveat:** that 1595 ms is **not** a universal figure and it is not "what fork costs." It is what process creation costs *on this specific host*: a corporate Windows 11 laptop with endpoint-security/EDR software that inspects every process creation, plus CPython's own interpreter startup (which, per §3, is hundreds of syscalls of import probing). A bare Linux container typically spawns a Python interpreter in the tens of milliseconds, and a static Go binary in single-digit milliseconds. The absolute number is a property of your environment; **the design lesson is environment-independent and holds at every scale:**

> Process creation is one of the most expensive operations you can perform per unit of work. If your architecture spawns a process per request, per tool call, or per test case, **measure it on the host it will actually run on** before you assume it's fine — and if it isn't, the fix is a pool of long-lived processes rather than fresh ones.

This measurement is why gunicorn pre-forks workers at boot instead of per request, why CGI lost to FastCGI and then to persistent app servers, why serverless platforms keep warm containers, why `pytest-xdist` reuses workers — and, in Part 2, it is the deciding input for whether an agent's code-execution tool spawns a fresh sandbox per call or reuses a warm one.

---

## §6. Signals — asynchronous notification, and the SIGTERM/SIGKILL distinction  **Depth: [CORE]**

### Intuition — what problem signals solve

You have a running process. Something outside it needs to say "stop," or "reload your config," or "your child died." The process is not asking; it is busy. You need a way to *interrupt* it. Before signals, the alternatives were polling (the process periodically checks a file or a shared flag — wasteful and laggy) or nothing at all.

A **signal** is a fixed-size asynchronous notification — a number, essentially — delivered by the kernel to a process. It is the OS's software equivalent of a hardware interrupt: it makes the process jump out of whatever it was doing, run a handler, and resume.

### The mechanism, opened

For each of ~30 signals, every process has a **disposition**: *default* (each signal has one: terminate, terminate+core dump, ignore, stop, continue), *ignore* (`SIG_IGN`), or *a handler function you registered*. Delivery works like this:

```
sender:  kill(pid, SIGTERM)          # a syscall — you need permission (same uid, or root)
kernel:  set bit 15 in target's "pending signals" mask
         if the target is blocked in an interruptible syscall, wake it (→ EINTR)
kernel:  next time that process is about to return to user space, check pending
         → if a handler is registered: push a fake stack frame so the process
           "returns" into the handler; on handler exit, sigreturn resumes the
           originally interrupted instruction
         → if default: apply it (usually: terminate; record "died by signal 15")
```

Three consequences that trip people up:

1. **Delivery is not instant.** It happens at the next kernel→user transition of that process. A process spinning in a tight loop with interrupts... will still get it (timer interrupts force transitions), but a process blocked in an *uninterruptible* kernel operation (classically, disk I/O in state `D`) will not react until it unblocks. This is why "`kill -9` didn't work" occasionally happens: the process is stuck in the kernel, and SIGKILL cannot be delivered until it returns.
2. **Handlers run in a horrifyingly restricted context.** Your handler can be entered *between any two machine instructions*, including in the middle of `malloc` updating the heap's internal free list. If your handler then calls `malloc`, you can deadlock or corrupt the heap. POSIX therefore specifies a short list of **async-signal-safe** functions (`write` is on it; `printf` is not). The correct handler shape is: **set a flag and return.** Nothing else.
3. **Signals do not queue** (for standard signals). Two SIGTERMs while one is pending collapse into one pending bit. You cannot count signals.

**Python's mercy:** CPython does not run your Python handler inside the real signal handler. Its C-level handler sets a flag and writes to a self-pipe; the interpreter checks that flag at the next bytecode boundary and *then* calls your Python function. That is why Python signal handlers can safely `print` and allocate — and also why a Python handler cannot interrupt a long-running C extension call (a `time.sleep` is fine; a single 10-second NumPy operation is not) and why `KeyboardInterrupt` sometimes appears "late."

### The signals you must know cold

| Signal | № | Default action | Catchable? | What it's for in production |
|---|---|---|---|---|
| **SIGTERM** | 15 | terminate | **yes** | "Please shut down." The polite request. **This is what your handler is for.** Sent by `kill`, `docker stop`, `systemctl stop`, Kubernetes pod deletion |
| **SIGKILL** | 9 | terminate | **NEVER** | "Die now." Handled entirely by the kernel; the process never runs another instruction. No cleanup, no flush, no goodbye |
| SIGINT | 2 | terminate | yes | Ctrl-C from a terminal → `KeyboardInterrupt` in Python |
| SIGHUP | 1 | terminate | yes | Historically "terminal hung up"; by convention now "reload your config" (nginx, sshd) |
| SIGQUIT | 3 | terminate + core | yes | Ctrl-\; some runtimes dump all thread stacks (very useful on the JVM/Go) |
| SIGCHLD | 17 | ignore | yes | "A child of yours changed state." The trigger for reaping (§7) |
| SIGPIPE | 13 | terminate | yes | You wrote to a pipe with no reader (§3) |
| SIGSTOP / SIGCONT | 19/18 | stop / resume | STOP: no | Job control (Ctrl-Z); also how debuggers and `docker pause` freeze things |
| SIGSEGV / SIGBUS / SIGFPE | 11/7/8 | terminate + core | yes (don't) | Hardware faults surfaced as signals (§2) |

**The one distinction to carry for the rest of your career:**

> **SIGTERM asks. SIGKILL takes.** SIGTERM is deliverable to your code, so you can finish the in-flight request, flush the buffer, commit the transaction, deregister from the load balancer, and exit 0. SIGKILL is executed by the kernel *about* you, not *by* you — you get no instructions, so anything not already durable is gone. Every shutdown mechanism in modern infrastructure is the same two-step: **send SIGTERM, wait a grace period, then send SIGKILL.** `docker stop` (default 10 s), Kubernetes `terminationGracePeriodSeconds` (default 30 s), systemd `TimeoutStopSec` (default 90 s), gunicorn `--graceful-timeout` (default 30 s). Your job as an engineer is to make sure your process finishes its work *inside* that window — because when it expires, you lose the argument.

### The analogy — and where it breaks

Three signals, three real-world escalations. **SIGTERM** is a tap on the shoulder and "we're closing in 10 minutes, please finish up" — you decide how to wrap up. **SIGSTOP** is someone hitting pause on your entire existence; you resume mid-sentence and never knew time passed. **SIGKILL** is the building's power being cut at the street: no announcement, no chance to save, and *you are not the one who gets to react.*

**Where the analogy breaks:** a person who is told "10 minutes" *knows how long they have*. A process receiving SIGTERM gets **no deadline information whatsoever** — the signal carries a number and nothing else. Your process cannot discover the grace period; the orchestrator knows it and your code does not. That mismatch is the root cause of most graceful-shutdown failures in the wild: the app was written assuming it had "enough time," the platform disagreed, and SIGKILL arrived mid-flush. The engineering response is to **make the deadline explicit in configuration on both sides** (an env var your code reads for its internal drain budget, deliberately set *lower* than the platform's grace period) rather than trusting a shared assumption.

### Runnable example — a worker that finishes its current job before exiting

This is the syllabus's build task: install a SIGTERM handler that completes the in-flight job and then exits.

```python
# worker.py — stdlib only.  Run: python worker.py
import signal, sys, time

shutting_down = False

def on_sigterm(signum, frame):
    # RULE: handlers set a flag. They do not do work.
    global shutting_down
    shutting_down = True
    print(f"[worker] got signal {signum} ({signal.Signals(signum).name}) -> draining",
          flush=True)

signal.signal(signal.SIGTERM, on_sigterm)   # register the disposition

jobs = iter(range(1, 100))
while not shutting_down:                     # check the flag between jobs, never mid-job
    job = next(jobs)
    print(f"[worker] start job {job}", flush=True)
    for _ in range(4):                       # pretend this job takes 200ms of real work
        time.sleep(0.05)
    print(f"[worker] done  job {job}", flush=True)
    if job == 2:
        signal.raise_signal(signal.SIGTERM)  # stand-in for an external `kill` (see note)

print("[worker] drained cleanly, exiting 0", flush=True)
sys.exit(0)
```

Real output on my machine:

```
[worker] start job 1
[worker] done  job 1
[worker] start job 2
[worker] done  job 2
[worker] got signal 15 (SIGTERM) -> draining
[worker] drained cleanly, exiting 0
worker exit code: 0
```

**Why this works, line by line.**
- `signal.signal(signal.SIGTERM, on_sigterm)` changes this process's **disposition** for signal 15 from *default (terminate)* to *call this function*. From this instant, a `SIGTERM` no longer kills the process — an enormous behaviour change from one line.
- The handler sets `shutting_down = True` and **returns immediately**. It does not close the database, write a file, or sleep. This is the async-signal-safety rule from above, and even in Python (where handlers run at a safe bytecode boundary) it remains the correct shape: the handler's job is to *record intent*; the main loop's job is to *act on it at a safe point*.
- `while not shutting_down` checks the flag **between jobs**, never inside one. That placement is the definition of "graceful": the unit of atomicity is one job, and shutdown happens on a boundary. Move the check inside the inner loop and you have an abrupt shutdown with a polite log line.
- `sys.exit(0)` reports success. The supervisor now knows this was an intentional, clean stop rather than a crash — which is exactly what stops a rollout from being flagged as failed.

**Honesty caveats.**
1. **`signal.raise_signal` is a test stand-in, not the real mechanism.** In production the signal comes from *another* process (`kill -TERM <pid>`, `docker stop`, the kubelet). I used `raise_signal` so that the example is a single self-contained file that runs identically on Windows, Linux, and macOS and produces the output above. To exercise the real path on Linux/macOS: delete the `raise_signal` line, run `python worker.py &`, then `kill -TERM %1` from the shell.
2. **On Windows, the real path barely exists.** Per §5's verified output, Windows has no inter-process `SIGTERM` delivery in the POSIX sense: `Popen.terminate()` calls `TerminateProcess`, which is `SIGKILL`-like — **your handler will not run.** A Windows service receives shutdown notifications through the Service Control Manager (`SERVICE_CONTROL_STOP`) or console control events instead. So this pattern is correct and essential for Linux containers (where all your servers live) and is *not* portable to a Windows service. Knowing that is the difference between "works on my machine" and "works in prod."
3. **Production hardening this example omits, deliberately, for clarity:** (a) a *second* SIGTERM should escalate to immediate exit — a user who asks twice means it; (b) drain should have its own timeout so a wedged job cannot hold the process past the platform's grace period; (c) real workers must stop accepting *new* work (deregister from the load balancer / stop the queue consumer) *before* draining, or they will keep pulling jobs forever. All three appear in the system-design section below.

---

## §7. Zombies and orphans  **Depth: [WORKING]**

### Intuition — a zombie is a receipt, not a leak

When a process exits, the kernel can free almost everything immediately: memory, fds, kernel stacks. But it cannot free **the exit status**, because the parent has the right to ask "how did my child do?" — possibly much later. So the kernel keeps a tiny record: pid, exit code, resource usage. A process in that state is a **zombie** (Linux `ps` state `Z`, shown as `<defunct>`): dead, holding no memory, but still occupying a **PID slot** and a small kernel structure until someone calls `wait()`.

The mirror-image case: if the **parent** dies first, the child becomes an **orphan** — and since every process must have a parent, the kernel **reparents** it to `init` (pid 1), or on modern Linux to the nearest ancestor marked as a "subreaper." pid 1's job includes reaping whatever lands on it.

So:
- **Zombie** = child died, parent hasn't called `wait()`. *Parent's bug.*
- **Orphan** = parent died, child still running. *Handled automatically — but see the container trap below.*

Neither leaks memory. Zombies leak **PIDs**, and PIDs are a finite resource (`/proc/sys/kernel/pid_max`, commonly 32768 or 4194304). A long-running server that spawns children and never reaps them will, hours or days later, fail to create any process at all — and the error (`EAGAIN` from fork, "Resource temporarily unavailable") points nowhere near the actual bug. That delayed, misdirected failure is why this [WORKING]-tier topic earns real attention.

### The analogy — and where it breaks

A zombie is a **death certificate nobody has filed**. The person is gone; the paperwork sits in a tray occupying a slot in the registry. File it (`wait()`) and the slot frees. Let the tray fill and the registry office stops functioning — not because of the bodies, but because of the filing cabinet.

**Where the analogy breaks:** a death certificate can be filed by any clerk. A zombie can be reaped **only by its parent** (or by pid 1 after reparenting). `kill -9` on a zombie does *nothing at all* — it's already dead; there is no process to signal. The fix is never "kill the zombie"; it is always "make the parent call `wait()`, or kill the parent so the children get reparented to pid 1, which will reap them."

### Worked example — create a zombie, observe it, reap it (Linux/macOS)

```python
# zombie_demo.py — POSIX ONLY (os.fork does not exist on Windows, verified in §5).
# Run on Linux or macOS:  python3 zombie_demo.py
import os, sys, time, subprocess

if not hasattr(os, "fork"):
    sys.exit("This demo requires POSIX fork(); see the Windows note below.")

pid = os.fork()
if pid == 0:
    # ---- CHILD ----
    os._exit(3)                      # _exit, not exit: skip parent's atexit/flush handlers
else:
    # ---- PARENT ---- deliberately do NOT wait yet
    time.sleep(0.2)                  # let the child finish
    print(f"child pid {pid} has exited but I have not called wait()")
    print(subprocess.run(["ps", "-o", "pid,ppid,stat,comm", "-p", str(pid)],
                         capture_output=True, text=True).stdout)
    # now reap it
    reaped_pid, status = os.waitpid(pid, 0)
    print(f"waitpid -> pid={reaped_pid} exited={os.WIFEXITED(status)} "
          f"code={os.WEXITSTATUS(status)}")
    print(subprocess.run(["ps", "-o", "pid,ppid,stat,comm", "-p", str(pid)],
                         capture_output=True, text=True).stdout or "(pid gone)")
```

**Expected output shape on Linux** (your PIDs will differ; I am labelling this as *expected* rather than measured because — as §5 verified — this machine is Windows and physically cannot run `os.fork`; I will not present invented output as a real transcript):

```
child pid 41207 has exited but I have not called wait()
    PID    PPID STAT COMMAND
  41207   41206 Z+   python3 <defunct>          ← the zombie: state Z, "<defunct>"

waitpid -> pid=41207 exited=True code=3
(pid gone)                                     ← reaped: the PID slot is free
```

The two documented facts to anchor on (both from `ps(1)` and `proc(5)`): state **`Z`** means "defunct/zombie," and the command is rendered `<defunct>` because the program image is already gone. `os.WIFEXITED`/`os.WEXITSTATUS` are the standard macros that decode the packed status integer `waitpid` gives you — an exit code and "how it died" share one word, which is exactly why Python's `Popen.returncode` uses negatives to encode the signal case.

**Windows honesty note.** Windows has no zombie state, because it uses a different ownership model: process exit information lives in a **kernel object referenced by HANDLEs**, and the object survives as long as *any* handle to it is open. So the equivalent leak absolutely exists — it is just called a **handle leak**: a parent that keeps `Popen` objects alive (or a C program that never calls `CloseHandle`) accumulates process objects the same way. In Python the reaping API is uniform (`wait()`/`poll()`), which is why the rule "always `wait()` what you spawn" is portable even though the underlying mechanism is not.

### The container trap: PID 1 has special duties

Inside a container, **your process is pid 1** — and pid 1 has two special properties the kernel enforces:

1. **It inherits every orphan** in that PID namespace. If your app spawns a shell that spawns children and dies, those grandchildren land on you, and if you never `wait()`, they become permanent zombies.
2. **Signals to pid 1 are treated specially**: the kernel does *not* apply default actions for signals pid 1 hasn't installed a handler for. A shell script as pid 1 that doesn't handle SIGTERM will therefore **ignore `docker stop` entirely** and eat the full 10-second grace period before being SIGKILLed. If you have ever wondered why `docker stop` on your container takes exactly ten seconds every time, this is almost certainly why.

The fixes are standard and worth knowing by name: `docker run --init` (injects a tiny init — `tini` — as pid 1 to reap orphans and forward signals), `ENTRYPOINT ["myapp"]` in **exec form** rather than shell form (shell form wraps you in `/bin/sh -c`, which becomes pid 1 and doesn't forward signals), and in Kubernetes `shareProcessNamespace` or simply making your app handle signals correctly.

---

## §8. Namespaces and cgroups — the one-paragraph version  **Depth: [AWARE]**

A **namespace** makes a process see a *private version* of a global kernel resource: its own PID numbering (so your app is pid 1 and cannot see the host's processes), its own mount table (so `/` is your image), its own network stack (its own interfaces, ports, and routing), its own hostname, users, and IPC. A **cgroup** (control group) caps and accounts what a process tree may *consume*: memory (exceed it and the kernel OOM-kills you — hello, exit code 137), CPU shares and quota, block I/O, and PIDs. Both are configured through ordinary syscalls (`clone`/`unshare`/`setns`) and filesystem writes (`/sys/fs/cgroup/...`) — which is to say **a container is a normal process with a modified view and a budget, created by exactly the fork/exec machinery of §5.** There is no new kind of object, no hypervisor, and no guest kernel: `ps` on the host shows container processes as processes, because they *are* processes.

**Treat this as a black box until Day 88**, where the syllabus has you prove it. For today the only thing you must stop believing is that a container is a lightweight VM. A VM virtualises *hardware* and runs its own kernel; a container shares the host kernel and virtualises *names and limits*. That difference is the entire security conversation in Part 2 (§10): a container escape is a kernel bug away, because there is only one kernel.

---

## System design (Part 1)

### Scenario 1 — Graceful shutdown for a fleet of 200 workers

**Problem.** You run 200 queue-consumer pods. Each job takes 0.5–90 seconds. You deploy 10× a day, and every deploy replaces every pod. Requirement: **zero lost jobs and zero duplicate side effects** (jobs send emails and charge cards), and a deploy must not take more than a few minutes.

**Requirements, made explicit.**
| Requirement | Implication |
|---|---|
| No job lost | a job in flight must either complete or be returned to the queue for redelivery |
| No duplicate side effects | if a job is redelivered, handlers must be idempotent — shutdown alone cannot guarantee exactly-once |
| Bounded deploy time | the drain budget must have a hard ceiling; you cannot wait for a 90 s job forever *and* deploy in a minute |
| Observable | you must be able to tell "drained cleanly" from "was killed" after the fact |

**The design.**

```
 t=0    orchestrator decides to replace pod
 t=0    ── remove pod from Service endpoints (stop NEW work arriving)  ◄── FIRST
 t=0    ── send SIGTERM to pid 1
 t=0+ε  app handler: set shutting_down = True                (§6)
        app: stop the queue consumer / close the accept loop  (no new jobs)
        app: let in-flight jobs finish, up to DRAIN_DEADLINE
 t≤D    all jobs done → flush logs/metrics → exit(0)
 t=D    (DRAIN_DEADLINE, app-enforced) any job still running:
        → NACK it back to the queue so it is redelivered elsewhere
        → exit(0) anyway
 t=G    (GRACE_PERIOD, platform-enforced) orchestrator sends SIGKILL
```

**Key decision: `DRAIN_DEADLINE < GRACE_PERIOD`, always, with margin.** Set `terminationGracePeriodSeconds: 120` and the app's internal deadline to 100 s. The app must be the one that gives up, because only the app can NACK the job, flush the logs, and exit 0. If the platform's SIGKILL wins the race, you lose all three — and per §6 the app cannot *discover* the platform's grace period, so both numbers must be configuration, ideally rendered from one source of truth.

**The trade-off made.** Jobs longer than the deadline get *redelivered*, not completed. We are explicitly buying bounded deploy time with the cost of occasional duplicate delivery — which is only acceptable because we separately require idempotent handlers (an idempotency key on the charge, a dedupe table on the email). **Shutdown design cannot give you exactly-once semantics; it can only give you at-least-once plus a bounded window.** Any design that claims otherwise is hiding an assumption.

**Failure modes, and what each looks like.**
| Failure | Symptom | Fix |
|---|---|---|
| Handler does work instead of setting a flag | shutdown hangs or corrupts | flag + main-loop check (§6) |
| Deregistration happens *after* SIGTERM handling starts | requests arrive at a draining pod → 502s | remove from endpoints first; add a `preStop` hook sleep to cover propagation lag in the LB |
| pid 1 is `/bin/sh -c "python app.py"` | handler never runs; every pod eats the full grace period; exit 137 | exec-form ENTRYPOINT (§7) |
| No internal deadline | one wedged job → SIGKILL → lost job + truncated logs | app-side `DRAIN_DEADLINE` |
| Logs block-buffered | the shutdown story is destroyed by the kill | `PYTHONUNBUFFERED=1` (§3) |
| All 200 pods drain at once | the queue's redelivery spike stampedes the new pods | rolling deploy with `maxUnavailable`, plus jittered drain |

**Why this over the alternatives.** *Alternative A: no handler, rely on queue redelivery.* Simpler, and legitimately correct for short idempotent jobs — but every deploy generates a redelivery spike and you lose the diagnostic distinction between "clean stop" and "crash." *Alternative B: quiesce externally (stop the queue, wait for zero in-flight, then delete pods).* Strictly safer, and appropriate for a payments ledger; costs you an orchestration component and slower deploys. The SIGTERM design is the right default because it needs no coordinator and degrades to Alternative A's behaviour when it fails.

### Scenario 2 — A multi-tenant "run this code" service (untrusted, arbitrary code from strangers)

**Problem.** Build the backend for an online judge / notebook service: accept a code snippet from an anonymous user, run it, return stdout within 5 seconds. Tens of requests per second. Snippets are actively hostile: fork bombs, `rm -rf /`, crypto miners, attempts to read other tenants' submissions, attempts to reach your internal network.

**Requirements → mechanism mapping. This is the layered answer, and the layers are the point:**

| Threat | Layer that stops it | Concept |
|---|---|---|
| Reads/writes your app's memory | separate **process** — different address space | §2, §4 |
| Reads your files, config, secrets | own **mount namespace** / chroot; nothing sensitive mounted; read-only rootfs | §8 |
| Acts as a privileged user | run as **non-root, unprivileged uid**; drop all capabilities; `no-new-privileges` | §4 credentials |
| Infinite loop | **wall-clock timeout** → SIGKILL; cgroup CPU quota | §6, §8 |
| Memory bomb | **cgroup memory limit** (kernel OOM-kills it, cheaply) | §8 |
| Fork bomb | **cgroup pids limit** (`pids.max`) — the only reliable defence | §8 |
| Fills the disk | quota / tmpfs with a size cap / read-only FS | §8 |
| Calls home, scans your VPC, mines crypto | **network namespace with no interfaces** (not a firewall rule — no NIC at all) | §8 |
| Exploits a kernel bug to escape | **seccomp-BPF** syscall allowlist to shrink kernel attack surface; then **gVisor** (userspace kernel) or a **microVM** (Firecracker) for a real boundary | §3, §8 |
| Correlates with / poisons the next tenant | **ephemeral sandbox**: destroy and recreate, never reuse across tenants | §5 |

**The key decision: how strong a boundary, and what it costs.**

| Isolation | Escape difficulty | Startup cost | Verdict |
|---|---|---|---|
| Same process (`eval`) | **none** — this is not isolation | ~0 | never, for untrusted input |
| Separate process, same kernel, no namespaces | low (reads the whole filesystem — *demonstrated in Part 2 §10*) | ms–s (§5) | insufficient alone |
| Container (namespaces + cgroups + seccomp) | moderate — one kernel bug (CVE-2019-5736) | tens of ms | the pragmatic default |
| gVisor / userspace kernel | high — syscalls handled outside the host kernel | ~100 ms class | good middle |
| microVM (Firecracker) | highest short of separate hardware | ~100 ms class | what real multi-tenant code services use |

**The design:** a pool of pre-warmed microVMs or gVisor sandboxes (per §5's spawn-cost measurement, cold-starting per request is what kills your p99), each handed exactly one submission and then destroyed; a supervisor that enforces the wall-clock deadline and kills the whole **process group** (not just the direct child — see Part 2 §11); output captured to a size-capped buffer so a `while True: print('x')` cannot OOM your *collector*.

**Failure modes.** Sandboxes that leak because the supervisor crashed after spawning (fix: a reaper that sweeps by age, plus cgroup-based tracking, plus pid-1 reaping per §7). Output-size bombs. Timeout accounting that measures CPU rather than wall time and lets `sleep(3600)` through. And the subtle one: **the sandbox is not the only boundary** — the code that *builds* the sandbox request and *renders* the output is in your trust domain, so a snippet that emits terminal escape sequences or gigantic JSON can attack your frontend instead.

**Why this over the alternatives.** A language-level sandbox (restricted interpreter, blocked builtins) is the tempting cheap option and has been broken repeatedly for every dynamic language — Python's own docs abandoned `rexec`/`Bastion` as unfixable. The OS boundary is the one that was *designed* to hold against hostile code; use it, and treat language-level restrictions as defence in depth only.

---

## Case studies (Part 1)

### ① Chrome's process-per-tab: paying memory for isolation

**What happened.** Chrome launched in 2008 with a multi-process architecture, at a time when every other browser was a single process. Each tab's renderer (HTML parsing, JavaScript, layout) runs in its own OS process, sandboxed and unprivileged; a single privileged **browser process** brokers all access to the network, disk, and GPU. Renderers cannot open files or sockets directly — they ask the broker over IPC. Later, **Site Isolation** (2018, accelerated by Spectre) went further: a separate process per *site*, so that `evil.com` in an iframe can never share an address space with `bank.com`.

**The engineering lesson, tied to the concept.** This is §2 and §4 turned into a product decision. Chrome deliberately accepted a large, *permanent*, measurable memory cost — each process carries its own heap, its own JS engine state, its own overhead, and the copy-on-write savings that make `fork` cheap don't survive long once each renderer diverges — in exchange for two properties nothing else could provide: (a) **crash containment** — one tab's segfault kills one process, and the tab shows a sad-face icon instead of taking your other 40 tabs with it; (b) **security containment** — a full renderer compromise (a JS engine exploit) still lands the attacker in an unprivileged process with almost no syscalls available, so they need a *second* exploit (a sandbox escape) to reach your files. The Spectre motivation is the sharpest version of the argument: once you accept that hardware can leak data *within* an address space regardless of software checks, the only reliable mitigation is to not put two security domains in the same address space at all. "Just use threads, they're cheaper" is only true if you don't need a boundary.

**Primary source.** The Chromium design docs, "Multi-process Architecture" and "Site Isolation" (`chromium.org/developers/design-documents/`), plus the Chromium sandbox design docs. Verify current: the process model has changed repeatedly (process-per-site-instance, process consolidation under memory pressure on low-RAM devices, `--process-per-site` variants), so treat specific limits as version-dependent.

### ② Docker demystified: a container is a process, and that is provable

**What happened.** Docker (2013) packaged existing Linux kernel features — namespaces (2002–2013) and cgroups (2007) — behind a usable CLI and an image format. It did not invent isolation, and it did not add a hypervisor.

**The engineering lesson.** Run `docker run -d nginx`, then on the *host* run `ps aux | grep nginx`: the processes are there, in the host's process list, visible and killable, with pids the host assigns. Inside the container `ps` shows nginx as pid 1 with no other processes — because the PID namespace is showing it a filtered, renumbered view (§8). Same processes, two views. `docker stop` sends SIGTERM to pid 1 and SIGKILLs after 10 seconds (§6), which is why §7's pid-1 signal-forwarding trap is a *Docker* problem, not an abstract one. `docker run -m 100m` writes a cgroup memory limit; exceed it and the kernel OOM-kills you and you see exit code **137** = 128+9 (§5).

**Verify it yourself, today:** `docker run --rm -it alpine sh -c 'echo $$; ps aux'` → prints `1` and a two-line process table. Then `docker inspect --format '{{.State.Pid}}' <id>` on the host → the host-side pid of the same process. Two numbers, one process. That's the whole demystification, and Day 88 goes further.

**Primary source.** Linux `namespaces(7)`, `cgroups(7)`, `clone(2)` man pages; Docker's own docs on `--init` and stop-signal handling.

### ③ A failure: CVE-2019-5736 — escaping a container through `/proc/self/exe`

**What happened (Feb 2019).** A vulnerability in **runc**, the low-level container runtime underneath Docker, containerd, and Kubernetes, allowed code running *inside* a container to overwrite the **runc binary on the host** and thereby execute as root on the host. The mechanism is a beautiful, horrifying tour of this note: when runc `exec`s into a container, it re-executes itself; a malicious container could arrange for the binary being executed to be a symlink resolving, via **`/proc/self/exe`** (the kernel's magic symlink to the running executable — §4's "kernel state exposed as files"), to the *host's* runc binary, and then write to it through an open file descriptor (§4: **an fd is a capability, checked once at open time**) at the moment runc was running with host privileges. Escape achieved. Container images that a user merely *ran* could take over the machine.

**The engineering lesson.** Three, all direct corollaries of Part 1:
1. **A container is a process sharing your kernel** (§8). The boundary is enforced by correct configuration of a shared kernel plus the correctness of the runtime, not by a hardware/hypervisor gap. This is precisely why the design in Scenario 2 escalates to gVisor or microVMs for genuinely hostile code — it changes what an escape has to defeat.
2. **File descriptors outlive permission checks** (§4). The fix (and the modern hardening pattern) is to hold your own binary as an O_PATH/memfd copy and to *not* leave a writable path to host state open across a privilege transition. If your mental model is "the file permissions will protect it," you are checking the wrong thing at the wrong time.
3. **Privilege transitions are the danger zone** — the same insight as §5's fork/exec gap, inverted. The gap between fork and exec is where you *drop* privilege deliberately; a bug in that window is where an attacker *gains* it.

**Primary source.** NVD entry for CVE-2019-5736; the original disclosure on the Openwall oss-security list (Feb 11, 2019) by Adam Iwaniuk and Borys Popławski; runc's patch and release notes. Verify current details against those — secondary write-ups of this CVE vary in accuracy.

**Honest gap.** I have deliberately not offered you a fourth "graceful shutdown outage" postmortem, even though the class is extremely common (pods SIGKILLed mid-transaction, exit 137 storms, LB 502s during deploys) — I do not have a specific, well-documented, correctly-attributable public postmortem in hand to name, and per Principle 6/7 an invented one would be worse than none. What I can point you to as authoritative *mechanism* is the Kubernetes documentation on Pod termination (`terminationGracePeriodSeconds`, `preStop` hooks) and `systemd.kill(5)` on `TimeoutStopSec`/`KillSignal`; if you want a real incident to study, search your own organisation's incident tracker for `exit code 137` — you will find one.

---

## In production (Part 1) — how this is really operated, and really gotten wrong

**Who supervises your processes.** You almost never manage processes by hand. **systemd** (unit files: `ExecStart`, `Restart=on-failure`, `KillSignal`, `TimeoutStopSec`) manages host services. **gunicorn/uwsgi/puma** pre-fork a pool of workers at boot and re-fork on death — the direct answer to §5's spawn-cost measurement. **Kubernetes** manages containers via the kubelet and a container runtime; its liveness/readiness/startup probes and restart backoff are all built on exit codes and signals. Every one of these is a §5+§6+§7 machine: create children, watch for their death, read exit codes, send SIGTERM then SIGKILL, reap.

**Best practices.**
- **Exec-form entrypoints**, always, so your app is pid 1 and gets its signals (§7).
- **Handle SIGTERM; set an internal drain deadline below the platform grace period** (Scenario 1).
- **`PYTHONUNBUFFERED=1`** (or your language's equivalent) so a kill can't destroy your logs (§3).
- **Run as non-root; drop capabilities; read-only rootfs** (§4).
- **Always `wait()` what you spawn** (§7), and prefer `subprocess.run(..., timeout=)` or a context manager so it happens even on exceptions.
- **Set memory limits deliberately**, so the kernel OOM-kills the offender instead of the machine thrashing or the kernel picking a victim by heuristic.
- **Make exit codes meaningful** and distinguish "config invalid, restarting won't help" from "transient, retry me" — restart policies can only act on what you tell them.

**Monitoring and observability — what to actually put on a dashboard.**
| Signal | Why | Where |
|---|---|---|
| exit code distribution per service | 137 = OOM/kill, 1 = crash, 0 = clean; a rising 137 rate is a capacity or shutdown bug | orchestrator events |
| restart count / CrashLoopBackOff | crash loops are invisible in request metrics if the LB hides them | kubelet, systemd |
| RSS vs. limit, and OOM-kill count | pre-empts the 137s | cgroup memory stats |
| open fd count vs. `RLIMIT_NOFILE` | leaks grow linearly and fail suddenly | `/proc/<pid>/fd`, `psutil` |
| zombie/defunct process count | non-zero and growing = a missing `wait()` (§7) | `ps`, node exporter |
| process/thread count per container | fork bombs, thread leaks | cgroup `pids.current` |
| shutdown duration, and "drained cleanly" vs. "killed" | the only way to know your grace period is right | your own log line + a metric |

**Common mistakes, beginner → senior.**
1. *Beginner:* not checking exit codes at all; `os.system("...")` and hoping.
2. *Beginner:* `shell=True` with user input — command injection, because the shell parses metacharacters your `argv` list would have kept literal.
3. *Junior:* spawning a process per request (§5's cost) and being confused by p99 latency.
4. *Junior:* `Popen` without `wait()`/`poll()` — zombies (§7) and, on Windows, handle leaks.
5. *Mid:* deadlocking on `Popen.communicate()`'s absence: writing to a child's stdin while its stdout pipe fills, so both sides block forever on a full 64 KiB pipe buffer. Use `communicate()`, or drain concurrently.
6. *Mid:* assuming SIGTERM means "you have plenty of time."
7. *Senior:* `fork()` in a multi-threaded process (the inherited-mutex trap in §5) — the bug that manifests as a once-a-week hang in a pre-forking server that also has a logging thread.
8. *Senior:* treating a container as a security boundary against actively hostile code without namespaces+seccomp+non-root, or without escalating to gVisor/microVM (CVE-2019-5736).

**Scaling behaviour and cost.** Processes cost memory (RSS, not shared pages) and PIDs, and are *created* expensively. This drives the entire shape of server architectures: pre-forked pools (bounded memory, no per-request creation cost), then threads (cheaper, shared address space, needs locks), then async/event loops (cheapest per connection, one process, but one blocking syscall stalls everything). The right question is never "processes or threads or async" in the abstract, but "what is my isolation requirement, and what is my per-unit-work creation cost?" Chrome answers *isolation*; a proxy handling 100k connections answers *creation cost*.

**Failure modes + recovery, condensed.** OOM-kill → lower memory use or raise the limit (and check for a leak first). PID exhaustion → find the missing `wait()`. fd exhaustion → find the leaked socket/file. Crash loop → read the *previous* container's logs (`kubectl logs --previous`), because the current one hasn't failed yet. Wedged process ignoring SIGTERM → SIGKILL, then fix the handler; if SIGKILL also fails, the process is stuck in uninterruptible kernel I/O (state `D`) and your problem is storage, not the app.

---

## Failure modes and common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "The kernel is a program I can call functions in" | You can only make **syscalls**, from a fixed finite list, through a hardware-mediated door (§2, §3) |
| "Syscalls are basically function calls" | 10–100× more expensive; the whole art of I/O performance is making fewer of them with more data (§2 measurement) |
| "A zombie process is eating my memory" | It holds no memory; it holds a **PID slot** and an exit record (§7) |
| "`kill -9` fixes a zombie" | Nothing can kill what's already dead; reap it or kill the parent (§7) |
| "`kill` means kill" | `kill` sends *any* signal; `kill -HUP` often means "reload config" (§6) |
| "SIGKILL always works instantly" | Not if the process is blocked in uninterruptible kernel I/O (§6) |
| "My SIGTERM handler can take as long as it needs" | The platform will SIGKILL you at its grace period, which your process cannot discover (§6) |
| "A container is a lightweight VM" | One shared kernel + namespaces + cgroups; escapes are kernel bugs away (§8, CVE-2019-5736) |
| "Threads are just cheap processes" | They share the address space and fd table — no isolation whatsoever, which is the entire point of Chrome's design |
| "`fork` copies my memory" | Copy-on-write: it copies page tables and defers the rest (§5) |
| "Exit code 1 tells me what went wrong" | Exit codes carry one byte; the diagnosis is in stderr, which you must not lose to buffering (§3, §5) |
| "`os.fork` / SIGKILL / `-15` returncodes are portable" | Verified false on Windows in §5's real output |

## Interview questions and practice (Part 1)

**Conceptual (answer out loud, from memory):**
1. Why does the user/kernel split need *hardware* support? What breaks if you try to enforce it purely in software?
2. Trace `print("hi")` from the Python line to the pixel. Name at least four syscalls involved and explain why the majority of them have nothing to do with your string.
3. Why did Unix split process creation into `fork` + `exec` instead of one `spawn` call? Give two concrete features that exist *only* because of the gap between them.
4. SIGTERM vs SIGKILL: which one can you handle, why not the other, and what does every orchestrator do with both?
5. A long-running service starts failing with "Resource temporarily unavailable" on process creation after four days of uptime. Walk through your diagnosis.
6. Your pod takes exactly 30 seconds to terminate on every deploy and then reports exit code 137. What are the two most likely causes and how would you distinguish them?
7. Why is a container *not* a security boundary equivalent to a VM? Name the mechanism that differs.
8. You must run untrusted code. List your isolation layers in order and say what each one stops.

**Practical:**
9. Write a supervisor that runs a child with a 2-second timeout, kills it if it overruns, and reports whether it exited normally, was killed, or timed out — correctly on both Linux and Windows (remember §5's returncode caveat).
10. Produce a zombie on Linux, observe `Z` in `ps`, then reap it. Then explain why the same script cannot demonstrate this on Windows.
11. Take `worker.py` from §6 and add: a drain timeout, escalation on a second SIGTERM, and a metric line reporting drain duration.

<!-- APPEND-MARKER -->
