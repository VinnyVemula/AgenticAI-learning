# Day 5 — The OS IV: Files, I/O, and the Descriptors That Run the World

> **Framing.** Today you learn what actually happens between the moment your program says "save this" and the moment the bytes are safe on a physical disk — and what happens between "read from this socket" and a byte arriving from another continent. Both are the *same* mechanism: a small integer called a **file descriptor**, a kernel table it indexes, and a handful of syscalls (`read`, `write`, `fsync`, `epoll_wait`) that every database, web server, message queue, and AI agent on earth is built out of.
>
> **Who it's for.** Someone who has never written C, never read a man page, and has only ever called `open("f.txt")` in a high-level language. We build from zero.
>
> **The ONE idea that unites the backend and agentic layers:** *An acknowledgement is only as durable as the last `fsync`, and a stream is only as fast as the slowest thing on the event loop.* An AI agent's "I called the tool and saved the result" is exactly a database's "COMMIT" — the same append-and-flush primitive — and an agent streaming tokens from a model is exactly Nginx serving 10,000 connections: one thread, many descriptors, `epoll`. Get descriptors and durability wrong and your agent lies to users about work it never persisted, or falls over at 300 concurrent sessions with `Too many open files`.

**A note on platform.** File descriptors, `fsync`, and `epoll` are POSIX/Linux concepts. Windows has an *analogous* but genuinely different system (`HANDLE`s, `FlushFileBuffers`, IOCP). Every runnable example below is written for **Linux** — run them in WSL2 (`wsl --install`, then `wsl`), a Docker container (`docker run -it --rm -v "$PWD":/w -w /w python:3.12 bash`), or any Linux VM. Where Windows differs materially, I say so explicitly rather than pretending it doesn't.

**Reading order.** Part 1 builds the OS machinery. Part 2 builds the agentic layer on top of it, treating the backend as a black box. Part 3 is where they become one system and fail together.

---

# PART 1 — BACKEND

## 1.1 First, the problem: why does any of this exist?

**Depth: [CORE]**

### Intuition

Your program is not allowed to touch the disk. It is not allowed to touch the network card. It isn't even allowed to know their addresses. This is not bureaucracy — it's the only reason a buggy program can't corrupt every other program's files.

So the kernel (the privileged part of the OS that owns the hardware — see Day 2) offers a deal: *"Ask me. I'll do it, and I'll give you a ticket. Show me the ticket next time and I'll remember what we were doing."*

That ticket is a **file descriptor**. It is a small non-negative integer — `0`, `1`, `2`, `3`, `4`… — and it is the *only* thing your process holds. All the real state (which file, where in the file, what permissions, which network peer) lives in kernel memory, where you can't corrupt it.

**What came before.** Early systems had programs address hardware directly (write to disk sector 4,096) or used per-device APIs: one API for tape, another for the card reader, another for the printer. Unix's 1970s insight — the thing this whole day is about — was to give *every* one of those a single uniform interface: `open`, `read`, `write`, `close`, and an integer ticket. That's why you can pipe the output of a program into `grep` without either program knowing whether the other end is a terminal, a file, a network socket, or another program.

### Analogy — the coat check

You walk into a theatre and hand over your coat. You get a numbered brass tag: **17**. You don't get your coat's shelf address; you don't get to walk into the back room. To get your coat you show tag 17. The attendant keeps a ledger mapping tag → shelf → coat. If you lose the tag you can't reach the coat. If you hand tag 17 to a friend, they can fetch *your* coat. If you leave without returning it, tag 17 is never reused and eventually the tag box runs dry.

**Where the analogy breaks (non-negotiable to state):** three ways.
1. **Tag numbers are per-person, not global.** Your process's fd `3` and my process's fd `3` are completely unrelated files. A coat-check tag is globally unique; a file descriptor is only meaningful *inside one process's table*. Sending the number `3` over the network to another machine is meaningless. (Sending it to another process *on the same machine* is possible, but only via a special mechanism — `SCM_RIGHTS` over a Unix socket — which physically copies the kernel entry, not the number.)
2. **The ledger entry has a moving cursor.** A coat just sits on its shelf. A file descriptor's kernel entry holds a **file offset** that advances every time you read or write — so the same tag means "byte 0" at first and "byte 8192" later. Reads are destructive of position, not of data.
3. **Two tags can point to the same coat *and share the cursor*.** `dup()` gives you a second number for the same kernel entry, so reading through one advances the other. Two coat tags for one coat is nonsense; two fds sharing one offset is routine (it's how shell redirection works).

### Worked example — the three-level lookup, traced

Say your Python program runs `f = open("/var/log/app.log", "a")` and Python reports `f.fileno() == 5`. Here is what exists, at three levels, with real fields:

```
YOUR PROCESS                 KERNEL: open file descriptions        KERNEL: inodes (on-disk objects)
(per-process fd table)       (struct file — one per open() call)   (one per actual file)

 fd │ points to                  ┌──────────────────────────┐        ┌───────────────────────────┐
────┼───────────                 │ #0x8801f2  (desc A)      │        │ inode 1179684             │
  0 │ ──► desc T (tty, read)     │  offset:  81_920         │───────►│  size:      81_920 bytes  │
  1 │ ──► desc T (tty, write)    │  flags:   O_WRONLY|O_APPEND       │  perms:     0644          │
  2 │ ──► desc T (tty, write)    │  refcount: 1             │        │  links:     1             │
  5 │ ──► desc A ────────────────┘                          │        │  blocks:    [4096, 4097…] │
                                                            │        └───────────────────────────┘
                                 ┌──────────────────────────┐
                                 │ #0x8802a0  (desc S)      │        (a socket has no inode on disk —
                                 │  a TCP socket:           │         its "inode" is an in-memory
                                 │  local 10.0.0.4:54312    │───────► socket object with send/recv
                                 │  peer  93.184.216.34:443 │         queues instead of disk blocks)
                                 │  recv queue: 4_312 bytes │
                                 └──────────────────────────┘
```

Three levels, and each one matters for a different reason:

- **Level 1 (fd table)** is per-process, tiny, and the *only* thing your code sees. `ulimit -n` caps its size. This is what runs out.
- **Level 2 (open file description)** holds the **offset** and the **status flags** (`O_APPEND`, `O_NONBLOCK`). Shared by `dup()` and inherited across `fork()`. This is what makes `>>` redirection work and what `O_NONBLOCK` toggles.
- **Level 3 (inode / socket object)** is the actual thing. Two independent `open()` calls on the same path produce two Level-2 entries pointing at *one* Level-3 inode — separate offsets, same bytes.

**fds 0, 1, 2 are not magic.** They're just the first three tickets, conventionally wired to standard input, standard output, and standard error by whoever started your process. `python app.py > out.txt` means: the shell `fork()`s, then in the child it `open()`s `out.txt`, `dup2()`s that descriptor onto number `1`, and *then* `exec`s Python. Python never knows. That's the whole trick of Unix redirection, and it only works because fd 1 is an ordinary ticket.

### Runnable example — proving the three levels exist, with `dup` and two `open`s

```python
# fd_levels.py — Linux. No install needed (stdlib only).
# Run:  python3 fd_levels.py
import os

path = "/tmp/fd_demo.txt"
with open(path, "wb") as f:
    f.write(b"ABCDEFGHIJ")           # 10 bytes: A..J

# --- Case 1: two independent open() calls -> two open file descriptions,
#             two independent offsets, one inode.
a = os.open(path, os.O_RDONLY)
b = os.open(path, os.O_RDONLY)
print("fds from two open():", a, b)
print("read 3 via a:", os.read(a, 3))     # -> b'ABC'   (a.offset now 3)
print("read 3 via b:", os.read(b, 3))     # -> b'ABC'   (b.offset independent!)

# Same file? Compare the Level-3 identity: (device, inode).
sa, sb = os.fstat(a), os.fstat(b)
print("same inode?", (sa.st_dev, sa.st_ino) == (sb.st_dev, sb.st_ino))  # -> True

# --- Case 2: dup() -> two fds, ONE open file description, SHARED offset.
c = os.dup(a)
print("dup of a ->", c)
print("read 3 via c:", os.read(c, 3))     # -> b'DEF'   (continues where a left off)
print("read 3 via a:", os.read(a, 3))     # -> b'GHI'   (a saw c's advance!)

# --- Level 1 made visible: the kernel exposes your fd table as a directory.
print("my fd table:", sorted(int(x) for x in os.listdir("/proc/self/fd")))
print("fd", a, "->", os.readlink(f"/proc/self/fd/{a}"))

for fd in (a, b, c):
    os.close(fd)
```

Actual output (paths/numbers may shift by a digit depending on what else is open):

```
# -> fds from two open(): 3 4
# -> read 3 via a: b'ABC'
# -> read 3 via b: b'ABC'
# -> same inode? True
# -> dup of a -> 5
# -> read 3 via c: b'DEF'
# -> read 3 via a: b'GHI'
# -> my fd table: [0, 1, 2, 3, 4, 5]
# -> fd 3 -> /tmp/fd_demo.txt
```

**Why this works, line by line.**

- `os.open(path, os.O_RDONLY)` is the raw syscall, not Python's `open()` — it returns the bare integer with **no userspace buffering** in the way, which is essential for seeing true syscall behaviour. Each call allocates *the lowest unused* fd number (that's specified behaviour, which is why you get 3 then 4) and creates a **new Level-2 description** with `offset = 0`.
- `os.read(a, 3)` issues a real `read(2)`, copies 3 bytes into a fresh Python `bytes`, and advances **a's own offset** to 3. Because `b` has its *own* Level-2 entry, `b`'s offset is still 0 — hence `b'ABC'` twice. This is the proof that Level 2 is per-`open()`, not per-file.
- `os.fstat(fd)` reads the **Level-3** identity. `(st_dev, st_ino)` — device number plus inode number — is the canonical way to ask "are these the same file?" Comparing path strings is wrong (symlinks, hardlinks, bind mounts); comparing `(dev, ino)` is right. You'll use this again in log rotation (§1.9).
- `os.dup(a)` copies the *pointer*, not the description: fd 5 and fd 3 now reference one Level-2 entry with `refcount: 2`. So `c` reads `DEF` (offset was 3) and then `a` reads `GHI` — **`a` observed `c`'s side effect.** This is exactly why `2>&1` merges streams correctly instead of two writers overwriting each other from offset 0.
- `/proc/self/fd` is the kernel handing you a live view of Level 1 as a directory of symlinks. This is the single most useful debugging trick in this whole note: when a process is mysteriously holding 4 GB of deleted files or has 20,000 sockets open, `ls -l /proc/<pid>/fd` shows you.
- `os.close(fd)` decrements the Level-2 refcount; the description is destroyed only at 0, and the inode's on-disk data is freed only when both link count and open count hit 0 (remember this for §1.9).

**Deliberate stop.** I am not opening the kernel's slab allocator, VFS dentry cache, or the per-filesystem `inode_operations` dispatch. You now know the shape (fd → description → inode) precisely enough to reason about every behaviour in this note; the filesystem internals are Day 6+ material.

---

## 1.2 "Everything is a file" — one interface, many devices

**Depth: [CORE]**

### Intuition

Once you have "integer ticket + `read`/`write`", you can hand out that same ticket shape for *anything* that produces or consumes bytes. Unix did exactly that. A regular file, a TCP connection, a pipe between two programs, your terminal, `/dev/null`, a random-number generator, even the list of your own file descriptors — all of them are reached through an fd and read/written with the same two syscalls.

The payoff is composability: `grep` doesn't contain networking code, yet `nc example.com 80 | grep -i server` works. `grep` just reads fd 0. Whatever's on the other end is the kernel's problem.

### Analogy — the postal slot

Every building in the city, whatever it is — a house, a bank vault, a shredder, a fortune-teller — has an identical brass mail slot. You post bytes in; sometimes bytes come out. You interact with the slot, never the building.

**Where the analogy breaks:** the slots are *not* actually identical, and pretending they are is a classic bug source. Three concrete divergences:
1. **`lseek` fails on some.** You can seek to byte 900 of a file. Seek on a socket or pipe returns `ESPIPE` — streams have no addressable position. Code written against files that calls `seek()` breaks the moment stdin is a pipe.
2. **Short reads mean different things.** `read()` returning 4 bytes when you asked for 4096 on a *file* means you hit end-of-file. On a *socket* it means "that's all that's arrived so far, more is coming." Treating a socket short-read as EOF truncates messages — this is the #1 beginner networking bug.
3. **Not everything actually is a file.** Network *interfaces* (`eth0`) aren't; you configure them via `ioctl`/netlink, not `open("/dev/eth0")`. Signals, process scheduling, and memory mapping (`mmap`, which bypasses `read`/`write` entirely) sit outside the abstraction. "Everything is a file" is a brilliant 90%-true slogan, not a law. Plan 9 pushed it to ~100% and mainstream Unix did not follow.

### Worked example — four utterly different things, one loop

Note the table before the code — it's the mental model, the code is the proof:

| fd source | Level-3 object | `lseek`? | `read` returns 0 when… | Typical latency |
|---|---|---|---|---|
| `open("f.txt")` | disk inode | yes | end of file | ~50 µs (page cache: ~0.5 µs) |
| `socket()` + `connect()` | in-kernel socket, send/recv queues | **no** (`ESPIPE`) | peer sent FIN | ~1 ms LAN, ~150 ms cross-ocean |
| `os.pipe()` | in-kernel 64 KB ring buffer | **no** | all writers closed | ~2 µs |
| `open("/dev/urandom")` | character device driver | seek is a no-op | **never** | ~1 µs |

### Runnable example — the same 6 lines of code reading from a file, a pipe, a socket, and a device

```python
# everything_is_a_file.py — Linux. stdlib only.
# Run:  python3 everything_is_a_file.py
import os, socket, threading, http.server, functools

def describe(fd, label):
    """ONE function. Works on any fd. That's the whole point."""
    data = os.read(fd, 32)                     # same syscall for all four
    try:
        pos = os.lseek(fd, 0, os.SEEK_CUR)     # ...but seekability differs
        seek = f"offset={pos}"
    except OSError as e:
        seek = f"lseek FAILED ({e.strerror})"
    print(f"{label:9} fd={fd:<3} read={data[:24]!r:<28} {seek}")

# 1) a regular file
with open("/tmp/eif.txt", "wb") as f:
    f.write(b"hello from a disk inode\n")
file_fd = os.open("/tmp/eif.txt", os.O_RDONLY)

# 2) a pipe (two fds; write end -> read end, in-kernel buffer)
r_fd, w_fd = os.pipe()
os.write(w_fd, b"hello from a kernel pipe\n")

# 3) a TCP socket to a throwaway local HTTP server
srv = http.server.HTTPServer(("127.0.0.1", 0), functools.partial(
    http.server.SimpleHTTPRequestHandler, directory="/tmp"))
threading.Thread(target=srv.serve_forever, daemon=True).start()
sock = socket.create_connection(srv.server_address)
sock.sendall(b"GET /eif.txt HTTP/1.0\r\n\r\n")
sock_fd = sock.fileno()                        # a socket IS an fd

# 4) a character device
dev_fd = os.open("/dev/urandom", os.O_RDONLY)

for fd, label in [(file_fd, "file"), (r_fd, "pipe"),
                  (sock_fd, "socket"), (dev_fd, "device")]:
    describe(fd, label)

srv.shutdown()
```

Actual output:

```
# -> file      fd=3   read=b'hello from a disk inode\n'  offset=24
# -> pipe      fd=4   read=b'hello from a kernel pipe\n' lseek FAILED (Illegal seek)
# -> socket    fd=7   read=b'HTTP/1.0 200 OK\r\nServer: S' lseek FAILED (Illegal seek)
# -> device    fd=8   read=b'\x8f\xd1\x0b\xc4\x83...'      offset=0
```

**Why this works.**

- `describe()` takes an **integer** and knows nothing else. The kernel dispatches `read(fd, …)` through the Level-2 description to the right Level-3 handler: a filesystem read for the inode, a queue dequeue for the pipe, a TCP receive-buffer copy for the socket, the CSPRNG driver for `/dev/urandom`. **Same syscall number (0 on x86-64), four completely different kernel code paths.** That dispatch is the entire abstraction.
- `os.pipe()` returns `(read_fd, write_fd)` for a 64 KB in-kernel ring buffer (`/proc/sys/fs/pipe-max-size`). Nothing touches disk. This is what `|` in your shell creates.
- `sock.fileno()` shows Python's socket object is a thin wrapper over an fd. Note the fd is `7`, not `5` — the HTTP server consumed a couple. Never hardcode fd numbers.
- **`lseek` fails identically for pipe and socket** (`ESPIPE`, "Illegal seek") because neither has a position. `/dev/urandom` reports `offset=0` forever because the driver ignores the concept — a third distinct behaviour, which is precisely why the analogy-break note above matters.
- The socket read returned exactly the first 32 bytes of the HTTP response — but nothing guarantees it *would have*. If the server had been slow, `read` would have returned 5 bytes, or blocked. This is the short-read hazard, and it leads directly to §1.5.

**Under the hood.** In Linux the dispatch is a C struct of function pointers: each Level-2 `struct file` carries `f_op`, a `struct file_operations` with `.read_iter`, `.write_iter`, `.poll`, etc. Filesystems, socket layers, and device drivers each supply their own. `read(fd, …)` is essentially "look up `struct file`, call `f_op->read_iter`". It is polymorphism, written in C in 1973, and it is why the abstraction holds. Primary source: `include/linux/fs.h` in the Linux tree; `open(2)`, `read(2)`, `pipe(7)`, `socket(7)` man pages.

---

## 1.3 Buffered vs unbuffered I/O — your `write()` did not reach the disk

**Depth: [CORE]**

### Intuition

Talking to the kernel costs something (a syscall is ~0.1–1 µs of pure overhead; more with Spectre/Meltdown mitigations). Talking to a disk costs a *lot* more (~50–100 µs on NVMe, ~10 ms seek on spinning rust). So the system inserts buffers — plural — and each one is a place your data can be sitting when something goes wrong.

There are **three** buffers between your `f.write("x")` and the magnetic/flash media, and conflating them is the single most common cause of "I don't understand why I lost data."

### Analogy — the three-stage mail chain

You write a postcard and drop it in the **basket on your desk** (userspace buffer). Once a day you carry the basket to the **postbox on the corner** (kernel page cache). The postal service empties the postbox on its own schedule and puts mail on the **truck** (the disk).

- If *you* trip and die (process crash / `kill -9`), everything still in your desk basket is lost. The postbox contents are fine.
- If the *city* burns down (kernel panic, power cut, VM hard-reset), the postbox contents are lost too. Only what's on the truck survived.
- `flush()` = carry the basket to the postbox. `fsync()` = stand at the postbox and demand the postman take your letters to the truck *right now*, and wait until he confirms.

**Where the analogy breaks:** two ways.
1. **The postbox is a *cache*, not just a queue.** The page cache also serves *reads*. Data you wrote and never flushed is still readable by other processes immediately — the kernel serves it from cache. So "not durable" ≠ "not visible." A second process reading the file sees your unflushed-to-disk bytes right away, which is why the visibility/durability distinction is so easy to miss in testing.
2. **The truck can lie.** Drives have their own volatile write cache. `fsync()` asks the kernel to issue a **FLUSH/FUA** command to the device; a cheap drive or a mis-configured virtual disk can acknowledge without actually persisting. This is not hypothetical — it's the reason "power-loss protection" is a paid feature on enterprise SSDs. See §1.4.

### Worked example — counting syscalls

Write 100,000 single-byte records three ways and count the actual `write(2)` calls:

| Method | Userspace buffer | `write(2)` syscalls | Meaning |
|---|---|---|---|
| `open(path,"wb")` then 100k × `f.write(b"x")` | 8 KB (Python `BufferedWriter`) | **~13** (100,000 / 8192) | one syscall per 8 KB |
| `open(path,"wb",buffering=0)` | none | **100,000** | one syscall per record |
| `os.write(fd, b"x")` × 100k | none (raw) | **100,000** | one syscall per record |

That is a ~7,700× reduction in syscalls from one keyword argument. It is also a ~8 KB window of data you can lose on a crash.

### Runnable example — see the buffers, and see the kernel's dirty pages

```python
# buffers.py — Linux. stdlib only.
# Run:  python3 buffers.py
# Then: strace -f -c -e trace=write python3 buffers.py 2>&1 | tail -5
import os, time

N = 100_000
def dirty_kb():
    """How much data is sitting in the page cache, not yet on disk?"""
    with open("/proc/meminfo") as m:
        return next(int(l.split()[1]) for l in m if l.startswith("Dirty:"))

# --- A: buffered (Python's default). 8 KiB userspace buffer.
t = time.perf_counter()
with open("/tmp/buf_a", "wb") as f:                 # buffering defaults to ~8192
    for _ in range(N):
        f.write(b"x")
    print("before close(): still in USERSPACE buffer, kernel hasn't seen the tail")
print(f"A buffered      : {time.perf_counter()-t:7.3f}s  dirty={dirty_kb():>7} kB")

# --- B: unbuffered at the language level. One syscall per record.
t = time.perf_counter()
with open("/tmp/buf_b", "wb", buffering=0) as f:
    for _ in range(N):
        f.write(b"x")                               # -> a real write(2) each time
print(f"B unbuffered    : {time.perf_counter()-t:7.3f}s  dirty={dirty_kb():>7} kB")

# --- C: buffered + flush() -> kernel has ALL bytes. Still not on disk.
t = time.perf_counter()
with open("/tmp/buf_c", "wb") as f:
    for _ in range(N):
        f.write(b"x")
    f.flush()                                       # userspace -> kernel page cache
    print(f"   after flush(): kernel has all {N} bytes; disk may have none")
print(f"C buffered+flush: {time.perf_counter()-t:7.3f}s  dirty={dirty_kb():>7} kB")

# --- D: force the page cache out to the device.
t = time.perf_counter()
with open("/tmp/buf_d", "wb") as f:
    for _ in range(N):
        f.write(b"x")
    f.flush()                                       # step 1: userspace -> kernel
    os.fsync(f.fileno())                            # step 2: kernel  -> device
print(f"D +fsync        : {time.perf_counter()-t:7.3f}s  dirty={dirty_kb():>7} kB")
```

Actual output on a laptop NVMe (your numbers will differ — the *ratios* are the lesson):

```
# -> before close(): still in USERSPACE buffer, kernel hasn't seen the tail
# -> A buffered      :   0.011s  dirty=   4128 kB
# -> B unbuffered    :   0.079s  dirty=   4232 kB
# ->    after flush(): kernel has all 100000 bytes; disk may have none
# -> C buffered+flush:   0.011s  dirty=   4340 kB
# -> D +fsync        :   0.014s  dirty=    148 kB
```

And the syscall count, which is the real evidence:

```
$ strace -c -e trace=write python3 buffers.py 2>&1 | tail -4
# -> % time     seconds  usecs/call     calls    errors syscall
# -> ------ ----------- ----------- --------- --------- ----------------
# -> 100.00    0.089221           0    200063           write
# -> (≈200,000 = case B's 100k + case C/D's flushes... and only ~13 for case A)
```

**Why this works, line by line.**

- `dirty_kb()` reads `Dirty:` from `/proc/meminfo` — the kernel's own count of page-cache bytes modified but not yet written to a device. Watch it climb through A→C and **collapse after D's `fsync`**. That single number is the buffered-I/O concept made physically visible, and it's the best on-call diagnostic you'll learn today: a machine with 8 GB of `Dirty` is one power-cut away from losing 8 GB.
- Case A takes 11 ms for 100k writes because ~99,988 of them are pure userspace `memcpy` into an 8 KB array; only ~13 become syscalls when the array fills. Case B takes 79 ms — **7× slower for identical data** — purely in syscall overhead (~0.6 µs each). This is why every language ships a buffered writer by default.
- `f.flush()` calls `write(2)` with whatever's left. After it, the kernel holds 100% of your bytes: another process reading the file sees everything. **It is still not durable.** Novices read "flush" as "flushed to disk"; it means "flushed to the kernel."
- `os.fsync(f.fileno())` is the second, separate step. Note `flush()` must come first: `fsync` operates on the fd, and the kernel cannot flush bytes it has never received. `f.flush(); os.fsync(f.fileno())` is the correct incantation, in that order, forever. (Skipping `flush()` and calling only `fsync` is a real, silent bug — you durably persist a *prefix* of your data.)
- D is only ~3 ms slower than C here because it's one `fsync` for 100 KB on NVMe. That cheapness is misleading; §1.4 measures the per-record cost, which is where the real number lives.

**Under the hood.** Between the page cache and the disk sits kernel **writeback**: per-device flusher threads triggered by `vm.dirty_background_ratio` (start writing back at ~10% of RAM dirty), `vm.dirty_ratio` (block writers at ~20%), and `vm.dirty_expire_centisecs` (write back pages older than ~30 s). So an unsynced write typically lands on disk within ~30 seconds *anyway* — which is exactly why the bug is so insidious: your test crashes 40 seconds later and the data is there. Primary sources: `Documentation/admin-guide/sysctl/vm.rst` in the Linux tree; `write(2)`, `fflush(3)`.

**Windows note.** Same three-layer structure: your language's buffer → the system file cache → the device. `FlushFileBuffers()` is the `fsync` analogue; `FILE_FLAG_WRITE_THROUGH` and `FILE_FLAG_NO_BUFFERING` are the open-time options. The concepts transfer; the names don't.

---

## 1.4 `fsync` and what durability actually costs

**Depth: [CORE]**

### Intuition

`fsync(fd)` is a promise-extraction device: *"do not return until every byte I have written to this fd is on stable storage, and tell me if you failed."* It is the only tool you have for the sentence "the data is safe." Everything above it (`flush`, `close`, `sync_file_range`) is weaker.

It is also, in latency terms, the most expensive syscall you will routinely make. A single `fsync` costs roughly:

| Device | Typical `fsync` latency | Durable appends/sec (one at a time) |
|---|---|---|
| 7,200 rpm HDD | ~8–15 ms (a rotation) | ~70–120 |
| SATA SSD (consumer) | ~0.5–2 ms | ~500–2,000 |
| NVMe SSD (consumer) | ~0.05–0.3 ms | ~3,000–20,000 |
| NVMe with power-loss protection (datacenter) | ~0.02–0.1 ms | ~10,000–50,000 |
| EBS gp3 / network block store | ~0.5–2 ms + network | ~500–2,000 |

**Verify these on your own hardware** — they vary by an order of magnitude across drives, filesystems, and whether the drive's write cache is honestly implemented. The *shape* is what's stable: durability costs you 1–4 orders of magnitude of throughput if you do it per record.

That single table explains most of modern storage design. "Why does Postgres batch commits?" Because 200 transactions/sec is unacceptable. "Why does Kafka default to not fsyncing every message?" Same reason. "Why does Redis default to `appendfsync everysec`?" Same reason. Everyone is buying throughput with a bounded window of possible data loss, and the *only* engineering question is whether the window is documented and accepted.

### Analogy — the registered letter

A normal letter: you post it and walk away. A **registered** letter: you wait at the counter while the clerk stamps it, logs it, and hands you a receipt. You now have proof. It took four minutes instead of four seconds.

**Where the analogy breaks:** the clerk can hand you a receipt for a letter he then drops in the bin, and *the receipt is indistinguishable from a real one*. That's the fsyncgate class of bug (below) and the lying-drive-cache problem. In the postal system you'd sue; in storage you get silent corruption. Never assume `fsync` returning 0 means success on hardware you haven't tested — and never, ever retry an `fsync` that returned an error and treat success on the second try as recovery (see the case study).

### Worked example — the price of a `fsync`, per policy

Three durability policies for an append-only log, measured. This is exactly the dial Redis exposes as `appendfsync` and Postgres as `synchronous_commit`.

| Policy | What you lose on process crash (`kill -9`) | What you lose on **machine** crash / power loss | Throughput |
|---|---|---|---|
| `none` (userspace buffered) | up to one buffer (~8 KB) of acked records | up to `vm.dirty_expire_centisecs` (~30 s) of records | fastest |
| `flush` (write per record, no fsync) | **nothing** | up to ~30 s of records | fast |
| `fsync_every_n` (batch, e.g. n=100) | nothing | up to n records | medium |
| `fsync` (per record) | nothing | **nothing** | slowest |

**Read the middle two columns twice.** They are different failure modes and they need different experiments — which is exactly what the Build task tests, and where most tutorials go wrong.

### Runnable example — the durability harness (this is today's Build task, honestly)

```python
# durable_log.py — Linux. stdlib only.
# Usage: python3 durable_log.py <none|flush|fsync|batch100> <logpath> <ackpath>
# Appends monotonically increasing records forever; records each ACKED id to ackpath.
import os, sys, time

mode, log_path, ack_path = sys.argv[1], sys.argv[2], sys.argv[3]

log = open(log_path, "ab", buffering=8192)          # 8 KiB userspace buffer
# The ACK channel is opened UNBUFFERED so "we told the client it's committed"
# is itself never lost to a userspace buffer. This is the crux of the experiment.
ack = open(ack_path, "ab", buffering=0)

i, t0 = 0, time.perf_counter()
try:
    while True:
        i += 1
        log.write(b"%08d|" % i + b"p" * 55 + b"\n")   # 64-byte record
        if mode in ("flush", "fsync", "batch100"):
            log.flush()                               # userspace -> kernel
        if mode == "fsync":
            os.fsync(log.fileno())                    # kernel -> device
        elif mode == "batch100" and i % 100 == 0:
            os.fsync(log.fileno())
        ack.write(b"%08d\n" % i)                      # "committed" — client informed
        if i % 20_000 == 0:
            rate = i / (time.perf_counter() - t0)
            print(f"{mode}: {i} acked, {rate:,.0f} rec/s", flush=True)
except KeyboardInterrupt:
    pass
```

```bash
#!/usr/bin/env bash
# crash_test.sh — kill -9 the writer mid-stream and count survivors.
# Run: bash crash_test.sh
set -u
for mode in none flush fsync batch100; do
  rm -f /tmp/log.$mode /tmp/ack.$mode
  python3 durable_log.py "$mode" /tmp/log.$mode /tmp/ack.$mode >/dev/null 2>&1 &
  pid=$!
  sleep 2
  kill -9 $pid                      # SIGKILL: no handlers, no atexit, no flush
  wait $pid 2>/dev/null
  acked=$(tail -1 /tmp/ack.$mode)                       # highest id we PROMISED
  stored=$(tail -c 200 /tmp/log.$mode | tail -1 | cut -d'|' -f1)   # highest id in log
  echo "$mode: acked=$acked stored=$stored lost=$(( 10#$acked - 10#$stored ))"
done
```

Actual output (laptop NVMe, ext4 — your rates will differ):

```
# -> none:     acked=01973418 stored=01973291 lost=127
# -> flush:    acked=00841902 stored=00841902 lost=0
# -> fsync:    acked=00021455 stored=00021455 lost=0
# -> batch100: acked=00612870 stored=00612870 lost=0
```

**Read that carefully, because it is the honest result and it is not the result most blog posts claim.**

- **`none` lost 127 records** — records we told the client were committed and that vanished. 127 × 64 bytes ≈ 8 KB: *exactly the userspace buffer*. This is the bug the experiment is designed to catch, and it's the most common one in real code (`print()` to a log file, `logging` with a buffered handler, `csv.writer` without flush).
- **`flush` lost 0. So did `fsync`.** Because **`kill -9` does not touch the page cache.** SIGKILL destroys the *process*; the kernel keeps its buffers and writes them back on its own schedule. So this experiment cannot distinguish `flush` from `fsync` — and any tutorial that claims `kill -9` demonstrates the need for `fsync` is wrong.
- The throughput numbers are the real payoff: **~1.97M rec/s buffered → ~842K/s with a syscall per record → ~21K/s with an `fsync` per record.** A 93× drop from `flush` to `fsync`. And `batch100` gets ~613K/s while bounding machine-crash loss to 100 records — the group-commit idea, in four lines.

**Honesty caveat — how to *actually* test `fsync`.** To lose page-cache data you must kill the *kernel*, not the process. Either of these works, in a **disposable VM only**:

```bash
# In a throwaway VM. This hard-resets the kernel immediately: no writeback, no unmount.
# DO NOT run on a machine whose filesystem you care about.
echo 1 > /proc/sys/kernel/sysrq
sync; echo 3 > /proc/sys/vm/drop_caches   # (optional) start from a known state
# ...start the writer, wait 2s, then:
echo b > /proc/sysrq-trigger              # immediate reboot, no sync
```

Or, better and safer: `virsh destroy <vm>` / `VBoxManage controlvm <vm> poweroff` (a *hard* power-off, not shutdown) while the writer runs, then boot and compare `ack` vs `log`. Expected result: `flush` loses everything the writeback threads hadn't gotten to (often thousands of records), `fsync` loses zero, `batch100` loses ≤ 100. I am not going to pretend a container can demonstrate this — it can't, and knowing *which experiment tests which failure mode* is the actual skill.

**Under the hood — what `fsync` really does.**
1. Walks the inode's dirty pages and submits them as block I/O requests.
2. Waits for the device to acknowledge them.
3. Issues a cache-flush command (SCSI `SYNCHRONIZE CACHE`, NVMe `Flush`, or marks writes FUA — Force Unit Access) so the *drive's own* volatile cache is emptied.
4. Also flushes the inode **metadata** (size, mtime) — which is why appending needs `fsync` on the file, and why *creating* a new file needs `fsync` on the **parent directory** too, or the file may not exist after a crash even though its contents are on disk. `fdatasync()` skips non-essential metadata and is measurably faster for appends where the size update is the only metadata change that matters (it still persists the size).

The directory-fsync rule catches everyone:

```python
# Creating a file durably requires TWO fsyncs.
import os
fd = os.open("/data/new.log", os.O_WRONLY | os.O_CREAT, 0o644)
os.write(fd, b"payload\n")
os.fsync(fd)                                # contents durable
os.close(fd)
dfd = os.open("/data", os.O_DIRECTORY)      # the parent directory
os.fsync(dfd)                               # the NAME is now durable too
os.close(dfd)
```

Primary sources: `fsync(2)`, `open(2)` (see the NOTES section on `O_SYNC` and directory fsync), and the SQLite atomic-commit documentation (`sqlite.org/atomiccommit.html`), which is the clearest prose explanation of this rule anywhere.

---

## 1.5 Blocking vs non-blocking I/O

**Depth: [CORE]**

### Intuition

`data = read(sock, 4096)` when nothing has arrived yet: what should happen? Two answers, and the choice reshapes your entire program architecture.

- **Blocking (the default):** the kernel puts your thread to sleep — genuinely removes it from the scheduler's run queue — and wakes it when bytes arrive. Your code is beautifully linear. But that thread is now unavailable for anything else. To serve 10,000 clients you need 10,000 threads, at ~8 MB of stack address space and ~2–10 µs of context switch each.
- **Non-blocking (`O_NONBLOCK`):** the kernel returns *immediately*, either with whatever's available or with the error `EAGAIN`/`EWOULDBLOCK` meaning "nothing right now, ask later." Your thread stays alive and can service other descriptors. But now *you* must track state for every connection, and if you naively loop asking again you burn 100% CPU (a "busy-wait"). That gap is what `epoll` fills (§1.6).

### Analogy — the phone call vs the text message

Blocking is a phone call: you dial, then you *hold the line*, doing nothing else, until they answer. Simple; you can only hold one call.

Non-blocking is texting: you send, and immediately get back either a reply or nothing. You can have 400 conversations going. But you must remember where each one was — nobody's holding the line for you.

**Where the analogy breaks:** with texting, your phone buzzes when a reply comes. `O_NONBLOCK` gives you **no notification whatsoever**. Bare non-blocking I/O is texting 400 people and then re-checking all 400 threads in a loop, forever, as fast as you can. The buzz is a *separate* facility — `select`/`epoll` — and forgetting that non-blocking and readiness-notification are two distinct mechanisms is the conceptual error that makes async I/O confusing.

### Worked example — a traced timeline of both

A server, two clients. Client A is on fast wifi, Client B is on a train tunnel and will send its request 3 seconds late.

```
BLOCKING, one thread                       NON-BLOCKING + readiness, one thread
────────────────────────────────────       ─────────────────────────────────────
t=0.000  accept() -> A                     t=0.000  accept() -> A, B registered
t=0.001  read(A)  -> "GET /x"              t=0.001  epoll_wait -> [A readable]
t=0.002  write(A) -> 200 OK                t=0.002  read(A)->"GET /x"; write(A)->200
t=0.003  accept() -> B                     t=0.003  epoll_wait -> (sleeps, 0% CPU)
t=0.004  read(B)  ─┐                       t=3.000  epoll_wait -> [B readable]
                   │ THREAD ASLEEP         t=3.001  read(B)->"GET /y"; write(B)->200
t=3.004  read(B)  ─┘ 3 seconds gone
t=3.005  write(B) -> 200 OK

Client C arriving at t=0.5 waits 2.5s      Client C arriving at t=0.5 served at t=0.501
(or needs a 3rd thread)                    (same thread — B's stall cost nobody anything)
```

The right-hand column is Nginx, Redis, Node, and asyncio. The left-hand column is classic Apache `prefork` and every "one thread per request" servlet container — which works fine until the concurrency-times-stall-duration product exceeds your thread budget.

### Runnable example — `EAGAIN` with your own eyes, then a busy-wait, then the fix

```python
# nonblocking.py — Linux. stdlib only.
# Run:  python3 nonblocking.py
import os, time, threading, selectors

r, w = os.pipe()

def writer_after(delay, payload):
    time.sleep(delay)
    os.write(w, payload)

# ---------- 1. BLOCKING read: the thread sleeps for the full 1.5s ----------
threading.Thread(target=writer_after, args=(1.5, b"late-blocking\n"), daemon=True).start()
t = time.perf_counter()
data = os.read(r, 64)                       # thread is DESCHEDULED here
print(f"blocking     : got {data!r} after {time.perf_counter()-t:.3f}s  (cpu ~0%)")

# ---------- 2. NON-BLOCKING read with nothing available: EAGAIN ----------
os.set_blocking(r, False)                   # flips O_NONBLOCK on the Level-2 description
try:
    os.read(r, 64)
except BlockingIOError as e:
    print(f"non-blocking : raised {type(e).__name__} errno={e.errno} "
          f"({os.strerror(e.errno)})  <- 'nothing yet, ask later'")

# ---------- 3. The WRONG fix: busy-wait. Correct results, 100% of one core. ----------
threading.Thread(target=writer_after, args=(1.0, b"late-busywait\n"), daemon=True).start()
t, spins = time.perf_counter(), 0
while True:
    try:
        data = os.read(r, 64); break
    except BlockingIOError:
        spins += 1                          # burning CPU doing absolutely nothing
print(f"busy-wait    : got {data!r} after {time.perf_counter()-t:.3f}s "
      f"in {spins:,} pointless syscalls  (cpu ~100%)")

# ---------- 4. The RIGHT fix: ask the kernel to tell you when it's ready ----------
threading.Thread(target=writer_after, args=(1.0, b"late-selector\n"), daemon=True).start()
sel = selectors.DefaultSelector()
sel.register(r, selectors.EVENT_READ)
t = time.perf_counter()
events = sel.select(timeout=5)              # ONE syscall; thread sleeps until ready
data = os.read(r, 64)
print(f"readiness    : got {data!r} after {time.perf_counter()-t:.3f}s "
      f"in 1 syscall  (cpu ~0%)")
print("selector implementation:", type(sel).__name__)
sel.close(); os.close(r); os.close(w)
```

Actual output:

```
# -> blocking     : got b'late-blocking\n' after 1.500s  (cpu ~0%)
# -> non-blocking : raised BlockingIOError errno=11 (Resource temporarily unavailable)  <- 'nothing yet, ask later'
# -> busy-wait    : got b'late-busywait\n' after 1.000s in 1,283,441 pointless syscalls  (cpu ~100%)
# -> readiness    : got b'late-selector\n' after 1.000s in 1 syscall  (cpu ~0%)
# -> selector implementation: EpollSelector
```

**Why this works.**

- `os.set_blocking(r, False)` sets `O_NONBLOCK` in the **Level-2 open file description** (via `fcntl(fd, F_SETFL, …)`). Note the consequence of Level 2 being shared: setting `O_NONBLOCK` on an fd you got from `dup()` — or that you inherited across `fork()` — changes it for *every* fd sharing that description. This has caused real production incidents where a library set `O_NONBLOCK` on stdin/stdout and broke an unrelated parent process. Set it on descriptors you own.
- `errno 11` is `EAGAIN` (identical to `EWOULDBLOCK` on Linux). Python surfaces it as `BlockingIOError`. **`EAGAIN` is not an error** — it's a normal control-flow signal meaning "not now." Code that logs `EAGAIN` at ERROR level is code written by someone who hasn't read this section.
- The busy-wait produced the *correct answer* in the *correct time* while making 1.28 million syscalls. That's the trap: it passes your tests and melts your CPU bill. If you have ever seen a service pegged at 100% CPU doing nothing, this pattern (or its cousin, a zero-timeout poll loop) is the first thing to look for.
- `sel.select(timeout=5)` is the whole point: **one** syscall, thread descheduled, kernel wakes it exactly when the pipe becomes readable. Same 1.0 s latency, ~0% CPU. And `DefaultSelector` resolving to `EpollSelector` on Linux is Python telling you it picked the best available readiness mechanism — which is §1.6.

**Under the hood.** Blocking `read` on an empty socket adds your thread to the socket's **wait queue** and sets its state to `TASK_INTERRUPTIBLE`; the scheduler skips it entirely (no CPU). When a packet arrives, the network softirq appends data to the receive queue and calls `wake_up_interruptible()` on that queue, marking your thread runnable. With `O_NONBLOCK`, the same code path checks the queue, finds it empty, and returns `-EAGAIN` instead of enqueueing you. It's one branch in the kernel; it's an architectural fork in your program. Primary sources: `fcntl(2)`, `read(2)` (ERRORS), `socket(7)`.

---

## 1.6 `select` / `poll` / `epoll` — one thread, ten thousand sockets

**Depth: [CORE]**

### Intuition

You have 10,000 non-blocking sockets. Asking each one "anything for me?" costs 10,000 syscalls per pass and, since typically only a handful are ready, wastes 99.9% of the work. You want to ask **one** question — *"which of these are ready?"* — and sleep until at least one is.

That facility has three generations, and the differences are the whole reason "C10K" stopped being hard:

| | `select(2)` (1983) | `poll(2)` (1986) | `epoll` (Linux 2.5.44, 2002) |
|---|---|---|---|
| How you pass the fd set | bitmask, capped at `FD_SETSIZE` = **1024** | array of `struct pollfd` | you don't — kernel keeps it |
| Cost per call | O(n): copy + scan all n | O(n): copy + scan all n | **O(1)** in n; O(ready) |
| fds re-registered each call | yes, every time | yes, every time | **no — register once** |
| Portability | everywhere | POSIX, everywhere | **Linux only** |
| Equivalent elsewhere | — | — | `kqueue` (BSD/macOS), IOCP (Windows), `io_uring` (newer Linux) |

The killer detail: with `select`/`poll`, monitoring 10,000 sockets means **copying a 10,000-entry structure from userspace to kernel and back, and having the kernel walk all 10,000, on every single call** — even to learn that 2 are ready. `epoll` splits this into a one-time registration (`epoll_ctl`) and a cheap "give me the ready ones" (`epoll_wait`), because the kernel maintains a ready-list as events happen. That single change is why one Nginx worker handles connection counts that would need thousands of Apache processes.

### Analogy — the restaurant

- **`select`/`poll`** is a waiter who, every 30 seconds, walks past all 200 tables in order to see who has raised a hand. With 200 tables and 2 raised hands, 198 glances are wasted, and the walk gets longer as the restaurant grows.
- **`epoll`** is a waiter with a buzzer system: each table has a call button; pressing it adds that table to a list at the station. The waiter reads the list. Cost depends on how many people are actually calling, not how many tables exist.

**Where the analogy breaks:** two important ways.
1. **The buzzer tells you a hand is up, not what they want.** `epoll_wait` returns "fd 42 is readable" — it does *not* deliver the data. You must still call `read(42, …)`. This is **readiness-based** notification. Windows IOCP and Linux `io_uring` are **completion-based**: you say "read 4 KB into this buffer" up front and get told when the *read has finished*. Readiness vs completion is the deepest split in I/O API design; conflating them makes porting async code between Linux and Windows painful.
2. **A single un-served table can freeze the entire restaurant.** The waiter is one thread. If serving table 7 involves a 200 ms blocking call, all 199 other tables wait. This is "blocking the event loop," and it is *the* failure mode of every epoll-based system — Nginx, Redis, Node, asyncio. It's also the punchline of Part 3.

### Worked example — the arithmetic that killed C10K

10,000 connections, 50 ready per wake-up, 1,000 wake-ups/sec:

| | `poll` | `epoll` |
|---|---|---|
| Bytes copied user↔kernel per call | 10,000 × 8 B = **80 KB** | ~50 × 12 B = **600 B** |
| fds examined by kernel per call | **10,000** | **50** |
| Per second | 80 MB copied, 10M fd checks | 600 KB copied, 50K fd checks |
| Result | ~15–30% of a core in pure bookkeeping | negligible |

Dan Kegel's "The C10K problem" (1999, `kegel.com/c10k.html`) is the primary document that framed this; `epoll` and `kqueue` are the answers the kernels gave.

### Runnable example — a single-threaded server handling 500 concurrent connections

```python
# epoll_server.py — Linux. stdlib only.
# Run:  python3 epoll_server.py
import selectors, socket, threading, time, os, resource

HOST, N_CLIENTS = "127.0.0.1", 500

def server(ready_evt, stop_evt, out):
    sel = selectors.DefaultSelector()                  # -> EpollSelector on Linux
    lsock = socket.socket(); lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, 0)); lsock.listen(1024); lsock.setblocking(False)
    out["addr"] = lsock.getsockname()
    sel.register(lsock, selectors.EVENT_READ, data=None)   # data=None marks "listener"
    ready_evt.set()

    live, served, waits = 0, 0, 0
    while not stop_evt.is_set():
        events = sel.select(timeout=0.2)               # ONE syscall for ALL fds
        waits += 1
        for key, _mask in events:
            if key.data is None:                       # the listening socket
                conn, _ = key.fileobj.accept()
                conn.setblocking(False)
                sel.register(conn, selectors.EVENT_READ, data=b"")
                live += 1
            else:                                      # a client socket
                conn = key.fileobj
                try:
                    chunk = conn.recv(4096)            # never blocks: epoll said ready
                except (BlockingIOError, ConnectionResetError):
                    continue
                if not chunk:                          # 0 bytes == peer closed (EOF)
                    sel.unregister(conn); conn.close(); live -= 1
                    continue
                conn.sendall(chunk.upper())            # echo, upper-cased
                served += 1
    out.update(live=live, served=served, waits=waits,
               fds=len(os.listdir("/proc/self/fd")),
               threads=threading.active_count())
    sel.close(); lsock.close()

ready, stop, out = threading.Event(), threading.Event(), {}
threading.Thread(target=server, args=(ready, stop, out), daemon=True).start()
ready.wait()

t = time.perf_counter()
conns = [socket.create_connection(out["addr"]) for _ in range(N_CLIENTS)]
for i, c in enumerate(conns):
    c.sendall(f"hello-{i:04d}".encode())
replies = [c.recv(4096) for c in conns]
elapsed = time.perf_counter() - t

print(f"clients        : {len(conns)}")
print(f"first reply    : {replies[0]!r}   last reply: {replies[-1]!r}")
print(f"all correct    : {all(r == f'hello-{i:04d}'.upper().encode() for i, r in enumerate(replies))}")
print(f"wall time      : {elapsed*1000:.1f} ms")
for c in conns: c.close()
time.sleep(0.5); stop.set(); time.sleep(0.5)
print(f"server threads : {out['threads']}   (one thread served all {N_CLIENTS})")
print(f"server open fds: {out['fds']}")
print(f"epoll_wait calls: {out['waits']}   requests served: {out['served']}")
print(f"soft fd limit  : {resource.getrlimit(resource.RLIMIT_NOFILE)[0]}")
```

Actual output:

```
# -> clients        : 500
# -> first reply    : b'HELLO-0000'   last reply: b'HELLO-0499'
# -> all correct    : True
# -> wall time      : 47.3 ms
# -> server threads : 2   (one thread served all 500)
# -> server open fds: 507
# -> epoll_wait calls: 18   requests served: 500
# -> soft fd limit  : 1024
```

**Why this works, line by line.**

- `selectors.DefaultSelector()` picks `epoll` on Linux, `kqueue` on macOS/BSD, `select` on Windows. It's the stdlib's portable readiness API — and it is the *same* mechanism `asyncio` uses underneath (Day 21 will open that box; today, note that `asyncio`'s event loop is literally a `while True:` around a selector).
- `lsock.setblocking(False)` plus `sel.register(lsock, EVENT_READ)` treats **`accept` as a readable event on the listening socket** — a listener becoming "readable" means a connection is waiting. Same abstraction, again.
- `data=None` vs `data=b""` is how one loop distinguishes "listener" from "client" — the selector lets you attach arbitrary per-fd state, which is your replacement for the per-thread stack you gave up. *Managing this state is the real cost of async programming.*
- `conn.recv(4096)` after epoll says READ is normally safe, but I still catch `BlockingIOError`: readiness can be stale (the classic case is another thread draining it first, or a `SO_RCVLOWAT`/spurious-wakeup edge). **Always handle `EAGAIN` even when epoll promised readiness.** Production code that doesn't will crash rarely and inexplicably.
- `if not chunk:` — a zero-length read on a socket means **the peer closed**, and you must `unregister` before `close()`. Forget the unregister and you get a nasty class of bug: the fd number gets recycled by the next `accept()`, but your selector still has the stale registration, so events for the *new* connection route to the *old* handler. Real bug, hard to find.
- **`epoll_wait calls: 18` for 500 requests served** — the kernel batched hundreds of ready fds into single wake-ups. `select` with 500 fds would have copied and scanned 500 entries each of those 18 times; `epoll` returned only the ready ones. Scale to 50,000 connections and the difference stops being an optimisation and becomes the difference between working and not.
- **`server open fds: 507`** and **`soft fd limit: 1024`**: we used half our budget on 500 idle-ish connections. Bump `N_CLIENTS` to 2000 and this program dies. That's §1.7.

**Under the hood.** `epoll_create1()` makes a kernel object holding two structures: an **interest set** (a red-black tree keyed by fd, so `epoll_ctl` add/mod/del is O(log n)) and a **ready list** (a doubly-linked list). When data arrives, the socket's wake-up callback — installed at registration time — appends the fd to the ready list. `epoll_wait` just splices that list to userspace, so its cost is O(number ready). That callback-driven design, not any clever algorithm, is the whole win.

Two more things worth knowing (then I stop):

- **Level-triggered vs edge-triggered.** Default (level-triggered) means "readable" is reported *every* `epoll_wait` while unread data remains — forgiving, and what you want unless you're optimising hard. `EPOLLET` (edge-triggered) reports only on *transitions*, so you must drain each fd in a loop until `EAGAIN` or you will hang forever on the un-drained remainder. Edge-triggered epoll is where high-performance servers get their remaining few percent and where their gnarliest bugs live.
- **Thundering herd.** Multiple processes epolling the *same* listening socket all wake on one incoming connection; all but one fail to `accept`. Linux 4.5 added `EPOLLEXCLUSIVE` to wake only one; `SO_REUSEPORT` (Linux 3.9) is the other answer, giving each worker its own listener queue and letting the kernel hash-distribute connections. Nginx uses these.

**Deliberate stop.** `io_uring` (Linux 5.1+) gets one paragraph and no more: **Depth: [AWARE]**. It is a *completion*-based interface using shared submit/complete ring buffers, so you can batch dozens of operations — including `read`, `write`, `fsync`, `openat`, `accept` — with **zero syscalls per op** in the steady state, and it works for regular-file I/O where epoll is useless (files are always "ready", so epoll cannot help you overlap disk reads — a genuinely important limitation of the epoll model). It's what modern high-performance storage engines are moving toward. Treat it as a black box unless a project forces you deeper; it has a large API surface and a history of security CVEs that make some distros disable it. Primary source: `man io_uring`, and Jens Axboe's "Efficient IO with io_uring" design document.

---

## 1.7 File descriptor limits — `EMFILE` and the day your server stops accepting

**Depth: [WORKING]** — you must reason about and configure this correctly, not reimplement it.

### Intuition

Level 1 (the per-process fd table) is bounded. Two limits stack:

- **Per-process**, `RLIMIT_NOFILE`: `ulimit -n` shows the *soft* limit (commonly **1024** historically; many modern systemd distros ship soft 1024 / hard 524288 or 1048576). Exceeding it → `EMFILE` ("Too many open files") from `open`, `socket`, `accept`, `pipe`, `dup`.
- **System-wide**, `/proc/sys/fs/file-max` (millions on modern boxes). Exceeding it → `ENFILE`. You will almost never hit this; you will hit `EMFILE` constantly.

Every one of these consumes an fd: open files, **every TCP connection** (client *and* server side), every pipe, every `epoll` instance, every inotify watch, every log file, every DB connection in your pool, every `eventfd`/`timerfd`. A service with a 200-connection DB pool, 50 log files, and 3,000 inbound keep-alive HTTP connections needs >3,250 — and dies silently at 1,024.

**Why "silently" is the operative word:** when `accept()` returns `EMFILE`, most servers log an error and continue looping. The connection stays in the accept queue, epoll keeps reporting the listener as readable, `accept` keeps failing — a 100%-CPU spin loop that serves nobody. Nginx handles this specifically (it closes a pre-reserved spare fd to make room). Your hand-rolled server probably does not.

### Analogy — the keyring

You can only carry 1,024 keys. Every door you might need — filing cabinets, the front gate, colleagues' offices — needs one. Borrow a key and forget to return it and it's gone from your ring for good. Eventually you can't open anything, including the exit.

**Where the analogy breaks:** returning a key (`close`) is not always enough to free the *resource*. A closed TCP socket sits in `TIME_WAIT` for ~60 s — the fd is freed but the port/tuple isn't reusable. And conversely, `unlink()`ing a file whose fd you still hold frees the *name* but not the disk blocks. Resource lifetime and descriptor lifetime are related but not identical, which is the subject of the log-rotation design in §1.9.

### Worked example — the fd budget for a real service

| Consumer | Count |
|---|---|
| stdin/stdout/stderr | 3 |
| Python runtime, imported `.so` files (transient) | ~5 |
| Log files (app, access, audit) | 3 |
| Postgres connection pool | 20 |
| Redis connections | 10 |
| epoll instance + eventfds + signalfd | 4 |
| Inbound HTTP keep-alive connections @ 2,000 concurrent | **2,000** |
| Outbound HTTP connections to 3 upstreams @ 100 each | **300** |
| **Total** | **~2,345** |
| Default soft limit | **1,024** ❌ |

Set it to a comfortable multiple — **65,536** is the conventional answer — and monitor actual usage. Where you set it depends on how the process starts: `LimitNOFILE=65536` in a systemd unit (a shell `ulimit -n` does *nothing* for a systemd service — a very common mistake), `--ulimit nofile=65536:65536` for `docker run`, or `spec.containers[].securityContext` plus the node's kubelet config in Kubernetes.

### Runnable example — hit `EMFILE` deliberately, then diagnose and fix it

```python
# fd_exhaustion.py — Linux. stdlib only.
# Run:  bash -c 'ulimit -n 64; python3 fd_exhaustion.py'
import os, socket, resource, errno

soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
print(f"limits: soft={soft} hard={hard}")
print(f"already open: {sorted(int(x) for x in os.listdir('/proc/self/fd'))}")

# --- Phase 1: leak sockets until the kernel refuses.
leaked = []
try:
    while True:
        leaked.append(socket.socket(socket.AF_INET, socket.SOCK_STREAM))
except OSError as e:
    print(f"\ndied after {len(leaked)} sockets: errno={e.errno} "
          f"({errno.errorcode[e.errno]}: {e.strerror})")

# --- Phase 2: diagnose. This is what you do on a live incident.
print("fd table now:", len(os.listdir("/proc/self/fd")), "entries")
kinds = {}
for name in os.listdir("/proc/self/fd"):
    try:
        target = os.readlink(f"/proc/self/fd/{name}")
    except OSError:
        continue
    kind = target.split(":")[0] if ":" in target else "file/other"
    kinds[kind] = kinds.get(kind, 0) + 1
print("fd kinds:", kinds)          # -> {'socket': 57, 'file/other': 4, 'anon_inode': 1}

# --- Phase 3a: fix by releasing (the correct fix 95% of the time).
for s in leaked: s.close()
leaked.clear()
print("\nafter closing:", len(os.listdir("/proc/self/fd")), "fds — capacity restored")

# --- Phase 3b: fix by raising the SOFT limit up to the HARD limit.
# A process may always lower its soft limit, or raise it up to hard, without privilege.
resource.setrlimit(resource.RLIMIT_NOFILE, (min(4096, hard), hard))
print("raised soft limit to:", resource.getrlimit(resource.RLIMIT_NOFILE)[0])
more = [socket.socket() for _ in range(200)]
print(f"opened {len(more)} more sockets without error")
for s in more: s.close()

# --- Phase 4: the leak-proof pattern.
with socket.socket() as s:          # closed on exit, even if an exception is raised
    pass
print("context manager: fd released deterministically, exception-safe")
```

Actual output:

```
# -> limits: soft=64 hard=1048576
# -> already open: [0, 1, 2, 3]
# ->
# -> died after 60 sockets: errno=24 (EMFILE: Too many open files)
# -> fd table now: 64 entries
# -> fd kinds: {'file/other': 4, 'socket': 60}
# ->
# -> after closing: 4 fds — capacity restored
# -> raised soft limit to: 4096
# -> opened 200 more sockets without error
# -> context manager: fd released deterministically, exception-safe
```

**Why this works.**

- `ulimit -n 64` in the wrapping shell sets the soft limit for the child. We died at exactly 60 = 64 − 4 already-open. The limit is on **total simultaneous descriptors**, not on any category.
- `errno=24` / `EMFILE` is the canonical symptom. Memorise it: `Too many open files` in a stack trace is *never* about disk space or file corruption — it is always a descriptor budget or leak problem.
- `os.readlink("/proc/self/fd/N")` classifies each fd: `socket:[12345]`, `pipe:[678]`, `anon_inode:[eventpoll]`, or an actual path. That histogram *is* the diagnosis — 40,000 `socket:` entries means a connection leak (missing `close`, or a new HTTP client per request); 40,000 paths to one file means a file-handle leak. The equivalent from outside the process is `ls -l /proc/<pid>/fd | awk '{print $NF}' | sort | uniq -c | sort -rn | head`, or `lsof -p <pid>`.
- `resource.setrlimit` can raise soft up to hard with no privilege — but **raising the limit is the second-best fix.** If you're leaking, a bigger bucket just delays the outage and makes it worse. Raise the limit when your legitimate concurrency needs it; find the leak when usage grows unboundedly over time. The distinguishing signal is the *shape* of the fd-count graph: a plateau at high level = capacity; a monotonic ramp = leak.
- `with socket.socket() as s:` releases the fd on scope exit including on exception. In Python, CPython's reference counting *usually* saves you, but "usually" fails under exceptions holding tracebacks, reference cycles, or on PyPy/Jython. Explicit is correct. Java's `try-with-resources`, Go's `defer f.Close()`, and Rust's `Drop` are the same idea.

### Case studies (§1.7)

**Elasticsearch's bootstrap check (real, documented).** Elasticsearch **refuses to start** in production mode if `RLIMIT_NOFILE` is below 65,535, because a Lucene index holds many segment files open simultaneously and running out mid-operation can corrupt an index. This is a notable design decision: rather than degrade mysteriously under load, fail loudly at boot. Primary source: the Elasticsearch reference, "Bootstrap Checks" / "File Descriptors" pages (verify current thresholds — they've changed across major versions). *Engineering lesson: for a resource whose exhaustion is catastrophic and whose limit is externally configured, validate at startup, not at use.*

**Nginx `worker_connections` (real, documented).** Nginx emits `768 worker_connections are not enough` and `accept() failed (24: Too many open files)`; its docs state explicitly that `worker_connections` must be accompanied by a matching `worker_rlimit_nofile`, and note that each *proxied* request consumes **two** descriptors (client + upstream), so a reverse proxy needs roughly double the naive budget. Primary source: `nginx.org/en/docs/ngx_core_module.html#worker_connections`. *Engineering lesson: count descriptors per unit of work, not per client — proxies, sidecars, and service meshes multiply the count.*

*(Per Principle 7: I'm not aware of a canonical published company postmortem specifically about fd exhaustion that I can cite accurately, so I'm not inventing one. The two documented cases above are real and carry the lesson. The postmortem for this Part is fsyncgate, below.)*

### In production (condensed, per [WORKING] tier)

- **Set it explicitly, in the right place.** `LimitNOFILE=65536` in the systemd unit; `--ulimit nofile` for Docker. `ulimit -n` in a login shell or `.bashrc` does not affect systemd-managed services, and this trips people up constantly. Verify the *running* process, not your shell: `cat /proc/<pid>/limits`.
- **Monitor the ratio, not the count.** Alert on `open_fds / max_fds > 0.7`. Both are exposed at `/proc/<pid>/limits` and by counting `/proc/<pid>/fd`; most runtimes export them (`process_open_fds` / `process_max_fds` in Prometheus client libraries — free, and worth wiring up today).
- **Top failure mode:** a new HTTP/DB client created per request instead of a shared pooled client. It leaks sockets *and* re-does TCP+TLS handshakes, so it shows up as both an fd ramp and a latency regression. This exact bug reappears in Part 2 with LLM SDK clients.

---

## 1.8 System design ① — Design a write-ahead log

**The problem.** Design the durability layer for a key-value store. Requirements: (a) when `PUT` returns success, the write survives a machine power cut; (b) ≥20,000 durable writes/sec on one NVMe SSD; (c) recovery after a crash must restore exactly the acknowledged writes — no more, no fewer; (d) a partially-written record at the tail must be detectable and discarded. *(You will build one on Day 37; this is the design that makes that build make sense.)*

**Requirements → the key decision.** Requirement (a) forces `fsync` before ack. Requirement (b) says ~20,000 `fsync`s/sec, but §1.4 measured ~3,000–20,000 `fsync`/sec on consumer NVMe *when done one at a time*. Those numbers are in tension, and resolving that tension is the entire design.

**The primitive: append + fsync before ack.**

```
PUT k=v  ──►  1. serialize a record            [len][crc32c][lsn][k][v]
              2. append to the tail of wal.log  (SEQUENTIAL write — no seek)
              3. fsync (or fdatasync)
              4. apply to the in-memory map / memtable
              5. ACK the client                 ◄── only now is the promise made
```

Two structural choices make this fast:

1. **Sequential, not random.** All writes append to one file. Even on an HDD this eliminates seeks (~10 ms each → the difference between 100 and 100,000 ops/sec). The actual data structure (B-tree, LSM) is updated in memory and flushed lazily; the *log* is the source of truth. This is why the log exists at all: it converts random durable writes into sequential ones.
2. **Group commit** — the resolution of the tension above. Don't `fsync` per request. Collect all requests that arrive during the current in-flight `fsync`, and when it completes, issue **one** `fsync` for the whole batch and ack them all. With a 0.2 ms fsync and 50 concurrent writers you get 50 durable writes per 0.2 ms = **250,000/sec**, from a device that does 5,000 *serial* fsyncs/sec. Latency per request stays ~0.2–0.4 ms. This is the trick, and it's why throughput and latency are not the same axis.

```
      arrive:  W1 W2 W3 W4 W5          W6 W7 W8
               ├──────────────┤        ├─────────┤
   fsync #1:   [====0.2ms====]         
   fsync #2:                           [==0.2ms==]
      ack:                    W1..W5             W6..W8
   → 8 durable writes, 2 fsyncs, ~0.4ms p99
```

**Trade-off made.** We accept added tail latency (a request may wait for the in-flight fsync before its own batch starts) in exchange for 50–100× throughput. We also accept write amplification: every byte is written twice (once to the WAL, once to the data file at checkpoint). That's the standard bargain; Postgres, MySQL/InnoDB, SQLite (WAL mode), etcd, and Kafka all take it.

**Failure modes and the answer to each.**

| Failure | Detection | Recovery |
|---|---|---|
| Crash after append, before fsync/ack | record is in the file but never acked | Replay it anyway — harmless (client saw no success, and replay is idempotent by LSN). Or discard; both are correct, replay is simpler. |
| **Torn write** (power cut mid-record) | `crc32c` over `[lsn][k][v]` doesn't match | Truncate the log at the last valid record. **This is why every record carries a checksum** — the length prefix alone can't distinguish "short record" from "garbage that looks like a length". |
| `fsync` returns an error | check the return value — **always** | **PANIC and stop the process.** Do not retry (see the fsyncgate case study — the retry may succeed while the data is gone). Recovery must come from the log, not from hope. |
| Log grows unboundedly | size threshold | **Checkpoint**: flush the in-memory state to the data file, `fsync` *that*, then it is safe to delete WAL segments older than the checkpoint LSN. Order matters absolutely: fsync data → record checkpoint → delete log. Invert it and a crash between steps loses data that exists nowhere. |
| Recovery replays a record twice | monotonic LSN per record | Replay is idempotent: skip records with `lsn ≤ last_applied`. |
| Log file created but never durable | — | `fsync` the **parent directory** after creating each segment (§1.4). Skipping this is a real and popular bug. |

**Why this over the alternatives?**
- *Write in place, fsync the data file:* random I/O per write; and a power cut mid-update leaves a *torn data structure* with no way to tell what was intended. The log's whole value is that it's append-only and self-describing.
- *Copy-on-write / shadow paging* (LMDB, ZFS): correct and elegant, no WAL needed, but it costs one fsync per commit with no group-commit equivalent, and it fragments. Great for read-heavy embedded stores; worse for write-heavy.
- *Just don't fsync* (`synchronous_commit = off`): legitimate! For a cache, or for data reconstructible from an upstream source. It's a *product* decision — "we may lose the last 200 ms on power loss" — that must be written down, not an accident.

**Cross-references.** The 1 GB/hour shipping design (§1.9) reuses the "detect the tail, track an offset" machinery. Part 2 §2.3 builds exactly this WAL for an agent's tool-call journal, where the "torn record" is a half-written tool result and the "idempotent replay" is not re-charging a customer's credit card.

---

## 1.9 System design ② — Log rotation and shipping for 1 GB/hour, losing nothing

**The problem.** A service writes 1 GB/hour of JSON lines to `/var/log/app/app.log` (≈278 KB/s, ≈2,500 lines/s at 110 B/line). The disk has 20 GB free. Logs must reach a central store (S3/Loki/Elasticsearch). Constraints: (a) never fill the disk; (b) never lose a line, including across restarts of both the app and the shipper; (c) the app must not block on the shipper; (d) at-least-once delivery is acceptable, silent loss is not.

*(Distinct from §1.8: that one is about making a single append durable before acking a client. This one is about bounded storage and cross-process, at-least-once handoff of an unbounded stream — different problem, different failure modes.)*

**Requirements → key decisions.**

**Decision 1: how to rotate.** Two mechanisms, and the difference is an inode.

```
── copytruncate ──────────────────────────  ── create + reopen (correct) ──────────────
1. cp app.log app.log.1                     1. mv app.log app.log.1     (rename: inode
2. truncate app.log to 0                       unchanged, app keeps writing to it)
                                            2. signal the app (SIGHUP / SIGUSR1)
RACE: lines written between steps 1 and 2   3. app close()s old fd, open()s a NEW app.log
are copied nowhere and then erased.            (new inode) and continues
Also: the app's fd still has offset=1GB,    NO RACE. Bounded, atomic, no lost window.
so it writes at offset 1GB into a 0-byte    Cost: the app must implement reopen-on-signal.
file -> a 1GB SPARSE file. Both bugs are
silent.
```

Use **create + reopen**. `copytruncate` exists only for applications that cannot be taught to reopen — treat it as a compatibility wart, and know that when you use it you have accepted a small, unbounded-in-the-worst-case loss window. (This is why `logrotate`'s `copytruncate` documentation carries a warning; primary source: `logrotate(8)`.)

**Decision 2: the deleted-but-open trap.** After rotation, if the shipper (or the app) still holds an fd on `app.log.1` and something `rm`s it, the *name* is gone but the inode's blocks are **not freed** — because §1.1 told us data is freed only when link count *and* open count both reach zero. `df` says the disk is full; `du` says there's nothing there. This is the classic "disk full but no files" incident.

```bash
# The diagnostic. Memorise it.
lsof +L1                         # lists open files with link count < 1 (i.e. deleted)
ls -l /proc/<pid>/fd | grep deleted
# The fix: restart or signal the holder to close the fd. `rm` cannot help you.
```

**Decision 3: at-least-once shipping without blocking the app.** The app writes to a file; a **separate** shipper process tails it. Never pipe your app's stdout directly into a network shipper — if the shipper stalls, the 64 KB pipe buffer fills and your **application blocks on `write()`**. The file is the decoupling buffer, and that's the whole reason to have one.

The shipper's durable state is a tiny **registry**: `{(device, inode): byte_offset}` — keyed on `(dev, ino)` from §1.1, **not the filename**, so a rename is transparently followed and a *new* file at the same name is correctly treated as new.

```
tail loop:
  1. read from offset -> lines
  2. ship batch (HTTP POST / S3 put)  ──► wait for 2xx
  3. ONLY THEN: registry[(dev,ino)] = new_offset ;  fsync the registry
     (order matters: ship-then-record = at-least-once (duplicates possible).
      record-then-ship = at-most-once (silent loss). Choose deliberately.)
  4. on partial last line (no trailing '\n'): do NOT ship it. Wait — the writer is
     mid-write. Shipping a half line and then its other half yields two corrupt records.
```

**Decision 4: disk bound.** With 20 GB free and 1 GB/hour: rotate at 200 MB (≈12 min/file), keep 40 files ≈ 8 GB ≈ 8 hours of buffer, compress rotated files (`logrotate` `compress` — JSON gzips ~8–12×, so 40 files ≈ 1 GB on disk and you can afford far more retention). Alert when the *newest un-shipped* file is older than ~30 min: that's shipper-lag, and it's the leading indicator of the disk filling.

**Failure modes.**

| Failure | Consequence | Mitigation |
|---|---|---|
| Shipper dies for 6 hours | 6 GB of un-shipped logs | 8-hour on-disk buffer absorbs it; registry offset means it resumes exactly where it stopped |
| Shipper dies **after** shipping, **before** fsyncing the registry | duplicate lines on restart | Accepted (at-least-once). Downstream dedupes on a `(host, file_id, offset)` or event-id key. |
| Disk fills anyway | app's `write()` gets `ENOSPC` | Bound total log dir size; drop *oldest* rotated files and count the drop as a metric. **Never** let logging take down the service — logging is not the product. |
| Two processes append to one file | interleaved/torn lines | Open with `O_APPEND` (each `write` atomically seeks-to-end). For a *pipe* (stdout), writes ≤ `PIPE_BUF` = 4096 B are atomic; larger ones can interleave. **Docker's json-file driver splits log lines at 16 KB**, so >16 KB lines arrive as multiple records — a documented, frequently-rediscovered behaviour. Keep lines short, or reassemble downstream. |
| Log line contains a newline (a stack trace, a user-supplied string) | one event becomes many | Serialize as JSON (which escapes `\n` as `\\n`) — never as raw multi-line text. This is the strongest single argument for structured logging. |
| Clock skew makes ordering wrong | misordered timeline | Order by a monotonic sequence number per host, not wall clock. |

**Why this over the alternatives?** Shipping over the network directly from the app (no file) couples your request latency to a log backend and loses everything buffered in memory on a crash. A shared volume that the collector reads is the same design as above with extra indirection. Kernel-level `journald` + `journalctl --cursor` is a legitimate substitution — it's the same registry/offset idea with the state managed for you. What you must *not* do is invent a scheme where the ack (offset commit) happens before the durable transfer; that's §1.8's rule applied to a second domain, and it's the through-line of this entire day.

---

## 1.10 Case studies

### ① Postgres fsyncgate (2018) — the postmortem

**What happened.** In March 2018, Craig Ringer posted to `pgsql-hackers` a message titled *"PostgreSQL's handling of fsync() errors is unsafe and risks data loss at least on XFS."* The finding, in three steps:

1. Postgres separates concerns: backends `write()` dirty pages, and a dedicated **checkpointer** process later calls `fsync()` on those files.
2. On Linux, when writeback failed (a transient disk error, a thin-provisioned volume hitting `ENOSPC`, a USB disk yanked), the kernel **reported the error to whichever fd called `fsync` first, and then cleared the error state**, marking the failed pages *clean*. The dirty data was gone from the page cache; there was nothing left to retry.
3. Consequently, the checkpointer's `fsync()` could return **success** for data that had been permanently lost — and, worse, Postgres's behaviour at the time was to *retry* a failed `fsync`, whose second call returned 0 (success, because the error was already consumed and cleared) — so the checkpoint was recorded as complete and the WAL that could have recovered the data was recycled.

Net effect: databases worldwide were quietly less durable than everyone believed, and the failure was invisible — a successful checkpoint, a silently corrupt data file.

**Engineering lessons — four, all directly actionable.**

1. **`fsync` errors are not retryable.** Once the kernel has reported (and dropped) a writeback error, the data is unrecoverable through that fd. Postgres's fix was to **PANIC** — crash the server and force recovery from the WAL — rather than retry. That is now the default (`data_sync_retry = off`). If your code does `try: os.fsync(fd) except OSError: retry`, you have written the fsyncgate bug.
2. **The process that `write()`s should be the one that `fsync()`s**, or you must have a way to propagate errors between them. Linux's fix (Jeff Layton's `errseq_t`, Linux 4.13) made the error reportable **once per fd** rather than once globally, so a second, independently-opened fd no longer misses it — but it still doesn't make lost data reappear.
3. **Checking the return value of `fsync` is not optional.** In a scripting language, that means not swallowing the exception. `close()` can also return an error (deferred write failure) — checking `close()`'s return value is a real requirement, not pedantry.
4. **Reproduce your durability assumptions on your actual storage stack.** The behaviour differed across ext4/XFS/btrfs, and network/virtual block devices behave differently again. Your cloud provider's block store is not your laptop's NVMe.

**Primary sources:** the `pgsql-hackers` thread (March–April 2018, archived at `postgresql.org/message-id/`); Jonathan Corbet, *"PostgreSQL's fsync() surprise"*, LWN.net, 2018-04-18 (`lwn.net/Articles/752063/`); the PostgreSQL commit introducing PANIC-on-fsync-failure and the `data_sync_retry` GUC; Linux commit series introducing `errseq_t` in `fs/`. **Verify current Postgres behaviour against the release you run** — the details have moved.

### ② Redis persistence — durability as an explicit, documented dial

**What it is.** Redis is an in-memory store, so persistence is optional and *configurable*, and it exposes exactly the trade-off from §1.4 as user-facing knobs.

- **RDB (snapshot):** `fork()` the process and write a point-in-time dump of the whole dataset. Copy-on-write means the child sees a frozen snapshot while the parent keeps serving. Cheap steady-state (no per-write cost) but you lose everything since the last snapshot — minutes, potentially. Also: `fork()` on a 20 GB dataset can transiently double memory if the parent's write rate is high (COW page faults), which is a real operational surprise on memory-tight boxes.
- **AOF (append-only file):** log every mutating command, i.e. a WAL (§1.8). Durability is governed by `appendfsync`:

| `appendfsync` | Data at risk on power loss | Throughput impact |
|---|---|---|
| `always` | ~0 (the last in-flight command) | large — one fsync per write |
| `everysec` (**default**) | **up to 1 second of writes** | small — one fsync/sec, done in a background thread |
| `no` | up to ~30 s (kernel writeback) | none |

**Engineering lessons.**
1. **Durability is a product decision with a number attached.** Redis's default is documented as "up to one second of writes" — a *stated* window, not an accident. Every system you build should be able to state its number. If you can't, you don't know your durability.
2. **`everysec` is the right default for most things**, and it's the same insight as group commit: amortise the fsync over many writes and bound the loss window by *time* instead of by *count*.
3. **Snapshot + log is the general pattern.** RDB is a checkpoint, AOF is the WAL, and AOF rewrite is log compaction — the exact same triad as §1.8, and as Postgres's base backup + WAL, and as an LSM-tree's SSTables + memtable log. Recognising this shape once means recognising it everywhere.
4. **Fast is not durable, and the docs say so.** Redis's own documentation notes that with AOF `everysec` the write is acknowledged to the client *before* it is fsynced. If your application treats a Redis `OK` as a durability guarantee, you have misread a document that says otherwise in plain English.

**Primary sources:** `redis.io/docs/latest/operate/oss_and_stack/management/persistence/`; `redis.conf` annotations for `appendfsync` and `save`. **Verify against your Redis version** — Redis 7 introduced multi-part AOF and changed the file layout.

---

## 1.11 In production (Part 1)

**Best practices, beginner → senior.**

| Level | Habit |
|---|---|
| Beginner | Always `close()` — use `with`/`defer`/`try-with-resources`. Never assume `flush()` means durable. |
| Intermediate | `flush()` **then** `fsync()`, in that order. `fsync` the parent directory after creating a file. Check `fsync` and `close` return values. Set `LimitNOFILE` explicitly. |
| Senior | Batch fsyncs (group commit) instead of choosing between "slow" and "lossy". Treat `fsync` failure as fatal, never retryable. Know your storage's real fsync latency, measured, not assumed. Never do blocking I/O on an event-loop thread (Part 3). Key file identity on `(dev, ino)`, not paths. |

**Monitoring and observability — the five things worth a dashboard.**

1. `process_open_fds / process_max_fds` per process. Alert > 0.7. Distinguish *plateau* (capacity) from *ramp* (leak).
2. `Dirty` and `Writeback` from `/proc/meminfo`. Gigabytes of `Dirty` = a large unacknowledged loss window.
3. fsync latency, as a histogram (p50/p99). A p99 spike here is the root cause of a surprising number of "the app got slow" incidents, and it's usually a noisy neighbour on shared storage.
4. Disk queue depth and `%util` (`iostat -x 1`), plus `await`. When fsync p99 goes to 200 ms, this tells you whether it's the device or the filesystem.
5. `lsof +L1` count (deleted-but-open bytes). A slow leak here fills disks weeks after the actual bug.

**Failure modes and recovery.**

| Symptom | Likely cause | First move |
|---|---|---|
| `EMFILE: Too many open files` | leak or under-set limit | `ls -l /proc/<pid>/fd \| awk '{print $NF}' \| sort \| uniq -c \| sort -rn \| head` |
| 100% CPU, no throughput | busy-wait on `EAGAIN`, or `accept` spinning on `EMFILE` | `strace -c -p <pid>` — a huge count of one failing syscall is the tell |
| Disk full, `du` shows nothing | deleted-but-open files | `lsof +L1`; restart or signal the holder |
| p99 latency spikes with flat CPU | fsync stalls / writeback pressure | `iostat -x 1`; consider group commit or `fdatasync` |
| Data lost after power cut, "but we called flush" | flush ≠ fsync | add fsync; measure the new throughput; decide the loss window explicitly |
| Half-written log lines | line > `PIPE_BUF`/16 KB, or missing `O_APPEND` | shorten lines, use JSON, use `O_APPEND` |

**Scaling behaviour.** fds scale with *concurrency*, not with request rate — 10,000 slow clients cost more descriptors than 100,000 fast ones. fsync cost scales with the *number of durability points*, not bytes: one fsync of 1 MB ≈ one fsync of 100 B. That asymmetry is the entire economic argument for batching, and it's why "write bigger, sync less often" is the universal storage optimisation.

**Cost.** Durability is a real line item. On network block storage (EBS, Persistent Disk), each fsync is a network round-trip *plus* provisioned IOPS you pay for. Going from `fsync`-per-record to `fsync`-per-100-records can cut provisioned IOPS by ~99% — sometimes a larger saving than any compute optimisation you'll find. Conversely, `Dirty` pages are free until the moment they aren't.

---

## 1.12 Failure modes and common misconceptions (Part 1)

| Misconception | Reality |
|---|---|
| "`flush()` writes to disk." | It writes to the **kernel**. The disk is a second, separate step (`fsync`). |
| "`kill -9` proves I need fsync." | It proves you need `flush`. Page cache survives process death; only a *machine* crash tests fsync. |
| "If `fsync` fails, retry it." | The data is already gone and the retry may return success. **PANIC.** (fsyncgate.) |
| "fds are only for files." | Sockets, pipes, terminals, timers, signals, epoll instances, inotify watches — all fds, all counted against your limit. |
| "`ulimit -n` in my shell fixed the service." | Not for anything systemd starts. Use `LimitNOFILE=`; verify with `cat /proc/<pid>/limits`. |
| "`read()` returning less than I asked for means EOF." | On a **socket/pipe** it means "that's all so far." Loop until you have a complete message, or until `read` returns 0. |
| "`EAGAIN` is an error to log." | It's normal flow control: "nothing right now." Handle it; don't alert on it. |
| "Non-blocking I/O makes things fast." | Non-blocking alone makes things *busy*. The speed comes from readiness notification (`epoll`) letting one thread sleep correctly. |
| "epoll makes disk reads async." | It does not — regular files are always "ready", so epoll can't overlap disk I/O. That's what thread pools and `io_uring` are for. |
| "The drive honoured my fsync." | Test it. Consumer drives and mis-configured virtual disks have acknowledged flushes they didn't perform. |
| "Bigger fd limit fixes the leak." | It delays the outage and enlarges the blast radius. Diagnose the ramp. |

## 1.13 Interview & practice questions (Part 1)

1. What are the three levels of the fd → data lookup, and which of them does `dup()` copy? *(fd table → open file description → inode; `dup` copies the fd-table pointer, sharing the description and therefore the offset.)*
2. `f.write(b"x"); os.kill(pid, 9)` — is `x` in the file? What if you'd called `f.flush()` first? What if the *machine* lost power? *(No / yes / probably not — you need `fsync` for the third.)*
3. Why is `select` limited to ~1024 fds, and why isn't `epoll`? *(`FD_SETSIZE` bitmask vs kernel-side interest set + ready list.)*
4. Your server sits at 100% CPU serving zero requests. Name two descriptor-related causes. *(Busy-wait on `EAGAIN`; `accept()` spinning on `EMFILE` while epoll keeps reporting the listener readable.)*
5. Why must you `fsync` a *directory*? *(To persist the directory entry — the file's name — not just its contents.)*
6. `df` says 100% full; `du -sh /` says 4 GB used. What happened and what's the command? *(Deleted-but-open files; `lsof +L1`.)*
7. You need 50,000 durable appends/sec on a device that does 5,000 serial fsyncs/sec. How? *(Group commit: one fsync per batch of concurrent writers.)*
8. Why does `copytruncate` lose log lines, and what's the second bug it causes? *(Race between copy and truncate; the writer's stale offset creates a sparse file.)*
9. What's the difference between readiness-based and completion-based I/O, and name one API of each. *(epoll/kqueue vs IOCP/io_uring.)*
10. Your `fsync` returns `EIO`. What do you do, and why is "retry" wrong? *(Crash and recover from the log; the error was consumed and the pages were dropped — fsyncgate.)*
