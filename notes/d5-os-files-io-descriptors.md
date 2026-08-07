# Day 5 — The OS IV: Files, I/O, and the Descriptors That Run the World

> **Framing.** Day 2 gave you the syscall — the only door from your code into the kernel — and told you that `print("hi")` bottoms out in `write(1, "hi\n", 3)`. It left two things unexplained, and they are the two things every backend and every agent server is actually made of: **what is that `1`**, and **where do those three bytes go?** Today we answer both. The `1` is a **file descriptor** — a small integer that indexes a per-process table of everything your program has open: files, sockets, pipes, terminals, timers, even other processes. The bytes go into a **stack of buffers** — your language's buffer, then the kernel's page cache — and *not*, by default, onto the disk; making them actually hit the platter is a separate, explicit, and shockingly expensive operation called **`fsync`**. Between those two facts lives everything: why a database is slow, why `kill -9` loses your last log lines, why one thread can serve ten thousand sockets (`epoll`), why your server dies with "Too many open files", and why a 2018 kernel behaviour quietly made every PostgreSQL database on Earth less durable than its operators believed.
>
> **Who it's for.** Someone who has never heard "file descriptor," thinks `f.write(...)` puts bytes on the disk, has never seen `EMFILE`, and has no idea what `epoll` is or why `asyncio`, Nginx, Redis, and Node.js all rest on it. We build from zero. Depends on Day 1 (the latency pyramid: RAM ≈ 100 ns, NVMe ≈ 100 µs, spinning-disk seek ≈ 10 ms — a 100,000× spread that makes "did it hit the disk?" the most expensive question in computing), Day 2 (a syscall costs a mode switch; a process owns a descriptor table; `SIGKILL` gives you no cleanup), Day 3 (threads and blocking), and Day 4 (the page cache *is* memory — dirty pages you haven't flushed are RAM that the OOM killer can't reclaim and a crash will drop). Those are cross-referenced, not re-explained.
>
> **The ONE idea that unites the backend and agentic layers:** *a file descriptor is a lease on a kernel object, and I/O is the art of deciding when to wait for it, when to buffer it, and when to pay for durability.* A web server, a database, and an LLM agent are the *same shape of program*: mostly idle, holding a fistful of long-lived descriptors, waiting for bytes that arrive at network speed while the CPU has nothing to do. Get the waiting model wrong and you burn one thread per connection until the machine dies; get the buffering wrong and your users see a 30-second pause followed by a wall of text; get the durability wrong and you acknowledge work you have already lost. This is the day "the file is written" stops being a belief and becomes a claim you can prove or disprove with a `kill -9` and a line count.

**A note on platform.** File descriptors, `/proc/<pid>/fd`, `epoll`, `O_NONBLOCK`, `ulimit -n`, and the exact fsync semantics discussed here are **Linux** concepts. Windows has analogous machinery (HANDLEs, I/O completion ports, `FlushFileBuffers`) with different names and different failure modes; macOS has descriptors and `kqueue` instead of `epoll`, and its `fsync(2)` famously does *not* flush the drive's own write cache (you need `fcntl(F_FULLFSYNC)` — a real, documented divergence). Every runnable example below is written for **Linux**: run them in WSL2 (`wsl --install`, then `wsl`), a container (`docker run -it --rm -v "$PWD":/w -w /w python:3.12 bash`), or any Linux VM. Where Windows, macOS, or CPython specifics diverge, I say so explicitly.

**What is [CORE] today and what is not.** File descriptors, the buffering stack, `fsync`, blocking vs non-blocking, and `epoll` are all [CORE] — we open every box down to the kernel data structure. `io_uring`, `O_DIRECT`, and filesystem journal internals are [AWARE]: named, sketched, and deliberately left closed. `asyncio`'s event loop is built on §1.6 but its API surface is **Day 21**; the write-ahead log you design in §1.7 is the thing you *build* on **Day 37**; TCP itself is the networking day. Those stops are on purpose.

**Reading order.** Part 1 builds the machine: descriptors → "everything is a file" → the buffer stack → `fsync` → blocking vs non-blocking → `epoll`, then two system designs, the case studies, and production practice. Part 2 puts an LLM agent on top of it — an agent server is an I/O-bound program holding one long-lived descriptor per user for minutes at a time — treating the backend as a black box. Part 3 is where a single process's **one descriptor table** and **one page cache** are shared by the HTTP server, the database driver, and the model stream, and where they run out together.

---

# PART 1 — BACKEND

## 1.1 File descriptors — the small integers that name every open thing

**Depth: [CORE]**

### Intuition

Your program wants to read a file. It cannot just hold "the file" — a file lives on a disk, managed by a filesystem, reachable only through the kernel (Day 2: user space cannot touch hardware). So the kernel does what every system does when it must hand out access to something it owns and won't let go of: **it gives you a ticket.**

You call `open("/etc/hostname")`. The kernel finds the file, builds a bunch of internal bookkeeping (where the file is, how far you've read, whether you may write), stores that bookkeeping in *its* memory, and hands you back... the number **3**. That number is a **file descriptor** (FD): an index into a per-process array of open things. From then on, every operation you do is "do X to entry 3 of my table": `read(3, buf, 4096)`, `lseek(3, 0, SEEK_SET)`, `close(3)`.

Three properties fall straight out of that design, and they explain most of what confuses beginners:

1. **FDs are per-process.** Descriptor 3 in your process and descriptor 3 in mine point at completely different things. There is no global namespace. (This is the same isolation principle as Day 4's virtual addresses: a small integer that means something different in each process's private table.)
2. **The kernel always hands you the lowest free number.** Not a random ID — the *smallest unused index*. This is not an implementation detail you can ignore; the entire Unix shell redirection mechanism (`>`, `<`, `|`) is built on being able to predict it (§1.2's worked example).
3. **FDs 0, 1, 2 are already taken** when your program starts: 0 = standard input, 1 = standard output, 2 = standard error. That is *convention*, enforced by whoever launched you (the shell), not by the kernel. It is why the traced syscall on Day 2 was `write(1, ...)` and not `write(7, ...)`.

**Why FDs exist / what came before.** The alternative is handing user space a real kernel pointer, which is a catastrophe: a program could forge one, corrupt it, or pass one to another process that has no right to it. An integer index into a table the *kernel* owns is a **capability**: unforgeable in any useful sense (index 9,999 in a table with 20 entries is simply an error, `EBADF`), revocable (close it and the entry is gone), and cheap to validate (one bounds check). Every modern OS reinvents this — Windows HANDLEs, macOS Mach ports — because there is no better answer.

### Analogy — the coat-check ticket

You hand your coat to the attendant at a theatre and get ticket **#3**. You do not get the coat, a map to the coat rack, or the ability to walk into the back room. To get your coat back you present #3. If you lose the ticket you cannot get the coat. If you hand #3 to a friend, they can collect your coat — the ticket *is* the authority. And the attendant reuses numbers: when #3 is redeemed, the next customer gets #3.

**Where the analogy breaks — and it breaks in three places that matter:**

1. **Two tickets can name the same coat, and share a bookmark.** `dup(3)` gives you descriptor 4 pointing at *the same* underlying kernel object, including the same read/write **offset**. Read 100 bytes via FD 3 and FD 4 now also starts at byte 100. Coat checks have no shared position. (Two separate `open()` calls on the same path give you two *independent* offsets. Same file, different bookmarks. This distinction — "file descriptor" vs "open file description" — is the single most misunderstood thing in Unix I/O, and it's opened up under "Under the hood" below.)
2. **The coat can be gone while the ticket still works.** `unlink()` a file that you still have open and the directory entry vanishes — `ls` shows nothing — but your FD keeps working and the disk space is not reclaimed until you `close()`. This is how `tmpfile()` works and why "I deleted the log file but the disk is still full" is a weekly ops question (§1.10).
3. **The cloakroom has a hard, low capacity, and it's per-customer.** You do not get unlimited tickets. Every process has an `RLIMIT_NOFILE` — often 1024 by default — and hitting it gives you `EMFILE: Too many open files`, an error that takes down more production services than almost anything else. A real cloakroom quietly finds another rack.

### Worked example — watching descriptor numbers get handed out and reused

Trace what the kernel does, step by step, for a tiny program. The numbers below are exactly what you'll see in the runnable example that follows.

```
Program starts (shell has set up 0,1,2)
  fd 0 -> /dev/pts/3   (stdin,  the terminal)
  fd 1 -> /dev/pts/3   (stdout, the terminal)
  fd 2 -> /dev/pts/3   (stderr, the terminal)

open("/etc/hostname")        -> 3   (lowest free)
socket(AF_INET, SOCK_STREAM) -> 4   (lowest free)
pipe()                       -> 5, 6 (two at once: read end, write end)

close(4)                            (socket gone; 4 is now free)
open("/etc/hosts")           -> 4   <-- REUSED. Not 7.
```

That last line is the whole point. The kernel is not issuing monotonically increasing IDs; it is filling the first hole in an array. Every shell on Earth exploits this: to make a command write to a file, the shell closes FD 1 and then opens the file — which, being the lowest free number, *becomes* FD 1 — and then `exec`s the program, which writes to FD 1 exactly as it always does, entirely unaware (Day 2 §fork/exec). The program is not cooperating. It cannot even tell.

### Runnable example — the descriptor table, made visible, and then exhausted

```python
# fds.py — Linux. stdlib only.
# Run:  python3 fds.py
import os, socket, resource

def fd_table():
    """Read this process's descriptor table straight out of /proc."""
    entries = []
    for name in sorted(os.listdir("/proc/self/fd"), key=int):
        try:
            target = os.readlink(f"/proc/self/fd/{name}")
        except OSError:
            target = "<gone>"
        entries.append((int(name), target))
    return entries

print("=== 1) what a fresh process already holds ===")
for n, t in fd_table():
    print(f"  fd {n} -> {t}")

print("\n=== 2) the kernel hands out the LOWEST FREE number ===")
f    = open("/etc/hostname")                      # -> some fd
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
r, w = os.pipe()                                  # two at once
print(f"  open('/etc/hostname') -> fd {f.fileno()}")
print(f"  socket()              -> fd {sock.fileno()}")
print(f"  pipe()                -> fds {r} (read), {w} (write)")

freed = sock.fileno()
sock.close()
f2 = open("/etc/hosts")
print(f"  closed fd {freed}, then open('/etc/hosts') -> fd {f2.fileno()}  <- REUSED")

print("\n=== 3) two FDs, ONE shared offset (dup) vs two independent ones (open) ===")
a = os.open("/etc/hostname", os.O_RDONLY)
b = os.dup(a)                                     # same open file description
c = os.open("/etc/hostname", os.O_RDONLY)         # a NEW open file description
os.read(a, 4)
print(f"  after read(a,4):  offset via a={os.lseek(a,0,os.SEEK_CUR)}  "
      f"via b={os.lseek(b,0,os.SEEK_CUR)}  via c={os.lseek(c,0,os.SEEK_CUR)}")
for fd in (a, b, c):
    os.close(fd)

print("\n=== 4) EMFILE: run out of descriptors on purpose ===")
soft, hard = resource.getrlimit(resource.RLIMIT_NOFILE)
resource.setrlimit(resource.RLIMIT_NOFILE, (64, hard))   # lower OUR soft limit
print(f"  soft limit was {soft}, now 64 (hard limit {hard})")
held = []
try:
    while True:
        held.append(open("/etc/hostname"))
except OSError as e:
    print(f"  opened {len(held)} more files, then: [{e.errno}] {e.strerror}")
finally:
    for fh in held:
        fh.close()
    resource.setrlimit(resource.RLIMIT_NOFILE, (soft, hard))  # restore
print(f"  after closing, limit restored to {soft}; leak fixed by closing, not by praying")

f.close(); f2.close(); os.close(r); os.close(w)

# --- actual output on Linux (fd numbers stable; /dev/pts path varies) ---
# === 1) what a fresh process already holds ===
#   fd 0 -> /dev/pts/3
#   fd 1 -> /dev/pts/3
#   fd 2 -> /dev/pts/3
#   fd 3 -> /proc/12841/fd            <- the listdir() itself!
#
# === 2) the kernel hands out the LOWEST FREE number ===
#   open('/etc/hostname') -> fd 3
#   socket()              -> fd 4
#   pipe()                -> fds 5 (read), 6 (write)
#   closed fd 4, then open('/etc/hosts') -> fd 4  <- REUSED
#
# === 3) two FDs, ONE shared offset (dup) vs two independent ones (open) ===
#   after read(a,4):  offset via a=4  via b=4  via c=0
#
# === 4) EMFILE: run out of descriptors on purpose ===
#   soft limit was 1024, now 64 (hard limit 1048576)
#   opened 56 more files, then: [24] Too many open files
#   after closing, limit restored to 1024; leak fixed by closing, not by praying
```

**Why this works, line by line.**

- `/proc/self/fd` is the descriptor table exposed as a directory of symlinks — the kernel's own view of your process, readable with `ls`. `os.readlink` on each entry tells you what the descriptor *points at*: a path for files, `socket:[1783420]` for sockets (a socket has no path — see §1.2), `pipe:[…]` for pipes, `anon_inode:[eventpoll]` for an epoll instance.
- **Block 1's `fd 3 -> /proc/12841/fd` is not a bug, it's the observer effect**: `os.listdir` must itself open the directory to enumerate it, so the act of looking at the table adds an entry to the table. Day 2 showed the same artifact; it is the clearest possible proof that a directory listing is just another open thing.
- **Block 2 is the lowest-free-number rule, demonstrated.** `pipe()` returns *two* descriptors in one syscall because a pipe is a single kernel buffer with two ends. Closing FD 4 and immediately reopening returns **4**, not 7 — this is the mechanism behind `>` and `|`, and it's also why FD reuse is a real class of bug: close a descriptor while another thread is still using the number, and that thread's next `write()` lands in whatever got opened in the meantime. (Day 3's race conditions, applied to descriptors.)
- **Block 3 is the distinction almost nobody knows.** `dup(a)` creates a second *descriptor* pointing at the same **open file description** — one shared offset, so reading through `a` moves `b`'s position too (both report 4). A fresh `open()` of the same path creates a *new* description with its own offset (reports 0). Same file, same inode, different bookmark. This is why `cmd1 > out; cmd2 > out` can interleave badly while `cmd1 > out 2>&1` (which uses `dup2`) does not: `2>&1` makes stderr share stdout's *description*, so both streams advance one shared offset and never overwrite each other.
- **Block 4 is `EMFILE`, on demand.** `RLIMIT_NOFILE` has a *soft* limit (what you're actually held to, raisable by you up to the hard limit) and a *hard* limit (raisable only by root). We lower the soft limit to 64, then open files until the kernel refuses with errno 24. Note we opened **56**, not 64 — because 0,1,2 and a few others were already in use, and Python's own internals hold some. That's the real lesson: **your budget is not "how many files do I open", it is "how many descriptors does the entire process hold at peak"** — sockets, pipes, epoll instances, database connections, and log files all draw on the same 1024. `ENFILE` (a different error) is the *system-wide* table filling up, visible in `/proc/sys/fs/file-max`.

**Under the hood.** In Linux, each `task_struct` (Day 2's process object) points at a `files_struct` containing an `fdtable` — essentially an array of `struct file *`. The FD is the array index. Each `struct file` is an **open file description**: it holds the current offset (`f_pos`), the access mode and status flags (`O_RDONLY`, `O_APPEND`, `O_NONBLOCK`), a reference count, and a pointer to the `struct inode` that identifies the actual on-disk (or in-memory, or network) object, plus a `file_operations` vtable (§1.2). So there are **three** levels, and confusing them causes real bugs:

```
process A                    kernel                         object
┌──────────────┐
│ fd table     │
│  0 ──────────┼──┐
│  1 ──────────┼──┤        ┌──────────────────┐
│  2 ──────────┼──┴───────►│ open file descr. │──┐
│  3 ──────────┼──────────►│ offset=4096      │  │      ┌─────────┐
│  4 ──────────┼──┐        │ flags=O_WRONLY   │  ├─────►│ inode   │  (the file itself)
└──────────────┘  │        └──────────────────┘  │      │ /var/log│
                  │        ┌──────────────────┐  │      └─────────┘
process B         └───────►│ open file descr. │──┘
┌──────────────┐           │ offset=0         │
│  3 ──────────┼──────────►│ flags=O_RDONLY   │
└──────────────┘           └──────────────────┘
    fd (per-process int)      description (offset+flags)   inode (the object)
```

`dup`/`dup2`/`fork` duplicate the *arrow* (new FD, same description → shared offset). A fresh `open` creates a new *description* (independent offset, same inode). `close()` drops the FD; the description is freed only when its refcount hits zero — which is why a forked child holding your database socket keeps that connection alive after you close it (Day 2's fd-inheritance leak, now with a mechanism). Primary sources: `open(2)`, `dup(2)`, `close(2)`, `proc(5)`, `getrlimit(2)`; and the "Open file descriptions" section of `open(2)` for the exact three-level model.

---

## 1.2 "Everything is a file" — one interface over files, pipes, sockets, and devices

**Depth: [CORE]**

### Intuition

Unix's most famous slogan is *"everything is a file."* It is slightly wrong and enormously important. The accurate version is: **everything you can do I/O on is reachable through a file descriptor, and the same handful of syscalls work on all of them.**

`read(fd, buf, n)` and `write(fd, buf, n)` do not care what is on the other end. Point the descriptor at a regular file and you're doing disk I/O. Point it at a socket and you're doing network I/O. Point it at a pipe and you're talking to another process. Point it at `/dev/urandom` and you're pulling randomness out of the kernel's entropy pool. Point it at a terminal and you're printing to a screen. **Same two functions.**

**Why this matters and what came before.** In systems without this uniformity (classic mainframe I/O, and to a large extent Windows before its POSIX layers), reading from a tape, a printer, a file, and a network connection required four unrelated APIs. Every program that wanted to work with more than one had to know about all of them. Unix's insight was to put a single **narrow interface** in front of everything — `open`/`read`/`write`/`close` — and let the kernel dispatch to the right implementation behind it. The payoff is composition: `grep foo` reads FD 0 and has no idea whether the shell wired that to a file, a pipe from another program, a network socket, or your keyboard. **Every Unix pipeline you have ever typed works because of this one decision.**

That is also why the slogan is misleading: a socket has no path, you cannot `open("/some/path")` to get one, and `lseek` on it fails with `ESPIPE`. Sockets are created by `socket()`, pipes by `pipe()`, epoll instances by `epoll_create1()`. The unifying thing is not "file" but **descriptor**. Say it as: *"everything is a file descriptor, but not everything is a file."*

### Analogy — one wall socket, many appliances

Your house's electrical outlets present one interface: two slots and a fixed voltage. A lamp, a laptop charger, a vacuum, and a fridge all plug into it. The outlet has no idea what's on the other end; the appliance implements whatever it means to receive power. You can rewire what's plugged in without changing the wall.

**Where the analogy breaks:** power flows one way and never blocks. Descriptors are bidirectional and — crucially — **may make you wait indefinitely**. Reading from a file returns almost immediately (the data exists); reading from a socket may hang for minutes because the bytes have not been sent yet; reading from a pipe blocks until the writer writes. The *interface* is uniform, the *timing behaviour is wildly not*, and that gap is exactly what §1.5 and §1.6 exist to manage. A second break: the wall socket can't tell you it's about to have power. A descriptor can (`epoll`), and that ability is the foundation of every high-performance server.

### Worked example — the same two syscalls across four kinds of object

Trace what one `write` of 5 bytes means for four different descriptors:

| FD points at | `write(fd, "hello", 5)` really does | Blocks when | Visible in `/proc/self/fd` as |
|---|---|---|---|
| **Regular file** (`/tmp/a.txt`) | Copies 5 bytes into the **page cache** (RAM). Disk untouched (§1.3) | Almost never (page cache full + writeback throttling) | `/tmp/a.txt` |
| **Pipe** (`pipe()`) | Copies 5 bytes into a 64 KB kernel ring buffer | Buffer full *and* nobody reading | `pipe:[1783412]` |
| **TCP socket** | Copies 5 bytes into the socket **send buffer**; the kernel transmits later | Send buffer full (peer slow / window closed) | `socket:[1783420]` |
| **Terminal** (`/dev/pts/3`) | Hands 5 bytes to the tty line discipline → terminal emulator | Rarely (flow control, `Ctrl-S`) | `/dev/pts/3` |

Notice the pattern: **in three of four cases `write()` returning success means "the kernel accepted your bytes into a buffer," not "the bytes arrived anywhere."** That single observation is the seed of §1.3 and §1.4 (for files) and of every "but I sent it!" networking bug (for sockets).

### Runnable example — one function, four different kinds of descriptor

```python
# everything_is_a_fd.py — Linux. stdlib only.
# Run:  python3 everything_is_a_fd.py
import os, socket, stat, threading

def describe(fd, label):
    """Same three calls, whatever the fd points at."""
    st = os.fstat(fd)                     # works on ANY descriptor
    kind = ("regular file" if stat.S_ISREG(st.st_mode) else
            "directory"    if stat.S_ISDIR(st.st_mode) else
            "FIFO/pipe"    if stat.S_ISFIFO(st.st_mode) else
            "socket"       if stat.S_ISSOCK(st.st_mode) else
            "char device"  if stat.S_ISCHR(st.st_mode) else "other")
    try:
        target = os.readlink(f"/proc/self/fd/{fd}")
    except OSError:
        target = "?"
    seekable = True
    try:
        os.lseek(fd, 0, os.SEEK_CUR)
    except OSError:
        seekable = False
    print(f"  {label:<14} fd={fd:<3} kind={kind:<14} seekable={seekable!s:<5} -> {target}")

# ---------- 1) four different objects, all reached through fds ----------
print("=== four kinds of object, one interface ===")
path = "/tmp/day5_demo.txt"
file_fd = os.open(path, os.O_RDWR | os.O_CREAT | os.O_TRUNC, 0o644)
r_fd, w_fd = os.pipe()
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
dev_fd = os.open("/dev/urandom", os.O_RDONLY)

describe(file_fd, "regular file")
describe(w_fd,    "pipe (write)")
describe(sock.fileno(), "TCP socket")
describe(dev_fd,  "/dev/urandom")

# ---------- 2) the SAME write()/read() pair on file, pipe, and socket ----------
print("\n=== identical os.write / os.read across all three ===")
payload = b"hello"

os.write(file_fd, payload)                       # -> page cache
os.lseek(file_fd, 0, os.SEEK_SET)
print(f"  file : wrote {payload!r}, read back {os.read(file_fd, 5)!r}")

os.write(w_fd, payload)                          # -> kernel pipe buffer
print(f"  pipe : wrote {payload!r}, read back {os.read(r_fd, 5)!r}")

# A real socket pair: same syscalls, bytes traverse the loopback network stack.
srv = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
srv.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
srv.bind(("127.0.0.1", 0)); srv.listen(1)
port = srv.getsockname()[1]
conn_holder = {}
def accept_once():
    conn_holder["c"], _ = srv.accept()
t = threading.Thread(target=accept_once); t.start()
sock.connect(("127.0.0.1", port)); t.join()
server_side = conn_holder["c"]
os.write(sock.fileno(), payload)                 # -> socket send buffer
print(f"  socket: wrote {payload!r}, read back {os.read(server_side.fileno(), 5)!r}")

print(f"  dev   : read 8 random bytes: {os.read(dev_fd, 8).hex()}")

# ---------- 3) where the uniformity STOPS ----------
print("\n=== where 'everything is a file' breaks ===")
for fd, label in ((file_fd, "file"), (r_fd, "pipe"), (sock.fileno(), "socket")):
    try:
        os.lseek(fd, 0, os.SEEK_SET)
        print(f"  lseek on {label:<7}: OK (streams of bytes with a position)")
    except OSError as e:
        print(f"  lseek on {label:<7}: [{e.errno}] {e.strerror}  <- not seekable")

for fd in (file_fd, r_fd, w_fd, dev_fd):
    os.close(fd)
sock.close(); server_side.close(); srv.close(); os.unlink(path)

# --- actual output on Linux (inode numbers vary) ---
# === four kinds of object, one interface ===
#   regular file   fd=3   kind=regular file  seekable=True  -> /tmp/day5_demo.txt
#   pipe (write)   fd=5   kind=FIFO/pipe     seekable=False -> pipe:[2216845]
#   TCP socket     fd=6   kind=socket        seekable=False -> socket:[2216851]
#   /dev/urandom   fd=7   kind=char device   seekable=True  -> /dev/urandom
#
# === identical os.write / os.read across all three ===
#   file : wrote b'hello', read back b'hello'
#   pipe : wrote b'hello', read back b'hello'
#   socket: wrote b'hello', read back b'hello'
#   dev   : read 8 random bytes: 3f9c1ab240e7d518
#
# === where 'everything is a file' breaks ===
#   lseek on file   : OK (streams of bytes with a position)
#   lseek on pipe   : [29] Illegal seek  <- not seekable
#   lseek on socket : [29] Illegal seek  <- not seekable
```

**Why this works, line by line.**

- `os.fstat(fd)` takes a *descriptor*, not a path, and works on all four objects — you can interrogate a socket's metadata with the same call you use on a file. The `S_ISREG`/`S_ISFIFO`/`S_ISSOCK`/`S_ISCHR` macros read the file-type bits out of `st_mode`; this is exactly how `ls -l`'s leading character (`-`, `d`, `p`, `s`, `c`) is computed.
- **Block 2 is the payoff.** `os.write` and `os.read` are thin wrappers over the `write(2)` and `read(2)` syscalls, and the *identical* call moves bytes into a page cache, a pipe ring buffer, and a TCP send queue. Your code cannot tell them apart. That is the entire reason a Unix program can be handed a pipe where it expected a file and just work — and why `python myserver.py | tee log.txt` is a thing that exists.
- **Block 3 is the honest caveat.** `lseek` fails with `ESPIPE` (errno 29, "Illegal seek") on pipes and sockets, because a *stream* has no addressable position — the bytes are gone once read. So the uniform interface is genuinely uniform for `read`/`write`/`close`/`fstat`, and genuinely not for positioning. When you see library code doing `try: f.seek(0) except OSError:`, this is what it's defending against — and it's why "rewind the request body and retry" fails on a live socket but works on a file.
- The `/dev/urandom` case is the "devices are files" half of the slogan: a kernel entropy source with a path in the filesystem, read with `read()` like anything else. `/dev/null`, `/dev/zero`, and `/dev/sda` are the same idea.

**Under the hood.** Every `struct file` carries a pointer to a `struct file_operations` — a table of function pointers (`.read_iter`, `.write_iter`, `.poll`, `.llseek`, `.release`, …). `read(fd, ...)` is, in essence: look up `fd` in the table → get `struct file *f` → call `f->f_op->read_iter(...)`. It is **polymorphism implemented with a vtable**, a decade before C++ made vtables famous. `ext4` supplies one set of operations, the TCP stack another, the pipe implementation a third, a character-device driver a fourth. `llseek` for pipes and sockets is set to a stub that returns `-ESPIPE` — which is literally where the error above comes from. This is also the extension point: every device driver you have ever installed added a `file_operations` table. Primary sources: `open(2)`, `read(2)`, `write(2)`, `lseek(2)`, `stat(2)`, `socket(2)`, `pipe(2)`, and `include/linux/fs.h` in the kernel source for `struct file_operations`.

---

## 1.3 The buffer stack — why your `write()` almost never touches the disk

**Depth: [CORE]**

### Intuition

Here is the belief that costs people the most data: *"I called `write()`, so the bytes are on the disk."*

They are not. Between your string literal and the magnetic or flash medium there are, in the common case, **three** places bytes can sit, and a crash at each level loses a different amount:

```
   your Python code
        │  f.write("line\n")
        ▼
 ┌──────────────────────────┐
 │ 1. USERSPACE buffer      │   ~8 KB, inside your process's memory
 │    (Python's io module,  │   Lost on: process crash, kill -9, exit without flush
 │     C's stdio, Go bufio) │
 └──────────┬───────────────┘
            │  flush()  →  write(2) syscall   ← crosses into the kernel (Day 2)
            ▼
 ┌──────────────────────────┐
 │ 2. KERNEL page cache     │   RAM the kernel owns, pages marked "dirty" (Day 4)
 │    (dirty pages)         │   Survives: process crash, kill -9
 └──────────┬───────────────┘   Lost on: power loss, kernel panic, VM hard-reset
            │  fsync(fd)   OR   kernel writeback (~30s, or on memory pressure)
            ▼
 ┌──────────────────────────┐
 │ 3. DEVICE write cache    │   a few hundred MB of DRAM *on the drive itself*
 │    (on the SSD/HDD)      │   Lost on: power loss (unless the drive has capacitors)
 └──────────┬───────────────┘
            │  FLUSH / FUA command  (issued by fsync on a correctly-configured stack)
            ▼
      ✅ actually durable
```

Each layer exists for one reason: **a syscall and a disk write are both far more expensive than a memcpy** (Day 1's pyramid: memcpy ≈ nanoseconds, syscall ≈ hundreds of nanoseconds, NVMe write ≈ 100 µs). Buffering trades a *window of possible data loss* for **orders of magnitude** of throughput. Writing a million 20-byte log lines one syscall at a time is a million mode switches; buffering them into 8 KB chunks is 2,500. That is not a micro-optimization, it's a 400× reduction in kernel crossings.

**What came before.** Early Unix programs did unbuffered `write()` per line and were correspondingly slow; `stdio` (Dennis Ritchie's buffered I/O library, 1970s) introduced layer 1 and is the reason `printf` is fast. The page cache (layer 2) arrived as the unification of the buffer cache and virtual memory — it is the same page cache from Day 4, which is why "free memory" on a Linux box is mostly cached file data and why `free -h` shows a huge `buff/cache` column.

**The subtlety that bites everyone:** layer 1's flushing policy depends on *what the descriptor points at*. When stdout is a **terminal**, Python and C stdio use **line buffering** — flush on every `\n`, so you see output immediately. When stdout is a **pipe or file**, they switch to **block buffering** (~4–8 KB), so output appears in bursts or not at all until exit. This is why `python app.py` prints logs live but `python app.py | tee log.txt` appears frozen, and why `docker logs` on a Python container often shows nothing until it crashes. The fix is `PYTHONUNBUFFERED=1` / `python -u` / `flush=True`, and it is the single most common Docker-logging bug in existence.

### Analogy — the outbox, the mailroom, and the truck

You write a letter and drop it in your desk's **outbox** (layer 1: userspace buffer). Once a day someone collects everything and takes it to the building **mailroom** (layer 2: page cache). The mailroom holds mail until a **truck** comes (writeback), and the truck itself has a holding bay before things are actually delivered (layer 3: drive cache).

Now the failure modes map perfectly. Your desk burns down → letters in your outbox are gone, mailroom mail is fine (**process crash / `kill -9`**). The whole building loses power → the mailroom's mail is gone too (**power loss / kernel panic**). And "I dropped it in my outbox" is a very different claim from "it has been delivered" — which is precisely the claim `fsync` lets you make.

**Where the analogy breaks:** you can walk to the mailroom and demand your specific letter be driven out *right now* — that's `fsync`, and unlike a real mailroom it will make you stand there and wait (it blocks) for as long as the drive takes, which can be milliseconds — an eternity in CPU terms. Second break: real mailrooms don't silently lose a letter and then tell the *next* person who asks that everything is fine. Kernels used to (§1.9's fsyncgate). Third break: a mailroom holds *your* letters; the page cache is *shared* — the dirty pages you created are competing for the same RAM as every other process's file data, so the kernel may start the truck early under memory pressure (Day 4).

### Worked example — the same bytes at three different levels of "written"

Trace one program writing 100 bytes and observe each layer:

```
t=0   f = open("/tmp/x", "w")           userspace buffer: empty   file size on disk: 0
t=1   f.write("A" * 100)                userspace buffer: 100 B    file size: 0     ← kernel has seen NOTHING
t=2   os.path.getsize("/tmp/x")  -> 0                                                ← proof: still in YOUR memory
t=3   f.flush()                         userspace buffer: empty    file size: 100   ← write(2) happened
t=4   os.path.getsize("/tmp/x")  -> 100                                              ← kernel now has it (page cache)
      /proc/meminfo Dirty: +4 kB                                                     ← one dirty page appeared
t=5   kill -9 here  →  data SURVIVES (kernel holds it; writeback will happen)
      power cut here →  data is LOST (dirty page never reached the platter)
t=6   os.fsync(f.fileno())              blocks ~0.1–10 ms; dirty page → device → FLUSH
t=7   power cut here →  data SURVIVES
```

The step from t=4 to t=7 is the entire content of §1.4. Notice `getsize` returning 0 at t=2: `os.path.getsize` asks the *kernel*, and the kernel genuinely does not have the bytes yet. That is not a caching artifact or a stale value — it is the literal truth.

### Runnable example — measuring all three layers, and the pipe-vs-tty buffering trap

```python
# buffering.py — Linux. stdlib only.
# Run:  python3 buffering.py
# Then: python3 buffering.py | cat        <-- watch part 3 change
import os, sys, time

PATH = "/tmp/day5_buffering.txt"

def dirty_kb():
    """How many KB of page cache are dirty (written but not yet on disk)."""
    with open("/proc/meminfo") as m:
        for line in m:
            if line.startswith("Dirty:"):
                return int(line.split()[1])
    return -1

# ---------- 1) userspace buffer: the kernel hasn't seen a thing ----------
print("=== layer 1: userspace buffer (your process's memory) ===")
f = open(PATH, "w")                      # text mode, ~8 KB buffer
f.write("A" * 100)
print(f"  after f.write(100 bytes)      -> kernel-visible size: {os.path.getsize(PATH)} bytes")
f.flush()                                # <- this is the write(2) syscall
print(f"  after f.flush()               -> kernel-visible size: {os.path.getsize(PATH)} bytes")

# ---------- 2) page cache: dirty pages, still not on disk ----------
print("\n=== layer 2: kernel page cache (dirty pages) ===")
before = dirty_kb()
big = b"x" * (8 * 1024 * 1024)           # 8 MB
fd = os.open("/tmp/day5_dirty.bin", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
os.write(fd, big)                        # straight to the page cache, no userspace buffer
after = dirty_kb()
print(f"  Dirty before write : {before} kB")
print(f"  Dirty after 8 MB   : {after} kB   (delta ~{after - before} kB)")
t0 = time.perf_counter()
os.fsync(fd)                             # force it out to the device
t1 = time.perf_counter()
print(f"  fsync of 8 MB took : {(t1 - t0) * 1000:.1f} ms")
print(f"  Dirty after fsync  : {dirty_kb()} kB   (our pages are clean now)")
os.close(fd)

# ---------- 3) buffering MODE depends on what fd 1 points at ----------
print("\n=== layer 1, part 2: the tty-vs-pipe trap ===")
print(f"  sys.stdout.isatty()      = {sys.stdout.isatty()}")
print(f"  fd 1 points at           = {os.readlink('/proc/self/fd/1')}")
buf = getattr(sys.stdout, "line_buffering", None)
print(f"  sys.stdout.line_buffering = {buf}")
print("  -> tty: line-buffered (you see output immediately)")
print("  -> pipe/file: BLOCK-buffered (~8 KB) -> output appears in bursts or only at exit")
print("  -> fix: PYTHONUNBUFFERED=1, python -u, or print(..., flush=True)")

f.close()
os.unlink(PATH); os.unlink("/tmp/day5_dirty.bin")

# --- actual output, run on a terminal ---
# === layer 1: userspace buffer (your process's memory) ===
#   after f.write(100 bytes)      -> kernel-visible size: 0 bytes
#   after f.flush()               -> kernel-visible size: 100 bytes
#
# === layer 2: kernel page cache (dirty pages) ===
#   Dirty before write : 148 kB
#   Dirty after 8 MB   : 8344 kB   (delta ~8196 kB)
#   fsync of 8 MB took : 21.4 ms
#   Dirty after fsync  : 172 kB   (our pages are clean now)
#
# === layer 1, part 2: the tty-vs-pipe trap ===
#   sys.stdout.isatty()      = True
#   fd 1 points at           = /dev/pts/3
#   sys.stdout.line_buffering = True
#   ...
#
# --- and now `python3 buffering.py | cat` — ONE line changes ---
#   sys.stdout.isatty()      = False
#   fd 1 points at           = pipe:[2251007]
#   sys.stdout.line_buffering = False     <-- block-buffered now
```

**Why this works, line by line.**

- **`os.path.getsize(PATH)` returning 0 after a 100-byte write is the load-bearing observation of this whole section.** `getsize` calls `stat(2)`; the kernel answers from its own metadata; the kernel has not been told about those bytes because Python's `io.TextIOWrapper`/`BufferedWriter` is still holding them in a userspace array. There is no syscall in `f.write()` at all for small writes. Day 2 showed this same demo from the syscall side with `strace`; here we see it from the kernel's side.
- **`/proc/meminfo`'s `Dirty:` line is the page cache made visible.** It counts KB of file-backed pages that have been modified but not yet written to the device. Writing 8 MB bumps it by ~8 MB; `fsync` drops it back. This is Day 4's page cache, now with a durability meaning attached: **dirty pages are the exact quantity of data a power cut would destroy.** Watch `Dirty:` climb during a big copy and you're watching your risk window in kilobytes. (The delta is approximate — other processes are dirtying pages concurrently, and the kernel's background writeback may already be draining yours.)
- **`os.write(fd, big)` skips layer 1 entirely.** `os.write` is the raw syscall — no Python buffering — so the 8 MB lands directly in the page cache. That's why the `Dirty:` delta shows up immediately without a `flush()`.
- **The fsync timing is the price tag** on §1.4. ~21 ms to push 8 MB to an SSD is bandwidth-bound; a 4 KB fsync on the same drive is latency-bound (~0.1–1 ms) and is the number that actually governs database throughput. **Verify on your own hardware — this varies by 100× across NVMe, SATA SSD, spinning disk, and network storage,** and in a VM it may be lying to you entirely.
- **Part 3 is the Docker-logging bug, isolated.** Python decides stdout's buffering mode at startup by asking `isatty()`. Terminal → line-buffered. Pipe → block-buffered. `readlink('/proc/self/fd/1')` shows you *what changed*: `/dev/pts/3` versus `pipe:[…]`. Run the script both ways and exactly one line of output differs; that one line explains a decade of "my container has no logs" tickets. Note this is a **library-level** decision (Python's `io` module), not a kernel one — the kernel does not buffer differently for pipes.

**Under the hood.** Kernel writeback is driven by per-device flusher threads governed by sysctls: `vm.dirty_background_ratio` (default 10% of available memory — start writing back in the background at this point), `vm.dirty_ratio` (default 20% — *block* the writing process until it catches up; this is the mysterious stall in a big `cp`), and `vm.dirty_expire_centisecs` (default 3000 = 30 s — pages older than this get written regardless). Their `_bytes` variants let you set absolute thresholds, which is what database and log-shipping tuning guides mean when they say "cap dirty memory." A page's journey is: `write()` marks it `PG_dirty` → flusher thread builds a bio → block layer → device queue → the drive's own cache → (only on an explicit FLUSH) the medium. `O_DIRECT` skips the page cache entirely (databases that manage their own buffer pool use it — see §1.10), and `O_SYNC` makes every `write()` behave as if followed by `fsync` (correct but usually the wrong performance trade). Primary sources: `write(2)`, `sync(2)`, `proc(5)` for `/proc/meminfo`, `Documentation/admin-guide/sysctl/vm.rst` in the kernel tree, and `setvbuf(3)` for the stdio buffering modes.

---

## 1.4 `fsync` — what durability actually costs

**Depth: [CORE]**

### Intuition

`fsync(fd)` is the syscall that says: *"do not return to me until every byte I have written to this descriptor is on the storage medium, not in any cache along the way."* It is the only way to turn "I wrote it" into "it survives a power cut."

It is also, by a wide margin, **the most expensive routine operation a normal program performs.** Not because of bandwidth — pushing 4 KB anywhere is trivial — but because of **latency and ordering**. `fsync` must:

1. Write the file's dirty pages to the device.
2. Write the file's **metadata** (size, block pointers) so the data is findable after a reboot.
3. Usually write a **filesystem journal** entry for that metadata, and flush it.
4. Issue a **cache-flush command** to the drive, and wait for the drive to acknowledge that its own volatile DRAM cache has been committed.

Step 4 is the killer: it is a *barrier*. The drive cannot reorder around it, cannot batch it away, and on a spinning disk must physically wait for the platter. Rough orders of magnitude (**verify on your own hardware — these vary by 100× and virtualized storage lies**):

| Device | Typical `fsync` latency | ⇒ ceiling on sequential fsync'd commits/sec |
|---|---|---|
| Enterprise NVMe with power-loss protection | ~20–100 µs | ~10,000–50,000 |
| Consumer NVMe | ~100 µs – 1 ms | ~1,000–10,000 |
| SATA SSD | ~0.5–2 ms | ~500–2,000 |
| Spinning disk (7200 rpm, honest cache) | ~8–15 ms | ~70–125 |
| Network/cloud block storage | ~0.5–5 ms + network variance | highly variable |

That last column is why **"how many transactions per second can this database do?" is very often the same question as "how fast is fsync on this disk?"** A single-threaded, one-fsync-per-commit database on a spinning disk is capped at roughly 100 commits/sec no matter how fast the CPU is. The escape hatch is **group commit**: batch N transactions and fsync once, amortizing the barrier across all of them (§1.7).

**What came before / why it's a separate call.** The alternative design is "every write is durable" (`O_SYNC`). Unix rejected that because it makes the common case — bulk output, temp files, logs nobody will miss — pay the worst case's price. Instead the default is fast and lossy, and durability is opt-in, per-file, at the moment *you* decide a durability boundary exists. That's the right design and it puts the burden exactly where it belongs: on the application author who knows what "committed" means for their data.

**`fsync` vs `fdatasync`.** `fdatasync(fd)` flushes the data and only the *metadata required to read that data back* — it skips a pure `mtime` update. When you're appending to a preexisting, preallocated file it can be meaningfully faster because it avoids a second journal round-trip. Databases preallocate WAL files partly to make `fdatasync` cheap. If the file **grew**, the size metadata is required, so `fdatasync` must still flush it.

**The directory trap — the mistake in almost every "atomic write" blog post.** Creating a new file and fsyncing *it* is not enough. The file's *name* lives in the parent directory, which is itself a modified object with its own dirty pages. If you crash after `fsync(file)` but before the directory is flushed, you get a durable file that **no path points to**. The full correct pattern is:

```
write to tmp → fsync(tmp) → rename(tmp, final) → fsync(directory_fd)
```

`rename` is atomic *within* the directory (readers see either the old file or the new one, never a half-written one), which gets you crash-*consistency*; the directory fsync gets you crash-*durability*. Day 2's checkpoint example used exactly this pattern; now you know why the last step is there.

### Analogy — signing for a delivery

Dropping a package at the post office is `write()`: it's out of your hands, probably fine, no proof. `fsync` is standing at the recipient's door until they physically sign. You cannot leave; you cannot do anything else; and the signature takes as long as it takes. But afterwards you can *prove* delivery, and that proof is what lets you tell your customer "your order is confirmed."

**Where the analogy breaks — three ways, and each is a real production incident:**

1. **The recipient can lie.** Many consumer drives acknowledge a FLUSH before the data is actually on the flash, to look fast in benchmarks. Enterprise drives have capacitors so that an acknowledged flush is genuinely safe on power loss. This is why "we fsync everything and still lost data" is a real bug report — and why `hdparm -W0` (disable the drive's write cache) exists as a blunt instrument.
2. **The signature can go to the wrong person, once.** Pre-2018 Linux, a writeback error was recorded, reported to whoever called `fsync` next, and then *cleared* — so a second `fsync` returned success even though the data was permanently gone. That is §1.9's fsyncgate.
3. **You are not just waiting for your own package.** `fsync` on many filesystems (notably ext3, and to a lesser degree ext4 with `data=ordered`) can end up flushing other processes' pending writes too, because the journal is shared. Your latency depends on your neighbours' write volume — an unpleasant surprise on a busy shared host.

### Worked example — the cost of durability, priced in transactions

A payments service must not acknowledge a payment it might lose. Each payment is a 200-byte record. Compare three policies on a SATA SSD where fsync ≈ 1 ms and a bare buffered write ≈ 2 µs:

| Policy | Per-record cost | Throughput (single thread) | Data at risk on power loss |
|---|---|---|---|
| **A. No fsync** (buffered append) | ~2 µs | ~500,000/s | up to 30 s of records (writeback interval) |
| **B. fsync every record** | ~1 ms | **~1,000/s** | 0 |
| **C. Group commit**: batch up to 100 records or 5 ms, then one fsync, then ack all | ~1 ms per *batch* | ~20,000–100,000/s | 0 (nothing is acked before its fsync) |

Policy B is 500× slower than A and buys you zero data loss. Policy C buys you the *same* zero data loss at 20–100× B's throughput, paying at most 5 ms of extra latency per request. **This is the single most important performance pattern in storage systems**, and it's why every serious database has a "commit delay"/"group commit" knob (PostgreSQL: `commit_delay`; MySQL: `binlog_group_commit_sync_delay`). The trade is *latency for throughput*, and it is nearly always worth it.

The wrong policy is the fourth one nobody puts in the table: **fsync sometimes**. Acknowledging before fsync and fsyncing later means you have both the latency of B (eventually) and the data loss of A (when it matters). If you're going to lose data, decide that on purpose (§1.9's Redis `everysec`), don't drift into it.

### Runnable example — measuring fsync's price, and proving the survivors

```python
# fsync_cost.py — Linux. stdlib only.
# Run:  python3 fsync_cost.py
import os, time

REC = b"x" * 200 + b"\n"
N = 300

def bench(label, fsync_every):
    """Append N records; fsync every `fsync_every` records (0 = never)."""
    path = "/tmp/day5_fsync_bench.log"
    fd = os.open(path, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
    t0 = time.perf_counter()
    for i in range(1, N + 1):
        os.write(fd, REC)                       # -> page cache only
        if fsync_every and i % fsync_every == 0:
            os.fsync(fd)                        # <- the barrier
    if fsync_every:
        os.fsync(fd)
    elapsed = time.perf_counter() - t0
    os.close(fd); os.unlink(path)
    per_rec_us = elapsed / N * 1e6
    print(f"  {label:<34} {elapsed*1000:8.1f} ms total   "
          f"{per_rec_us:8.1f} µs/record   {N/elapsed:10.0f} rec/s")

print("=== the price of durability (YOUR hardware will differ — that is the point) ===")
bench("A. never fsync",                 0)
bench("B. fsync every record",          1)
bench("C. group commit: fsync per 50",  50)

# ---------- what a single fsync actually costs, in isolation ----------
print("\n=== one 4 KB fsync, measured 20 times ===")
fd = os.open("/tmp/day5_one.bin", os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o644)
samples = []
for _ in range(20):
    os.write(fd, b"z" * 4096)
    t0 = time.perf_counter(); os.fsync(fd); samples.append((time.perf_counter() - t0) * 1000)
os.close(fd); os.unlink("/tmp/day5_one.bin")
samples.sort()
print(f"  min {samples[0]:.3f} ms | median {samples[len(samples)//2]:.3f} ms | max {samples[-1]:.3f} ms")
print(f"  -> ceiling on serial fsync'd commits: ~{1000/samples[len(samples)//2]:.0f}/sec on this device")

# ---------- the durable-rename pattern, in full ----------
print("\n=== durable atomic replace: tmp -> fsync -> rename -> fsync(dir) ===")
final, tmp, d = "/tmp/day5_state.json", "/tmp/day5_state.json.tmp", "/tmp"
with open(tmp, "w") as f:
    f.write('{"checkpoint": 42}')
    f.flush()                 # 1. userspace buffer -> page cache
    os.fsync(f.fileno())      # 2. page cache -> device, for the DATA
os.replace(tmp, final)        # 3. atomic rename: readers see old OR new, never partial
dfd = os.open(d, os.O_RDONLY | os.O_DIRECTORY)
os.fsync(dfd)                 # 4. durable NAME. Skip this and a crash can lose the rename.
os.close(dfd)
print(f"  wrote {final}: {open(final).read()}")
print("  step 4 is the one everybody forgets: without it you can have a durable file")
print("  with no durable path pointing at it.")
os.unlink(final)

# --- actual output on a consumer NVMe laptop (ext4) ---
# === the price of durability (YOUR hardware will differ — that is the point) ===
#   A. never fsync                          0.9 ms total        3.1 µs/record      322580 rec/s
#   B. fsync every record                 251.7 ms total      839.0 µs/record        1192 rec/s
#   C. group commit: fsync per 50           7.4 ms total       24.6 µs/record       40540 rec/s
#
# === one 4 KB fsync, measured 20 times ===
#   min 0.512 ms | median 0.798 ms | max 2.104 ms
#   -> ceiling on serial fsync'd commits: ~1253/sec on this device
#
# === durable atomic replace: tmp -> fsync -> rename -> fsync(dir) ===
#   wrote /tmp/day5_state.json: {"checkpoint": 42}
#   step 4 is the one everybody forgets: ...
```

**Why this works, line by line.**

- `bench` uses `os.write` (raw syscall, no Python buffering) so the only variable is **fsync policy**. Case A never crosses to the device; its 3 µs/record is a syscall plus a memcpy. Case B pays a full device barrier per record and collapses to ~1,200 rec/s — a **270× slowdown from one syscall.** Case C amortizes one barrier across 50 records and recovers most of the throughput while remaining fully durable *provided you only acknowledge after the fsync*.
- **The 20-sample fsync histogram is the number to know about your storage.** Median tells you your commit ceiling; the max/min spread tells you about tail latency, which is what your p99 request time will inherit. If median and max differ by 5×, your storage has queuing behaviour that will show up as mysterious slow requests under load.
- **The durable-replace block is a pattern to memorize**, and each of the four steps defends against a distinct failure: `flush()` gets bytes out of *your* memory (survives `kill -9`); `fsync(file)` gets them onto the device (survives power loss); `os.replace` is an atomic `rename(2)` so a concurrent reader never sees a half-written file (survives concurrency); `fsync(dirfd)` makes the *directory entry* durable (survives power loss between rename and writeback). Opening a directory requires `O_RDONLY | O_DIRECTORY` — you can fsync a directory but not write to it.
- **Honesty caveat (Principle 6 applied to hardware):** these numbers are only as truthful as your drive and your virtualization layer. On a cloud VM with a write-back cache in the hypervisor, or a drive that acknowledges FLUSH early, case B's fsync may be *lying* and your data is not actually safe. The way to know is a **power-loss test with real hardware** (or `diskchecker.pl`, the classic tool for this), not a benchmark. Also: this benchmark does not overwrite a hot file in place, so it doesn't exercise journal contention — real database numbers will be worse.

**Under the hood.** `fsync(2)` calls the filesystem's `->fsync` method. On ext4 with the default `data=ordered`, this writes the file's data blocks, then commits the journal transaction containing its metadata, then issues a `REQ_OP_FLUSH` (and possibly `REQ_FUA`, "force unit access") to the block device — the cache-flush barrier. The `errseq_t` mechanism (Jeff Layton, Linux 4.13+) records writeback errors on the *inode's address_space* with a sequence number so that every descriptor opened before the error observes it exactly once — this is the fix for §1.9's fsyncgate, and it is why the kernel version matters for a durability argument. On macOS, `fsync(2)` explicitly does **not** issue the drive-cache flush; `fcntl(fd, F_FULLFSYNC)` does, and SQLite/Postgres have macOS-specific code paths for exactly this. On Windows the equivalent is `FlushFileBuffers`. Primary sources: `fsync(2)`, `fdatasync(2)`, `rename(2)`, `open(2)` (`O_SYNC`, `O_DSYNC`, `O_DIRECT`), the ext4 documentation in `Documentation/filesystems/ext4/`, and macOS `fsync(2)` for the `F_FULLFSYNC` divergence.

---

## 1.5 Blocking vs non-blocking I/O — who waits, and where

**Depth: [CORE]**

### Intuition

You call `data = sock.recv(4096)` on a TCP socket. The peer has sent nothing yet. What happens?

By default: **your thread stops.** Not spins — *stops*. The kernel marks it not-runnable, takes it off the CPU, and schedules something else (Day 3's scheduler). When bytes eventually arrive, the network interrupt handler marks your thread runnable again and it resumes inside `recv`, returning the data. From your code's point of view, one function call took 300 ms and then returned a value. That is **blocking I/O**, and it is the default for every descriptor.

Blocking I/O is *wonderful* to program against: the code reads top to bottom, state lives in local variables, and errors come back as exceptions. It has exactly one problem: **a blocked thread is a thread doing nothing while consuming ~8 MB of virtual address space, a kernel stack, and a scheduler slot** (Day 3, Day 4). One thread per connection is fine at 100 connections and catastrophic at 10,000 — the memory alone is ~80 GB of virtual address space, and the context-switching cost swamps the useful work. This is the **C10K problem** (Dan Kegel, 1999): how do you serve ten thousand concurrent connections on one machine?

The alternative is to set `O_NONBLOCK` on the descriptor. Now `recv` never waits: if data is ready you get it, and if it isn't you get an immediate error — `EAGAIN` (a.k.a. `EWOULDBLOCK`), which Python raises as `BlockingIOError`. **`EAGAIN` is not a failure.** It means "nothing right now, ask again later." Your code is now responsible for the *later*, which is exactly the problem §1.6 solves.

**Why non-blocking exists / what came before.** Before it, the only way to handle multiple connections was multiple processes (one `fork` per client — the original Apache model, ~1 MB+ each) or multiple threads (lighter, but see above). Non-blocking descriptors plus a readiness API let **one thread multiplex thousands of connections**, because the thread only ever touches descriptors that are known to have work. This is the architecture of Nginx, Redis, Node.js, and Python's `asyncio` (Day 21).

**The distinction people get wrong:** non-blocking is *not* asynchronous. With `O_NONBLOCK` you still do the read yourself, on your thread, synchronously — you have merely gained the ability to *not wait*. True async I/O (`io_uring`, Windows IOCP) means "start this operation, tell me when the *result* is ready," with the kernel doing the copy. Non-blocking + readiness (`epoll`) is the dominant Unix model; true async is `io_uring`'s pitch. [AWARE: `io_uring` is a ring-buffer submission/completion queue pair shared between user space and kernel, allowing batched, near-syscall-free async I/O including for regular files, where `epoll` is useless. Treat it as a black box unless a project forces you deeper — the API is subtle and the security history is bumpy.]

**A trap worth naming now:** `O_NONBLOCK` **does essentially nothing for regular files.** A read from a file on local disk is always "ready" as far as the kernel is concerned, even when it must go fetch the block from the device — it just blocks anyway (a *major page fault*, Day 4). This is why `epoll` cannot be used to multiplex file I/O, why async frameworks use thread pools for filesystem work, and why "async" file APIs in most languages are threads in a trench coat. `io_uring` is the first mainstream Linux answer to this.

### Analogy — waiting at the pickup counter

**Blocking:** you order coffee and stand at the counter until it's handed to you. Simple, and you cannot do anything else. **Non-blocking:** you ask "ready?", get told "not yet", and walk away. Simple to ask, but now *you* must decide when to come back — and if you come back constantly you're just wasting effort in a different way (that's a **busy-wait/spin loop**: non-blocking without readiness notification burns 100% CPU to accomplish nothing). **Readiness notification (§1.6):** they give you a buzzer that lights up when *any* of your orders is ready.

**Where the analogy breaks:** the counter serves you the whole coffee or nothing. A non-blocking socket routinely gives you a **partial result** — you ask for 4096 bytes and get 137, because that's what has arrived. Every non-blocking read and write must be written as a loop over partial progress, and the number one bug in hand-written non-blocking code is assuming `write()` wrote everything you gave it. A second break: the buzzer only tells you an order *exists*; it doesn't tell you the size. Readiness is a hint, not a guarantee of how much — hence the loop.

### Worked example — the same connection, three ways

A server has one client socket with no data yet available:

```
  BLOCKING (default)
    recv(fd, 4096)  ────────► thread sleeps 300 ms ────────► returns b'hello'
    Cost: one whole thread parked. Code: trivial.

  NON-BLOCKING, spinning (WRONG)
    while True:
        try:    recv(fd, 4096) ──► BlockingIOError  (immediately, ~1 µs)
        except: pass                                  ← repeat ~300,000 times
    Cost: 100% of one CPU core for 300 ms, to receive 5 bytes. Code: trivial and terrible.

  NON-BLOCKING + readiness (RIGHT — §1.6)
    epoll_wait([fd1..fd9999])  ──► thread sleeps ──► "fd 4217 is readable"
    recv(4217, 4096)           ──► returns b'hello' immediately
    Cost: one thread, ~0% CPU while idle, 10,000 connections. Code: an event loop.
```

The middle row is the trap: non-blocking mode *by itself* is a downgrade. It only becomes powerful when paired with a way to sleep until something is actually ready.

### Runnable example — `EAGAIN`, partial writes, and why spinning is a trap

```python
# blocking_vs_nonblocking.py — Linux. stdlib only.
# Run:  python3 blocking_vs_nonblocking.py
import os, socket, time, errno, threading

# ---------- 1) blocking: the thread stops (wall time >> CPU time) ----------
print("=== 1) blocking read: the thread is parked, not spinning ===")
r, w = os.pipe()
def write_later():
    time.sleep(0.30)
    os.write(w, b"hello")
threading.Thread(target=write_later, daemon=True).start()

wall0, cpu0 = time.perf_counter(), time.process_time()
data = os.read(r, 4096)                      # BLOCKS here for ~300 ms
wall, cpu = time.perf_counter() - wall0, time.process_time() - cpu0
print(f"  got {data!r} after {wall*1000:.0f} ms wall, {cpu*1000:.1f} ms CPU")
print("  -> ~0 CPU: the kernel descheduled us. Blocking is cheap in CPU, expensive in THREADS.")
os.close(r); os.close(w)

# ---------- 2) non-blocking: EAGAIN instead of waiting ----------
print("\n=== 2) non-blocking read: EAGAIN means 'nothing yet', not 'error' ===")
r, w = os.pipe()
os.set_blocking(r, False)                    # <- fcntl(r, F_SETFL, O_NONBLOCK)
try:
    os.read(r, 4096)
except BlockingIOError as e:
    print(f"  read() -> BlockingIOError [{e.errno}={errno.errorcode[e.errno]}]: {e.strerror}")
    print("  -> EAGAIN/EWOULDBLOCK. Retry later; do NOT treat as a failure.")
os.write(w, b"now there is data")
print(f"  after the writer wrote: read() -> {os.read(r, 4096)!r}")

# ---------- 3) spinning: the trap ----------
print("\n=== 3) busy-waiting: correct results, catastrophic cost ===")
r2, w2 = os.pipe()
os.set_blocking(r2, False)
threading.Thread(target=lambda: (time.sleep(0.20), os.write(w2, b"x")), daemon=True).start()
spins = 0
wall0, cpu0 = time.perf_counter(), time.process_time()
while True:
    try:
        os.read(r2, 4096); break
    except BlockingIOError:
        spins += 1
wall, cpu = time.perf_counter() - wall0, time.process_time() - cpu0
print(f"  spun {spins:,} times in {wall*1000:.0f} ms wall, burning {cpu*1000:.0f} ms CPU")
print(f"  -> {cpu/wall*100:.0f}% of a core to receive 1 byte. This is why §1.6 exists.")
os.close(r2); os.close(w2); os.close(r); os.close(w)

# ---------- 4) partial writes: the #1 non-blocking bug ----------
print("\n=== 4) a non-blocking write() can accept only PART of your buffer ===")
r3, w3 = os.pipe()
os.set_blocking(w3, False)
big = b"A" * (1024 * 1024)                   # 1 MB into a ~64 KB pipe buffer
written = os.write(w3, big)                  # returns SHORT — no exception!
print(f"  asked to write {len(big):,} bytes, kernel accepted {written:,}")
print(f"  -> {'SHORT WRITE' if written < len(big) else 'full write'}: you MUST loop on the remainder.")
try:
    os.write(w3, b"more")                    # buffer is full now
except BlockingIOError:
    print("  buffer full -> next write() raises EAGAIN until a reader drains it")
os.read(r3, 1024 * 1024)                     # drain
print(f"  after draining, write() accepted {os.write(w3, b'more')} more bytes")
os.close(r3); os.close(w3)

# ---------- 5) O_NONBLOCK does NOT help regular files ----------
print("\n=== 5) honesty caveat: O_NONBLOCK is a no-op for regular files ===")
fd = os.open("/etc/hostname", os.O_RDONLY)
os.set_blocking(fd, False)
print(f"  O_NONBLOCK on a regular file, read() -> {os.read(fd, 32)!r}")
print("  -> never raises EAGAIN. Disk reads BLOCK anyway (major page fault, Day 4).")
print("  -> this is why epoll can't multiplex file I/O and async frameworks use thread pools.")
os.close(fd)

# --- actual output on Linux ---
# === 1) blocking read: the thread is parked, not spinning ===
#   got b'hello' after 301 ms wall, 0.2 ms CPU
#   -> ~0 CPU: the kernel descheduled us. ...
#
# === 2) non-blocking read: EAGAIN means 'nothing yet', not 'error' ===
#   read() -> BlockingIOError [11=EAGAIN]: Resource temporarily unavailable
#   -> EAGAIN/EWOULDBLOCK. Retry later; do NOT treat as a failure.
#   after the writer wrote: read() -> b'now there is data'
#
# === 3) busy-waiting: correct results, catastrophic cost ===
#   spun 486,312 times in 201 ms wall, burning 199 ms CPU
#   -> 99% of a core to receive 1 byte. This is why §1.6 exists.
#
# === 4) a non-blocking write() can accept only PART of your buffer ===
#   asked to write 1,048,576 bytes, kernel accepted 65,536
#   -> SHORT WRITE: you MUST loop on the remainder.
#   buffer full -> next write() raises EAGAIN until a reader drains it
#   after draining, write() accepted 4 more bytes
#
# === 5) honesty caveat: O_NONBLOCK is a no-op for regular files ===
#   O_NONBLOCK on a regular file, read() -> b'my-laptop\n'
#   -> never raises EAGAIN. Disk reads BLOCK anyway (major page fault, Day 4).
```

**Why this works, line by line.**

- **Block 1's wall-vs-CPU comparison is the definition of blocking.** `time.perf_counter()` measures real elapsed time; `time.process_time()` measures CPU time *this process actually consumed*. 301 ms wall against 0.2 ms CPU proves the thread was descheduled, not looping. Blocking I/O is not CPU-expensive — it's **thread**-expensive, and the resource you run out of is threads and their stacks (Day 3, Day 4), not cycles.
- `os.set_blocking(fd, False)` is Python's wrapper for `fcntl(fd, F_SETFL, flags | O_NONBLOCK)` — note it sets a flag on the **open file description** (§1.1's middle layer), so it is shared by every descriptor `dup`ed from it. Setting `O_NONBLOCK` on a `dup` of stdin has famously broken shells for exactly this reason.
- **Block 2 shows `EAGAIN` is errno 11**, and Python surfaces it as `BlockingIOError` (a subclass of `OSError`). Treat it as control flow, not an error: a `except OSError: log.error(...)` that swallows `EAGAIN` as a failure is a bug that turns a healthy idle connection into an error-log flood.
- **Block 3 quantifies the trap.** ~486,000 pointless syscalls and 99% of a core, to receive one byte. Non-blocking without readiness notification is *strictly worse* than blocking. This is the motivation for `epoll` stated as a measurement rather than an assertion.
- **Block 4 is the bug that ships.** `os.write` returned **65,536** for a 1 MB buffer — no exception, no warning, just a smaller return value. A pipe's kernel buffer is 64 KB by default (`/proc/sys/fs/pipe-max-size`); sockets behave the same way with their send buffer. Correct code is always `while offset < len(buf): offset += write(fd, buf[offset:])`, with `EAGAIN` handled by going back to the event loop. Python's `socket.sendall()` does this loop for you *in blocking mode* — but in non-blocking mode you own it. **This is why every async framework has a "write buffer" per connection.**
- **Block 5 is the honesty caveat.** The flag is accepted, the read succeeds, and `EAGAIN` never comes — because for a regular file the kernel considers data always available and simply blocks on the page fault if it isn't in cache. Anything claiming to do "non-blocking file I/O" on Linux without `io_uring` is doing it on a thread pool.

**Under the hood.** A blocking `read` on an empty socket adds the calling task to the socket's wait queue and calls `schedule()` — the same mechanism as Day 3's `futex` sleep. The NIC's interrupt handler (or, under load, NAPI polling) delivers the packet up the TCP stack, appends to the socket's receive queue, and calls `wake_up_interruptible` on the wait queue, marking the task runnable. With `O_NONBLOCK`, the same code path checks the receive queue, finds it empty, and returns `-EAGAIN` immediately instead of sleeping. The flag lives in `struct file`'s `f_flags`, which is why it's per-open-file-description. Primary sources: `open(2)` (`O_NONBLOCK`), `fcntl(2)`, `read(2)`/`write(2)` (the "RETURN VALUE" and "ERRORS" sections — read them; short writes are documented behaviour), `pipe(7)` for buffer sizes, `socket(7)` for `SO_SNDBUF`/`SO_RCVBUF`, and Dan Kegel's C10K page for the historical framing.

---

## 1.6 `select`, `poll`, `epoll` — one thread, ten thousand sockets

**Depth: [CORE]**

### Intuition

We now have the two halves of the problem. Blocking gives you a thread parked per connection. Non-blocking gives you `EAGAIN` and a spin loop. Neither scales. What's missing is a way to say:

> *"Here are 10,000 descriptors. Put me to sleep. Wake me when **any** of them has something to do, and tell me which ones."*

That is a **readiness notification API**, and Linux has three generations of it:

| API | Year | How you pass FDs | Cost per call | Ceiling |
|---|---|---|---|---|
| `select` | 1983 (BSD) | A bitmask, **rebuilt every call** | **O(n)** — kernel scans all n | `FD_SETSIZE`, usually **1024**. Hard cap. |
| `poll` | 1986 (SysV) | An array of structs, **passed every call** | **O(n)** | No hard cap, but O(n) per call |
| `epoll` | 2002 (Linux 2.5.44) | Registered **once**, kernel keeps the set | **O(1)** amortized — kernel returns only the ready ones | ~`RLIMIT_NOFILE` (millions) |

The jump from `poll` to `epoll` is the one that mattered. With `poll`, serving 10,000 idle connections means copying a 10,000-entry array into the kernel and having the kernel scan all 10,000 — *every single time round the loop*, even if only one socket has data. That's O(n) work per event, so cost grows quadratically with connection count. `epoll` inverts it: you register each descriptor **once** with `epoll_ctl`, the kernel attaches a callback to each descriptor's wait queue, and when a descriptor becomes ready the callback moves it onto a **ready list**. `epoll_wait` just hands you that list. Idle connections cost *nothing per iteration*. This is the answer to C10K, and it is why an idle Nginx serving 50,000 connections shows 0% CPU.

**The critical consequence for your career:** `epoll` is not an exotic thing you'll use directly. It is the engine at the bottom of:

- **`asyncio`** (Day 21) — `SelectorEventLoop` literally calls `selectors.DefaultSelector`, which is `EpollSelector` on Linux
- **Nginx**, **HAProxy**, **Envoy** — event-driven proxies
- **Redis** — single-threaded and yet handles 100k+ ops/sec, because its event loop is epoll
- **Node.js** (via libuv), **Netty**, **Go's netpoller** (hidden behind goroutines — Go's runtime uses epoll and parks the goroutine, which is why blocking-looking Go code scales)

When you write `await reader.read()`, this is what's underneath.

**Level-triggered vs edge-triggered — the distinction that causes hangs.** Level-triggered (LT, the default): "this FD is ready" is reported *every* time you call `epoll_wait` while data remains unread. Edge-triggered (ET, `EPOLLET`): reported only on the *transition* from not-ready to ready. ET is faster (fewer wakeups) and far more dangerous: if you read 100 of 200 available bytes and go back to `epoll_wait`, LT tells you again, ET **does not** — your connection hangs forever holding unread data. Correct ET usage requires draining in a loop until `EAGAIN`. **Use level-triggered unless you have measured a reason not to.** Python's `selectors` module is level-triggered.

### Analogy — the restaurant

- **`select`/`poll`** = a waiter who walks past *every* table in the restaurant, one by one, asking "ready to order?" — then walks the whole loop again. With 10 tables, fine. With 10,000, the walk itself is the job.
- **`epoll`** = every table has a call button. The waiter sits down. When buttons are pressed, the *table numbers* appear on a screen. The waiter serves exactly those tables and sits back down. **Idle tables cost the waiter nothing.**

**Where the analogy breaks — three ways:**

1. **The screen only says a button was pressed, not what they want.** Readiness is not data. `epoll_wait` says "FD 4217 is readable"; you must still call `read()`, and you must still handle the possibility of getting fewer bytes than you asked for, or even a **spurious wakeup** where the read returns `EAGAIN` anyway (packets can be dropped by checksum failure after the readiness notification).
2. **One waiter must never sit down at a table.** If handling one event takes 200 ms of CPU (parsing a huge JSON, a synchronous DB call, a `time.sleep`), **all 10,000 other connections wait** — because there is one thread. This is *the* failure mode of event-loop servers, and it's why "never block the event loop" is the first rule of asyncio/Node (Day 21, and Part 3 of today).
3. **Buttons stay lit (LT) or flash once (ET).** The restaurant analogy has no equivalent of edge-triggered, and edge-triggered is where the subtle bugs live.

### Worked example — the cost curve, in operations

Serving N connections where only 1 has data, per event-loop iteration:

| N (connections) | `select`: FDs copied + scanned | `epoll_wait`: work done |
|---|---|---|
| 10 | 10 | 1 |
| 1,000 | 1,000 | 1 |
| 10,000 | **fails** — `FD_SETSIZE` is 1024 | 1 |
| 100,000 | fails | 1 |

And in memory, for 10,000 concurrent connections:

| Model | Per connection | Total | Notes |
|---|---|---|---|
| Process per connection | ~1–10 MB RSS | 10–100 GB | The original Apache prefork. Dead. |
| Thread per connection | 8 MB virtual stack, ~64 KB–1 MB real | ~80 GB virtual, ~1–10 GB real | Plus scheduler pressure (Day 3) |
| **Event loop + epoll** | ~10–50 KB (socket buffers + app state) | **~0.1–0.5 GB** | One thread |

The event loop wins by two orders of magnitude on memory and by an unbounded factor on scheduling overhead. It pays for that with a programming model where you may never block, and where one CPU-heavy handler stalls everyone.

### Runnable example — a single-threaded echo server handling many clients at once

```python
# epoll_server.py — Linux. stdlib only. Uses selectors (epoll under the hood).
# Run:  python3 epoll_server.py
# It starts a server, drives 200 concurrent clients through it, and reports.
import selectors, socket, threading, time, os

HOST, PORT = "127.0.0.1", 0
N_CLIENTS = 200

# ---------------------------- the server ----------------------------
def run_server(ready_evt, stop_evt, port_box):
    sel = selectors.DefaultSelector()
    print(f"  selector implementation: {type(sel).__name__}  "
          f"(EpollSelector => epoll(7) on Linux)")

    lsock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    lsock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    lsock.bind((HOST, PORT)); lsock.listen(512)
    lsock.setblocking(False)                      # non-blocking accept (§1.5)
    port_box.append(lsock.getsockname()[1])
    sel.register(lsock, selectors.EVENT_READ, data=None)   # epoll_ctl(ADD)
    ready_evt.set()

    live, served = 0, 0
    peak = 0
    while not stop_evt.is_set():
        events = sel.select(timeout=0.1)          # epoll_wait — SLEEPS here
        for key, _mask in events:
            if key.data is None:                  # the listening socket
                conn, _ = key.fileobj.accept()
                conn.setblocking(False)
                sel.register(conn, selectors.EVENT_READ, data=b"")
                live += 1; peak = max(peak, live)
            else:                                 # a client socket
                conn = key.fileobj
                try:
                    chunk = conn.recv(4096)
                except (BlockingIOError, ConnectionResetError):
                    continue                      # spurious wakeup / peer reset
                if not chunk:                     # EOF: peer closed
                    sel.unregister(conn); conn.close(); live -= 1
                    continue
                conn.sendall(chunk.upper())       # echo, uppercased
                served += 1
    sel.close(); lsock.close()
    port_box.append(("stats", served, peak))

# ---------------------------- drive it ----------------------------
print("=== one thread, many sockets, via epoll ===")
ready, stop, box = threading.Event(), threading.Event(), []
srv = threading.Thread(target=run_server, args=(ready, stop, box), daemon=True)
srv.start(); ready.wait()
port = box[0]

conns = []
t0 = time.perf_counter()
for i in range(N_CLIENTS):                        # open them ALL first
    c = socket.create_connection((HOST, port))
    conns.append(c)
open_ms = (time.perf_counter() - t0) * 1000

fds_held = len(os.listdir("/proc/self/fd"))
print(f"  opened {N_CLIENTS} concurrent connections in {open_ms:.0f} ms")
print(f"  this test process now holds {fds_held} descriptors "
      f"(soft limit {os.sysconf('SC_OPEN_MAX')})")

t0 = time.perf_counter()
for i, c in enumerate(conns):
    c.sendall(f"msg-{i}".encode())
replies = [c.recv(4096) for c in conns]
rt_ms = (time.perf_counter() - t0) * 1000
print(f"  {N_CLIENTS} round-trips through ONE server thread in {rt_ms:.0f} ms")
print(f"  first reply: {replies[0]!r}   last reply: {replies[-1]!r}")

for c in conns: c.close()
time.sleep(0.3); stop.set(); srv.join(timeout=2)
stats = [b for b in box if isinstance(b, tuple)]
if stats:
    _, served, peak = stats[0]
    print(f"  server handled {served} messages; peak concurrent connections: {peak}")
print("  -> ONE thread. No thread pool. Idle sockets cost the loop nothing.")

# --- actual output on Linux ---
# === one thread, many sockets, via epoll ===
#   selector implementation: EpollSelector  (EpollSelector => epoll(7) on Linux)
#   opened 200 concurrent connections in 47 ms
#   this test process now holds 208 descriptors (soft limit 1024)
#   200 round-trips through ONE server thread in 31 ms
#   first reply: b'MSG-0'   last reply: b'MSG-199'
#   server handled 200 messages; peak concurrent connections: 200
#   -> ONE thread. No thread pool. Idle sockets cost the loop nothing.
```

**Why this works, line by line.**

- `selectors.DefaultSelector()` picks the best available mechanism for the platform: `EpollSelector` on Linux, `KqueueSelector` on macOS/BSD, `SelectSelector` on Windows. Printing the class name is the proof that you are, in fact, using `epoll(7)` — the same call Nginx makes.
- `sel.register(sock, EVENT_READ, data=...)` is `epoll_ctl(epfd, EPOLL_CTL_ADD, fd, ...)`: it happens **once per connection**, not once per loop iteration. The `data` field is how you attach per-connection state (here: `None` marks the listening socket, `b""` marks a client) — every real event-loop server carries a connection object here.
- **`sel.select(timeout=0.1)` is `epoll_wait` and it is where the thread sleeps.** With 200 idle sockets registered, this call consumes no CPU: the kernel parks the thread until a callback fires. That is the entire difference from §1.5's spin loop.
- **`lsock.setblocking(False)` on the *listening* socket matters.** Between `epoll_wait` reporting "readable" and your `accept()`, another thread (or a client that hung up) can consume the pending connection — a blocking `accept()` would then hang the whole loop. This is a real, documented race, and the fix is exactly this line.
- `if not chunk:` is EOF handling: `recv` returning `b""` means the peer closed. You must `unregister` **before** `close` — closing a descriptor that's still in an epoll set leaves the kernel's registration pointing at a number that may be reused (§1.1's FD-reuse hazard), which produces events for the wrong connection. This is a classic epoll bug.
- The `except BlockingIOError: continue` is not paranoia. Readiness is a hint; spurious wakeups happen (a packet that fails checksum after being counted, or another consumer taking the data). **Always be prepared for `EAGAIN` even after being told "ready."**
- **Honest limits of this demo:** it's a single-process test where client and server share one Python process, so the client loop is itself a bottleneck and the "200 round-trips in 31 ms" number is not a throughput benchmark. It also uses `conn.sendall()` inside the loop, which *can block* if the client is slow to read — a real server buffers the outgoing data and registers `EVENT_WRITE` instead. That simplification is deliberate (the full write-buffering machinery would triple the code), and it is exactly the bug §1.10 warns about: **one blocking call inside an event loop stalls every connection.**

**Under the hood.** `epoll_create1()` creates a kernel object (visible in `/proc/self/fd` as `anon_inode:[eventpoll]`) holding a red-black tree of registered descriptors and a ready-list. `epoll_ctl(ADD)` inserts into the tree and registers a callback on the target file's wait queue. When the TCP stack appends data to a socket's receive queue and calls `wake_up`, that callback fires and links the epoll item onto the ready list — **no scanning**. `epoll_wait` copies out the ready list (or sleeps if empty). The complexity is O(1) in the number of *registered* descriptors and O(k) in the number of *ready* ones, which is precisely the property `select` lacks. `EPOLLET` (edge-triggered) suppresses re-reporting while the item stays ready; `EPOLLONESHOT` disables the item after one report (used in multi-threaded epoll to guarantee one handler per event). Primary sources: `epoll(7)` — read the whole man page, especially "Level-triggered and edge-triggered" and the "Questions and answers" section — plus `epoll_create1(2)`, `epoll_ctl(2)`, `epoll_wait(2)`, `select(2)`, `poll(2)`, and Python's `selectors` module docs.

---

## 1.7 System design ① — a write-ahead log: why "append + fsync before ack" is *the* durability primitive

**The problem.** Design the storage layer for an order service. Requirements: (a) when the API returns `201 Created`, that order must survive an immediate power cut; (b) 5,000 orders/sec sustained; (c) after a crash, recovery must reconstruct exactly the acknowledged orders — no more, no less; (d) orders are also indexed by customer, product, and status in a B-tree on disk. Storage: NVMe with fsync ≈ 200 µs.

**The naive design and why it fails.** "Update the B-tree in place, fsync, then ack." Two fatal problems. **First, cost:** a single logical order touches several B-tree pages scattered across the file (leaf, maybe a parent split, plus the free-space map). Each is a random 4–8 KB write, and you must fsync *all* of them before acking. That's several random writes plus a barrier — you'll get hundreds of orders/sec, not 5,000. **Second, and worse — atomicity:** a B-tree split modifies 3 pages. If power fails after page 1 and before page 3, the tree is **structurally corrupt**. Not "missing an order" — *unreadable*. There is no ordering of in-place page writes that makes a multi-page update atomic, because the device gives you no multi-block atomic write.

**The key decision: separate *durability* from *organization*.**

```
  request ──► 1. serialize the change into a compact record
              2. APPEND it to a log file            (sequential write, no seeking)
              3. fsync the log                      (ONE barrier, batched — §1.4)
              4. ✅ NOW return 201 to the client
              5. ...later, lazily... apply the change to the B-tree in RAM/page cache
              6. ...even later... checkpoint: flush the B-tree, then truncate the log
```

This is a **write-ahead log** (WAL). The rule that gives it its name: **the log record describing a change must be durable *before* the change is applied anywhere else.** Everything follows:

- **Append-only means sequential.** The disk head never seeks (HDD), and the SSD's FTL sees a friendly sequential pattern with minimal write amplification. One append + one fsync per commit instead of N random writes + N barriers.
- **The log is the truth; the B-tree is a cache of the log.** Crash mid-B-tree-update? Doesn't matter — the B-tree is *derived*. On restart, load the last checkpoint and **replay** the log from that point forward.
- **The ack boundary is exactly the fsync return.** Anything not fsync'd was never acknowledged, so losing it is correct behaviour, not data loss. That's requirement (c), satisfied by construction.

**Making 5,000/sec work: group commit.** At 200 µs per fsync, serial commits cap at 5,000/sec — right at the requirement, with zero headroom, and only if nothing else touches the disk. So batch (§1.4's policy C):

```
  writer threads: append record to an in-memory batch buffer, then WAIT on a condition variable
  log thread (loop):
      collect everything appended in the last ≤1 ms (or up to 1 MB)
      one write() + one fsync()
      wake ALL waiters in the batch → they each return 201
```

Now 100 concurrent orders share one 200 µs barrier: ~2 µs of durability cost each, and throughput becomes bandwidth-bound (tens of thousands/sec) rather than latency-bound. Cost: up to 1 ms of added p50 latency. **That trade — a millisecond of latency for a 50× throughput multiple — is the single best deal in systems engineering**, and it's why PostgreSQL has `commit_delay` and MySQL has `binlog_group_commit_sync_delay`.

**Making recovery correct: torn writes and checksums.** A crash mid-append can leave a **partial record** — the last 300 bytes of a 500-byte record, or a 4 KB sector written and the next one not. Recovery must detect this, not parse garbage. Every record therefore carries `[length][CRC32C of payload][payload]`, and replay stops at the first record whose length runs past EOF or whose CRC fails. That truncation point *is* the durability boundary. (PostgreSQL additionally pads WAL to 8 KB pages with a per-page header; SQLite checksums each frame in its WAL.)

**Failure modes and what each costs:**

| Failure | What happens | Mitigation |
|---|---|---|
| Crash after fsync, before B-tree apply | Log has the record, B-tree doesn't | Replay from last checkpoint. **This is the normal case, not an error.** |
| Crash mid-append | Trailing partial record | CRC + length check; truncate at first bad record |
| Log grows without bound | Disk fills; recovery takes hours | **Checkpointing**: periodically fsync the B-tree, then truncate/recycle log segments before that point |
| fsync returns an error | §1.9's fsyncgate — data may be permanently gone | **Treat fsync failure as fatal.** Crash the process; do not retry and do not ack. |
| Disk full during append | `ENOSPC` mid-write, possibly a short write | Preallocate log segments at startup (also lets `fdatasync` skip metadata, §1.4) |
| Replica lags / falls behind | Failover loses acked writes | Ship log records to replicas; ack only after N replicas fsync (synchronous replication) |

**Why this over the alternatives.** *Mirror/shadow paging* (write a new copy, atomically swap a root pointer) gives atomicity without a log but amplifies writes badly and destroys locality. *`O_SYNC` on the data file* makes every write durable, which is correct and about 100× too slow. *Battery-backed write cache in a RAID controller* makes fsync nearly free and is a legitimate hardware answer — but it's a property of your hardware, not your software, and it does not survive moving to a cloud VM. The WAL is the design that is fast, correct, and portable, which is why *every* durable system converges on it: PostgreSQL WAL, MySQL InnoDB redo log, SQLite WAL mode, Kafka's partition log (where the log **is** the database), etcd/Raft's log, LSM-tree engines' commit log (RocksDB, Cassandra), and every filesystem journal underneath all of them.

**You will build this on Day 37.** Today's job is to understand *why* every line of it is there: `append` is §1.3 (page cache, sequential), `fsync` is §1.4 (the barrier and its price), group commit is §1.4's policy C, "crash on fsync error" is §1.9, and "fsync the directory too" is §1.4's four-step replace.

---

## 1.8 System design ② — log rotation and shipping for a service writing 1 GB/hour

**The problem.** A fleet of 40 API servers each writes ~1 GB/hour of structured application logs. Requirements: (a) never lose a log line, including during rotation and restart; (b) never fill the disk (each host has 50 GB free — that's 50 hours of headroom, so this *will* happen without rotation); (c) logs must reach a central store (S3/Elasticsearch/Loki) within ~1 minute; (d) log writing must never block or crash the application; (e) survive the collector being down for 2 hours.

Note this is a genuinely different problem from §1.7. The WAL optimizes for *strict durability of individual records at ack time*. Logs optimize for *never blocking the request path* and *bounded disk usage*, and they explicitly accept losing the last few unflushed lines on a hard crash. Same primitives, opposite trade-offs.

**The key decision: rotate by renaming an open descriptor, not by deleting it.**

The naive rotation is `rm app.log` at midnight. This is broken in a way that teaches the whole of §1.1: **the application still holds a descriptor to the now-nameless inode**. It keeps writing happily into a file with no path; `ls` shows nothing; `du` shows nothing; **`df` shows the disk filling anyway**, because the inode's blocks are not freed until the last descriptor closes. You have created an invisible, unreadable, unbounded file. (Recover it with `ls -l /proc/<pid>/fd` — the descriptor is right there, marked `(deleted)`, and you can copy the data back out of `/proc/<pid>/fd/N`. This is a genuinely useful ops trick.)

Two correct designs, and when to pick each:

**Design A — copytruncate-free rename + signal (the classic).**

```
  1. mv app.log app.log.1          ← rename; the app's FD still points at the same inode
  2. send SIGHUP (or SIGUSR1) to the app
  3. app's handler: close(fd); fd = open("app.log", O_WRONLY|O_CREAT|O_APPEND)
  4. compress app.log.1; ship it; delete after N days or M gigabytes
```

Between steps 1 and 3 the app writes into the renamed file — which is fine, because it's the same bytes and the shipper will pick them up. **No lines are lost**, because at no point is there a window where a write goes nowhere. This is what `logrotate` does with `postrotate ... kill -HUP`, and it's what Nginx and Postgres implement.

**Design B — never rotate in the app at all: write to stdout.** The application writes to FD 1 with **line-buffered or unbuffered output** and knows nothing about files. The supervisor (systemd/journald, Docker's json-file driver, Kubernetes' kubelet) owns the descriptor, the rotation, and the size caps. This is the twelve-factor answer and the right default for containers — with one enormous caveat you already know from §1.3: **if the app block-buffers stdout because it's a pipe, logs arrive in 8 KB bursts and the last partial buffer is lost on `kill -9`.** Set `PYTHONUNBUFFERED=1`. This single environment variable is the difference between having and not having logs for the crash you care about.

**Never blocking the request path (requirement d).** Writing a log line is a `write(2)` into the page cache: ~2 µs, effectively never blocks (§1.2's table). *But* it can block in two real situations: the disk is full (`ENOSPC` — the write fails, and naive code raises inside your request handler), and `vm.dirty_ratio` throttling (Day 4 / §1.3 — the kernel stalls a process that's dirtying pages faster than writeback drains them). Mitigations: (i) never fsync application logs — that's the whole reason logs are cheap and WALs are not; (ii) put logging behind a **bounded in-memory queue** with a background writer thread, and on queue-full **drop and count** rather than block (a dropped-lines counter you can alert on is infinitely better than a stalled API); (iii) reserve disk headroom and alert at 80%.

**Shipping (requirements c and e).** A sidecar/agent (Vector, Fluent Bit, Promtail, Filebeat) tails the files and forwards. The properties that matter:

- **Checkpoint the read position durably.** The agent stores `(inode, byte offset)` and fsyncs that checkpoint. Using the *inode number*, not the path, is what makes rotation safe — after `mv`, the agent follows the same inode to finish reading it, then picks up the new file. Path-based tailing loses the tail of every rotated file.
- **At-least-once, not exactly-once.** On restart the agent resumes from its last checkpoint and may resend a few lines. Accept duplicates and dedupe downstream with a record ID; the alternative (checkpoint before send) drops lines.
- **Backpressure with a disk buffer.** If the collector is down for 2 hours (requirement e), the agent must buffer — to *disk*, in a bounded ring, not to RAM (that's an OOM, Day 4). 2 hours × 1 GB/hour = 2 GB of buffer, so budget 3–4 GB and define what happens when it fills: **drop oldest** for logs (recent logs are more valuable), never "block the app."

**Sizing, concretely.** 1 GB/hour raw; gzip on structured JSON typically gets 8–12×, call it 10× → ~100 MB/hour shipped, ~2.4 GB/day/host, ~96 GB/day across 40 hosts. Local retention: rotate at 512 MB, keep 8 compressed files ≈ 512 MB × 8 / 10 ≈ 400 MB per host — comfortably inside 50 GB with room for the shipper's 4 GB buffer. Rotate on **size, not time**: a traffic spike at 3 a.m. must not be allowed to fill the disk just because the calendar says rotate at midnight.

**Failure modes:**

| Failure | Symptom | Mitigation |
|---|---|---|
| `rm` instead of rename-and-reopen | Disk fills, `du` and `df` disagree, no visible file | Rename + HUP, or write to stdout. Recover via `/proc/<pid>/fd` |
| `copytruncate` rotation | Lines written between copy and truncate are **lost**; offsets shift under the shipper | Avoid. Use rename + reopen. Only use copytruncate if the app truly can't reopen |
| App block-buffers stdout | No logs until crash, then partial | `PYTHONUNBUFFERED=1`, `python -u`, `flush=True`, or a real log handler |
| Collector down, unbounded memory buffer | Shipper OOMs, takes the host with it | Bounded **disk** buffer + drop-oldest policy |
| One log line is 200 MB (a stack dump of a huge object) | Rotation math breaks, shipper chokes | Truncate individual lines at the logging layer (e.g. 16 KB) |
| Shipper holds descriptors to every rotated file | `EMFILE` on the shipper (§1.1) | Cap concurrent open files; close after EOF + rotation confirmed |

**Why this over the alternatives.** *Logging directly to the network* (syslog/UDP to a collector) removes disk usage but makes the app's availability depend on the collector's, and UDP silently drops under load — you lose exactly the logs from the incident that caused the load. *Logging to a database* couples request latency to database health. *Files + tailing agent* is the design that keeps a failing collector from becoming a failing application, which is requirement (d) and the whole point.

---

## 1.9 Case studies

### ① PostgreSQL fsyncgate (2018) — when `fsync` returned success after losing your data

**What happened.** In March 2018, Craig Ringer posted to pgsql-hackers a finding that shook every database project on Linux. PostgreSQL's checkpointer relies on the standard contract: write data, call `fsync`, and if `fsync` returns 0 the data is durable; if it returns an error, retry. On Linux (and, it turned out, on several other kernels including at times FreeBSD, macOS, and OpenBSD), that contract was quietly broken in two ways:

1. **Errors were reported once, then cleared.** When the kernel's writeback failed — a transient device error, a thin-provisioned volume hitting its limit, an NFS hiccup, a USB disk yanked — it set an error flag on the inode's `address_space`. The *next* `fsync` returned `EIO`. But the flag was then **cleared**. A second `fsync` on the same file returned **success** — despite the data being permanently gone. PostgreSQL's error handling did exactly the wrong thing: it retried the fsync, got success, and concluded the checkpoint was complete. It then advanced its redo pointer and discarded WAL that was the only remaining copy of those changes.
2. **The error could be delivered to the wrong process.** Because the flag lived on the inode, a completely unrelated process that happened to `fsync` the same file first would consume the error, and PostgreSQL would never see it at all. Worse, dirty pages whose writeback had failed could be marked clean, so there was nothing left to retry even in principle.

The failure was silent: no crash, no log, no corruption warning. A database would report a successful checkpoint and continue serving, with a hole in its data.

**The engineering lessons — three, and all of them generalize.**

1. **`fsync` failure is not retryable, and must be fatal.** PostgreSQL's fix (12.0, backported to 9.4+) was to **`PANIC` — deliberately crash the whole server** — on any fsync failure, forcing crash recovery from WAL, which is the only path that reliably reconstructs state. The `data_sync_retry` GUC exists to restore the old behaviour on kernels where retry is safe, and it defaults to `off` (i.e. PANIC). Counterintuitively, **crashing is the safe behaviour** — the alternative is silently acknowledging data you no longer have. When you write durability code, encode this: `except OSError: os.abort()`, not `except OSError: retry()`.
2. **A syscall's man page describes intent, not universal behaviour.** The POSIX text was ambiguous enough that both kernel and database authors believed they were correct. Every durability guarantee you write should name a **kernel version and filesystem**, and should be validated with actual fault injection (`dm-flakey`, `dm-error`, or physically pulling power), not by reading documentation.
3. **Error state that is consumed-and-cleared is a broken design for anything shared.** The kernel's fix (Jeff Layton's `errseq_t`, merged in Linux 4.13, with Postgres-visible behaviour improving through 4.16+) gives each `struct file` a sequence-number snapshot so that **every descriptor opened before the error observes it exactly once**, rather than the first caller consuming it for everyone. Apply the same principle to your own systems: sticky, per-observer error state beats a shared flag someone can clear.

**Primary sources (verify current — kernel behaviour has changed since):** the pgsql-hackers thread "PostgreSQL's handling of fsync() errors is unsafe and risks data loss at least on XFS" (Craig Ringer, 2018-03-28); the PostgreSQL wiki page **"Fsync Errors"**; LWN.net, *"PostgreSQL's fsync() surprise"* (Jonathan Corbet, 18 April 2018); the follow-up paper *"Can Applications Recover from fsync Failures?"* (Rebello et al., USENIX ATC 2020), which found the same class of bug in ext4, XFS, SQLite, LMDB, LevelDB, and Redis; PostgreSQL release notes for 12.0 and the `data_sync_retry` documentation; Linux commit series introducing `errseq_t` in `include/linux/errseq.h`.

### ② Redis persistence — RDB vs AOF as an explicit durability-versus-throughput dial

**What it is.** Redis is an in-memory data store, so persistence is optional and configurable — which makes it the clearest teaching example of the trade-off in this whole day, because Redis *makes you choose in the config file*.

**RDB (snapshot).** Periodically, Redis `fork()`s (Day 2/Day 4: copy-on-write, so the child sees a frozen consistent view without stopping the parent) and the child serializes the entire dataset to a `.rdb` file, then atomically renames it into place (§1.4's pattern). Cheap during normal operation — the parent takes almost no I/O hit — and the snapshot is a single compact file, ideal for backups and fast restarts. **But:** everything since the last snapshot is lost on a crash. With `save 900 1`, that could be 15 minutes of writes. Additionally, the fork can double memory usage in write-heavy workloads as COW pages are copied (Day 4's Instagram case study, inverted), and on a large dataset the fork itself can pause the server for hundreds of milliseconds while page tables are copied.

**AOF (append-only file).** Every write command is appended to a log — a WAL by another name (§1.7). Recovery replays the commands. The durability dial is one config line, `appendfsync`, and it is exactly §1.4's three policies:

| `appendfsync` | Behaviour | Data at risk on power loss | Throughput impact |
|---|---|---|---|
| `always` | fsync after **every** write command | ~0 (one in-flight command) | Severe — bounded by fsync latency, hundreds to low thousands of ops/sec on typical disks |
| `everysec` (**default**) | Background fsync once per second | **up to ~1 second** of writes | Small — the common recommendation |
| `no` | Never fsync; let the kernel decide (§1.3's writeback, ~30 s) | up to ~30 s | None |

Because an AOF grows without bound (it records commands, not state), Redis periodically **rewrites** it into a compact form representing the current dataset — the checkpoint-and-truncate step from §1.7. Modern Redis (7.0+) uses a multi-part AOF: a base snapshot file plus incremental files with a manifest, which makes rewrite cheaper and safer. The recommended production configuration is **both** RDB and AOF: AOF for point-of-failure recovery, RDB for fast restarts and backups.

**The engineering lesson.** Redis's documentation is unusually honest that `everysec` means *you can lose one second of writes* — and it is the default, because for the overwhelming majority of Redis use cases (caching, sessions, rate-limit counters, queues with at-least-once semantics upstream) that is the correct trade. **The failure is not choosing `everysec`; it is not knowing you chose it.** Teams put Redis in front of a payments flow, never read the persistence page, and discover the one-second window during an outage postmortem. Any store you use has a setting like this. Find it, write the number down, and make sure the product decision matches it.

**Primary sources (verify current — Redis persistence internals changed in 7.0):** the Redis documentation page **"Redis persistence"** (redis.io/docs/latest/operate/oss_and_stack/management/persistence/), which documents `save`, `appendonly`, `appendfsync`, and the multi-part AOF; Redis `redis.conf` annotated defaults; Salvatore Sanfilippo's blog post *"Redis persistence demystified"* (2012), still the best conceptual walkthrough of the fork/COW snapshot mechanism.

**Honest gap (Principle 7).** I am not aware of a canonical, published *postmortem* for a Redis data-loss incident of the fsyncgate calibre — the Redis case is a **design** case study, not a failure case study, and I'm not going to invent an incident to balance the pair. The genuine failure case study for today is fsyncgate; the ATC 2020 paper cited above is the peer-reviewed generalization of it across many systems, including Redis.

---

## 1.10 In production (Part 1)

**Best practices, beginner → senior.**

- **Always close descriptors, and use the language's scoping tool to do it.** `with open(...) as f:` in Python, `defer f.Close()` in Go, RAII in C++. CPython's refcounting makes a leaked file *usually* close promptly, which teaches a bad habit that breaks the moment you're on PyPy, hold a reference in a list, or hit a reference cycle (Day 4 §1.6). For sockets and connection pools, closing is never automatic in practice.
- **Size `RLIMIT_NOFILE` deliberately, and know all three places it's set.** The shell (`ulimit -n`), systemd (`LimitNOFILE=` in the unit file — **systemd ignores your shell's ulimit entirely**), and containers (Docker `--ulimit nofile=65536:65536`; Kubernetes inherits from the container runtime). 1024 is a 1980s default that no network service should run with. 65536 is a reasonable starting point for a server.
- **Budget descriptors like memory.** Per process at peak: (max concurrent client connections) + (DB pool size × number of pools) + (HTTP client pool) + (log files) + (epoll instances) + (per-thread pipes/eventfds) + slack. If your server accepts 10,000 connections and holds a 100-connection DB pool, 1024 was never going to work.
- **Never fsync in a request handler unless durability is the product.** Logs, caches, temp files, metrics: no fsync. Payments, orders, ledger entries, WAL: fsync, batched (§1.4 policy C).
- **`PYTHONUNBUFFERED=1` in every container image.** Put it in the Dockerfile, not in the docs (§1.3).
- **Never block the event loop.** In an asyncio/Node/Redis-style server, any synchronous file read, `time.sleep`, CPU-heavy parse, or blocking DB driver call stalls *every* connection (§1.6's analogy break #2). Push those to a thread pool or a separate process.

**Common mistakes, in the order people make them:**

| Level | Mistake | What it looks like in production |
|---|---|---|
| Beginner | Believing `write()` = "on disk" | Data loss on power failure that nobody can explain |
| Beginner | `f.write()` without close/flush in a script | Truncated output files; "the last rows are missing" |
| Beginner | Treating `EAGAIN` as an error | Error logs flooded with harmless "resource temporarily unavailable" |
| Intermediate | Descriptor leak in an error path (`open` then an exception before `close`) | Slow-motion `EMFILE`: the service works for 6 hours, then every request fails |
| Intermediate | `rm` on a log file instead of rotate | Disk full, `du` says 2 GB, `df` says 100% |
| Intermediate | Assuming `write()` wrote everything | Silent truncation on sockets and pipes under load only |
| Intermediate | Ignoring the buffering mode change when stdout is a pipe | "Container has no logs" |
| Senior | fsync the file but not the directory | Rare, unreproducible "file vanished after power loss" |
| Senior | Retrying a failed fsync | §1.9. Silent data loss with a clean bill of health |
| Senior | Edge-triggered epoll without draining to `EAGAIN` | A small fraction of connections hang forever under load |
| Senior | One blocking call inside the event loop | p99 latency collapses; p50 looks fine; CPU is 3% |

**Monitoring and observability — what to actually put on a dashboard:**

- **Open descriptors vs limit, per process.** `ls /proc/<pid>/fd | wc -l` against `/proc/<pid>/limits`. Alert at 80%. In Prometheus: `process_open_fds / process_max_fds`. **This is the single highest-value I/O metric** because FD leaks are gradual, silent, and then total.
- **`/proc/meminfo` `Dirty:` and `Writeback:`** — your unflushed-data risk window in KB (§1.3), and an early indicator of writeback stalls.
- **fsync latency**, as a histogram, from your database or your own code. When p99 storage latency degrades (noisy neighbour, failing drive, cloud throttling), this moves before anything else.
- **`iostat -x 1`**: `await` (average I/O wait), `%util`, and queue depth. Sustained `%util` near 100 with rising `await` means the device is the bottleneck.
- **Per-process I/O**: `/proc/<pid>/io` (`read_bytes`, `write_bytes`, `cancelled_write_bytes`) tells you who is actually touching the disk.
- **`ss -s` / `ss -tan state time-wait | wc -l`** for socket counts by state — sockets are descriptors, and a `TIME_WAIT` pileup is an FD problem before it's a networking problem.
- **`lsof -p <pid>`** to find *what* is leaking once the count is climbing. `lsof | awk '{print $2}' | sort | uniq -c | sort -rn | head` finds the guilty process host-wide.

**Failure modes and recovery:**

| Failure | Detection | Recovery |
|---|---|---|
| `EMFILE` (per-process FD exhaustion) | "Too many open files" in logs; `accept()` failing while the service still runs | Raise `LimitNOFILE` **and** find the leak (`lsof -p`). Restart is a band-aid. Defensively: on `EMFILE` in an accept loop, `close()` a pre-reserved spare descriptor so you can `accept`-then-`close` and reject cleanly instead of spinning |
| `ENFILE` (system-wide) | Host-wide failures across unrelated services | `sysctl fs.file-max`; find the process holding hundreds of thousands |
| `ENOSPC` mid-write | Writes fail; often the app crashes in a weird place | Rotate/delete logs; check for deleted-but-open files via `lsof +L1`; **preallocate** critical files at startup |
| Deleted-but-open file eating disk | `df` ≫ `du` | `lsof +L1` finds it; restart the holder, or truncate via `/proc/<pid>/fd/N` |
| fsync stalls (device or cloud throttling) | p99 latency spikes; `iostat` `await` climbs | Reduce fsync rate via group commit; move WAL to a dedicated device; check cloud burst-credit exhaustion |
| Event loop stalled by a blocking call | Low CPU, high latency, all connections slow simultaneously | Profile; move the blocking call off-loop. asyncio's debug mode (`PYTHONASYNCIODEBUG=1`) logs slow callbacks |

**Scaling behaviour.** Sequential appends scale with device bandwidth (GB/s on NVMe); fsync'd commits scale with device *latency* and only improve with batching, faster hardware, or parallel independent logs. Connections scale with the event-loop model to ~100k+ per process (bounded by FDs, memory for socket buffers, and ephemeral port ranges on the client side). One epoll thread saturates around 50–200k events/sec depending on per-event work; past that, use `SO_REUSEPORT` with one accepting process per core (Nginx's model) rather than threads sharing one epoll set.

**Cost.** FDs themselves are nearly free (a pointer in an array, ~a few hundred bytes of kernel object). The real costs are (i) **socket buffers** — default `SO_RCVBUF`/`SO_SNDBUF` can be 128–256 KB each, so 10,000 idle connections can pin gigabytes of kernel memory; (ii) **fsync**, which converts directly into IOPS you pay for on cloud storage (provisioned IOPS is expensive, and burst credits run out at 3 a.m.); (iii) **log volume**, where the shipping and indexing bill routinely exceeds the compute bill for the service producing them. Cutting log verbosity is often the cheapest infrastructure saving available.

---

## 1.11 Failure modes & common misconceptions (Part 1)

- **"`write()` returned, so it's on disk."** No. It's in the page cache at best, and possibly still in your language's buffer (§1.3). Only a returned `fsync` (on a non-lying device) means durable.
- **"`flush()` makes it durable."** `flush()` moves bytes from your process's buffer into the *kernel*. It survives `kill -9`; it does not survive power loss. `flush()` then `fsync()` is the pair.
- **"I fsync'd the file, so the file is safe."** Only if its *name* is safe too. If you created or renamed it, fsync the parent directory (§1.4).
- **"fsync failed, I'll retry."** The most expensive misconception on this page (§1.9). Treat it as fatal.
- **"File descriptors are just for files."** They're for sockets, pipes, terminals, epoll instances, timerfds, eventfds, signalfds, inotify watches, and memfds. Your FD budget is consumed by all of them (§1.1, §1.2).
- **"`lsof` shows a file, so it exists."** A deleted-but-open file has no path but still holds disk space (§1.1, §1.8).
- **"Non-blocking makes I/O faster."** It makes I/O *not wait*. Without readiness notification it's strictly worse than blocking (§1.5's spin measurement).
- **"`O_NONBLOCK` gives me async file I/O."** It does nothing for regular files (§1.5). That's `io_uring` or a thread pool.
- **"`write()` writes everything I gave it."** Short writes are normal and documented on pipes and sockets (§1.5, block 4).
- **"epoll told me it's readable, so `read()` will return data."** Spurious wakeups exist. Always handle `EAGAIN` after a readiness event (§1.6).
- **"epoll is faster than select, so always use epoll."** For a handful of descriptors, `poll` is fine and simpler. epoll's win is asymptotic — it shows up at hundreds-to-thousands of *mostly idle* descriptors.
- **"Edge-triggered is just an optimization."** It's a different contract. Get it wrong and connections hang silently (§1.6).
- **"One thread per connection is always wrong."** For a few hundred connections doing heavy per-request CPU work, threads are simpler and perform fine. The event loop wins on *many mostly-idle* connections — which happens to be exactly the shape of an LLM agent server (Part 2).
- **"The page cache is wasted memory."** It's the reason your second read of a file is 1000× faster. `free -h`'s "available" column already accounts for it (Day 4).

---

## 1.12 Interview & practice questions (Part 1)

1. What is a file descriptor, precisely? Why an integer and not a pointer?
2. Explain the difference between a file descriptor, an open file description, and an inode. What does `dup()` duplicate, and what does a second `open()` create? Give a bug that depends on the distinction.
3. I call `f.write("x")` and then, from another terminal, `stat` the file — size 0. Explain every layer between the string and the disk, and say exactly what a `kill -9` versus a power cut would destroy at each layer.
4. Why is `fsync` slow? Walk through the four things it must do. What's the difference between `fsync` and `fdatasync`, and when does it matter?
5. Write the correct sequence for durably replacing a config file. Justify each of the four steps by naming the failure it prevents.
6. Your database does 100 commits/sec on a machine whose CPU is 4% busy. What's the most likely bottleneck, how would you confirm it, and what's the first fix that doesn't require new hardware?
7. What does `EAGAIN` mean and why is it not an error? What does a *short write* mean and why is it not an error?
8. Why does `O_NONBLOCK` not help with regular files? What are the two ways real systems get async file I/O?
9. Compare `select`, `poll`, and `epoll` in terms of complexity per call and capacity. Why exactly is epoll O(1) — what data structure change makes that true?
10. Explain level-triggered versus edge-triggered epoll. Describe a specific bug that only appears with edge-triggered.
11. A service works fine for six hours and then every new request fails with "Too many open files." Give three plausible causes and the exact commands you'd run to distinguish them.
12. Someone rotates logs with `rm app.log`. The app keeps running. What happens to disk usage, and how would you recover the data that's still being written?
13. Why does a Python container often show no logs until it crashes? What's the one-line fix, and which layer of the buffering stack does it change?
14. Explain group commit. What does it trade away, and by how much?
15. Your fsync returns `EIO`. What should your program do, and why is retrying wrong?
16. Design question: an event-loop server has p50 = 3 ms and p99 = 900 ms, with the CPU at 5%. What's your first hypothesis and how do you prove it?

---

# PART 2 — AGENTIC AI

> Part 1 built the machine. Part 2 puts an LLM agent on it and treats the backend as a black box — I will not re-explain descriptors, buffering, or epoll here; I'll point back at §1.x. The claim of this Part is narrow and load-bearing: **an agent server is the most I/O-bound program you will ever write**, and every property of it — its concurrency model, its cost, its failure modes, its user-visible latency — follows from Part 1's rules applied to descriptors that stay open for minutes instead of milliseconds.

## 2.1 An agent is an I/O-bound program that occasionally does arithmetic

**Depth: [CORE]** (for this Part)

### Intuition

Take one turn of a typical tool-using agent and account for the wall-clock time:

```
  agent turn (one user question, two tool calls, one final answer)
  ├── build the request, serialize JSON            ~2 ms     CPU
  ├── HTTP POST to the model provider              ~8 s      WAITING on a socket
  ├── parse the tool_use block                     ~1 ms     CPU
  ├── tool: SELECT ... FROM orders                 ~40 ms    WAITING on a socket (DB)
  ├── tool: GET https://api.example.com/...        ~250 ms   WAITING on a socket
  ├── HTTP POST to the model provider (round 2)    ~11 s     WAITING on a socket
  └── serialize the response                       ~1 ms     CPU
                                                   ─────────
                          total wall clock         ~19.3 s
                          total CPU                ~4 ms     ← 0.02%
```

**Your agent process is idle 99.98% of the time.** It is not computing; it is *holding descriptors open and waiting for bytes* — precisely the situation §1.5 and §1.6 exist for. The model inference is happening on someone else's GPUs; from your process's point of view it is indistinguishable from a slow file download.

This single fact determines the entire architecture, and it inverts the intuition most backend engineers bring from CPU-bound services:

- **Concurrency is nearly free, and you need a lot of it.** With an event loop, one process can hold thousands of in-flight agent turns, because each one costs a socket and a few KB of state, not a CPU. Sizing workers by CPU cores (`workers = 2 × cores + 1`, the Gunicorn folklore) is the *wrong* model here and will leave your machine 99% idle while requests queue.
- **Threads-per-request survives longer than you'd think, then falls off a cliff.** 200 concurrent agent turns on 200 threads works (Day 3). 5,000 does not — 5,000 × 8 MB of stack virtual address space plus scheduler pressure (Day 3, Day 4).
- **One blocking call poisons everything** (§1.6's analogy break #2). A synchronous `requests.get()` or a blocking DB driver inside an `async def` handler parks the *event loop thread* for 250 ms, during which none of the other 3,000 in-flight turns can make progress. In a CPU-bound service you'd never notice; here it's the top production incident.
- **Timeouts are load-bearing, not hygiene.** A model call can legitimately take minutes at high effort (Day 63 territory). A hung socket with no timeout holds a descriptor **forever**, and enough of them is §1.1's `EMFILE` — with the added insult that your service now looks healthy (low CPU) while accepting nothing.

**What came before / why this is new.** Classic web backends are a few-milliseconds-per-request affair where the dominant cost is CPU or a fast database round trip, and where "a request" is over before a human notices. Agent workloads broke that: a single logical request lives for **10 seconds to 10 minutes**, makes several nested network calls, and streams its output incrementally. The closest prior art is video streaming and long-polling chat — which is exactly why the industry reached for **Server-Sent Events** (§2.2), a technology from 2006, the moment agents shipped.

### Analogy — a switchboard operator, not a factory worker

A factory worker's throughput is bounded by how fast their hands move: more output needs more workers. A **switchboard operator** connects calls and then waits; one operator can hold hundreds of open lines because holding a line costs nothing. Adding operators doesn't help when the bottleneck is that the other party is still talking.

**Where the analogy breaks — two ways:**

1. **The operator has a physical limit on plugs; you have `RLIMIT_NOFILE`, and you can raise it.** But there is a real ceiling: every held connection also pins kernel socket buffers (§1.10's cost note) and per-request Python objects — including, critically, the **conversation history**, which grows every turn (Day 4 §2.3's agent memory leak). 5,000 held connections × 200 KB of accumulated messages = 1 GB of RSS that the OOM killer is very interested in.
2. **The operator knows when a call ends.** Your process often doesn't: the peer's network dies, no FIN arrives, and the socket sits in `ESTABLISHED` forever with no traffic. That is why **you** must impose timeouts and TCP keepalives; nothing else will.

### Worked example — where an agent's time and money actually go

For a single agent turn with ~12,000 input tokens and ~800 output tokens, on `claude-opus-5` ($5/MTok input, $25/MTok output — **verify current pricing**):

| Resource | Amount | Notes |
|---|---|---|
| Wall clock | ~19 s | dominated by two model round trips |
| **CPU on your server** | **~4 ms** | 0.02% of the wall time |
| Descriptors held, peak | 3 | client socket + provider socket + DB socket |
| Descriptor-seconds | ~19 + ~19 + ~0.04 ≈ **38 fd·s** | *this* is the resource you're actually spending |
| Tokens | 12,000 in / 800 out | ≈ $0.06 + $0.02 = **$0.08** |

Note the unit in bold. For a normal web service you provision on CPU-seconds; **for an agent service you provision on descriptor-seconds and tokens.** A machine that can hold 5,000 concurrent descriptors for 20 s each delivers 5,000 turns per 20 s ≈ 250 turns/sec, on a CPU that is almost entirely idle. If you provisioned that same box by CPU utilization you would run 8 workers and serve 8 concurrent turns.

### Runnable example — proving an agent turn is 99.9% waiting

```python
# agent_is_io_bound.py — any OS. stdlib only (simulates the network waits; no API key needed).
# Run:  python3 agent_is_io_bound.py
import time, json, hashlib

def fake_model_call(prompt: str, latency_s: float) -> str:
    """Stand-in for an HTTPS round trip to a model provider (§1.5: the thread WAITS)."""
    time.sleep(latency_s)                       # blocking wait on a socket, simulated
    return json.dumps({"text": "ok", "n": len(prompt)})

def fake_tool_call(name: str, latency_s: float) -> str:
    time.sleep(latency_s)
    return json.dumps({"tool": name, "rows": 3})

def real_cpu_work(payload: str) -> str:
    """The only genuinely CPU-bound thing an agent loop does: serialize/parse/hash."""
    h = hashlib.sha256(payload.encode()).hexdigest()
    return json.dumps({"h": h})

history = [{"role": "user", "content": "How many orders shipped late last week?"}]
prompt = json.dumps(history)

wall0, cpu0 = time.perf_counter(), time.process_time()
steps = []

def step(label, fn):
    w0, c0 = time.perf_counter(), time.process_time()
    fn()
    steps.append((label, time.perf_counter() - w0, time.process_time() - c0))

step("build request (CPU)",        lambda: real_cpu_work(prompt))
step("model call #1 (network)",    lambda: fake_model_call(prompt, 8.0))
step("parse tool_use (CPU)",       lambda: real_cpu_work(prompt))
step("tool: database (network)",   lambda: fake_tool_call("sql", 0.040))
step("tool: http api (network)",   lambda: fake_tool_call("http", 0.250))
step("model call #2 (network)",    lambda: fake_model_call(prompt, 11.0))
step("serialize reply (CPU)",      lambda: real_cpu_work(prompt))

wall, cpu = time.perf_counter() - wall0, time.process_time() - cpu0

print(f"{'step':<28}{'wall (ms)':>12}{'cpu (ms)':>11}")
print("-" * 51)
for label, w, c in steps:
    print(f"{label:<28}{w*1000:>12.1f}{c*1000:>11.3f}")
print("-" * 51)
print(f"{'TOTAL':<28}{wall*1000:>12.1f}{cpu*1000:>11.3f}")
print(f"\n  CPU utilization of this 'agent turn': {cpu/wall*100:.3f}%")
print(f"  descriptor-seconds spent waiting     : ~{wall:.1f} fd·s per concurrent socket")
print("\n  -> This process is a switchboard, not a factory. Size it by concurrency (§1.6),")
print("     not by CPU cores. One blocking call here stalls every other in-flight turn.")

# --- actual output ---
# step                           wall (ms)   cpu (ms)
# ---------------------------------------------------
# build request (CPU)                  0.0      0.028
# model call #1 (network)           8001.2      0.058
# parse tool_use (CPU)                 0.0      0.021
# tool: database (network)            40.4      0.019
# tool: http api (network)           250.7      0.021
# model call #2 (network)          11001.4      0.041
# serialize reply (CPU)                0.0      0.022
# ---------------------------------------------------
# TOTAL                            19293.7      0.210
#
#   CPU utilization of this 'agent turn': 0.001%
#   descriptor-seconds spent waiting     : ~19.3 fd·s per concurrent socket
```

**Why this works, line by line.**

- `time.sleep()` is an honest simulation of a blocking network wait for the purpose of *this* measurement: like `recv()` on a socket with no data, it parks the thread without consuming CPU (§1.5's block 1 measured exactly this property). It is **not** a simulation of the failure modes — a real socket can hang forever, return partial data, or reset; `sleep` always returns on schedule. That's the demo's honest limit.
- **`time.process_time()` vs `time.perf_counter()` is the whole experiment** (§1.5). CPU time counts only cycles this process burned. 0.21 ms of CPU against 19,294 ms of wall clock is the number that should reshape how you provision an agent service.
- The `real_cpu_work` calls are deliberately real (SHA-256 over the serialized history) so the CPU column isn't fabricated zeros — it shows genuine JSON+hash work landing in the tens of microseconds.
- The steps are ordered as a real tool-use loop: request → model → detect `stop_reason == "tool_use"` → run tools in *your* code → send results back → model. That loop shape is provider-independent (only field names differ); the point here is that **every arrow in it is a socket wait.**

---

## 2.2 Streaming — one descriptor held open for the length of a thought

**Depth: [WORKING]**

### Intuition

A non-streaming agent endpoint makes the user stare at a spinner for 19 seconds and then dumps a wall of text. Streaming turns that into first-token-in-800 ms and a steadily filling response. Same total time, radically different perceived latency — and it's the reason every chat product streams.

Mechanically, streaming means: **the HTTP response never ends until the answer does.** The server sends headers immediately, then writes chunks as tokens arrive, holding the client's socket open the whole time. The dominant wire format is **Server-Sent Events** (SSE): `Content-Type: text/event-stream`, and a body of `data: <payload>\n\n` frames. (WebSockets are the alternative; SSE won for LLM output because it's one-directional, works over plain HTTP/1.1 and HTTP/2, survives proxies better, and needs no special client.)

Now connect it to Part 1. Streaming means **your server holds one client descriptor open per active user for the entire duration of a generation**, while simultaneously holding one upstream descriptor to the model provider. It is §2.1's descriptor-seconds problem doubled, and it makes three of Part 1's facts operationally urgent:

1. **The buffering stack (§1.3) is now a correctness issue, not a performance one.** Anything between your `write()` and the user's browser that buffers will hold the tokens and release them in a lump — turning your beautifully streamed response back into a 19-second spinner. The culprits, in order of frequency: a reverse proxy with response buffering on (Nginx's `proxy_buffering` is **on by default**), your web framework's own response buffering, gzip compression middleware (which must buffer to compress), and a CDN.
2. **You need the event-loop model (§1.6).** 5,000 concurrent streams on threads is 5,000 parked threads. On an event loop it's 5,000 registered descriptors and a handful of KB each.
3. **Disconnects must be detected and must cancel upstream work** (§1.5, §1.10). If the user closes the tab, your server should notice the socket closing and abort the model request — otherwise you keep paying for tokens nobody will read, and you hold two descriptors for the rest of the generation.

### Analogy — a live phone call versus a letter

A non-streaming request is a letter: you write the whole thing, seal it, send it once. Streaming is a phone call: the line is open, words flow as they're spoken, and both parties can tell when the other hangs up.

**Where the analogy breaks — two ways, and both are production bugs:**

1. **There is no dial tone.** On a real call, silence tells you something's wrong. On an idle TCP connection, silence is indistinguishable from a working connection with nothing to say — and from a peer whose laptop went into a tunnel. That's why SSE implementations send **heartbeat/comment frames** (`: keep-alive\n\n`) every 15–30 seconds: to keep proxies from timing the connection out and to detect dead peers.
2. **On a phone line, nobody in the middle can hold your words and deliver them all at the end.** On HTTP, several intermediaries will do exactly that unless told not to (see the case study in §2.6). The analogy's "live" property is a *default* on phones and something you must actively defend on HTTP.

### Runnable example — an SSE endpoint, the curl transcript, and the buffering trap

```python
# sse_server.py — any OS. FastAPI + uvicorn.
#   pip install "fastapi>=0.110" "uvicorn>=0.29"
#   uvicorn sse_server:app --port 8000
#
# Drive it with:   curl -N http://127.0.0.1:8000/chat
#   (-N is REQUIRED: without it curl buffers and you see the whole thing at the end —
#    a client-side reproduction of the exact bug this section is about.)
import asyncio, json, time
from fastapi import FastAPI, Request
from fastapi.responses import StreamingResponse

app = FastAPI()

FAKE_TOKENS = ["Late ", "orders ", "last ", "week: ", "**14**", " of ", "312 ", "shipments."]

async def token_stream(request: Request):
    """Yield SSE frames. Each yield is one write() to the client's socket (§1.2)."""
    t0 = time.perf_counter()
    yield f"event: start\ndata: {json.dumps({'model': 'claude-opus-5'})}\n\n"
    try:
        for i, tok in enumerate(FAKE_TOKENS):
            # await = the event loop is free to serve other connections here (§1.6)
            await asyncio.sleep(0.4)            # stand-in for waiting on the provider socket

            # THE disconnect check: if the user closed the tab, stop paying for tokens.
            if await request.is_disconnected():
                print("  [server] client disconnected -> aborting upstream generation")
                return

            yield f"data: {json.dumps({'i': i, 'text': tok})}\n\n"

        elapsed = time.perf_counter() - t0
        yield f"event: done\ndata: {json.dumps({'seconds': round(elapsed, 2)})}\n\n"
    finally:
        # Runs on normal completion AND on client disconnect — release the upstream fd here.
        print("  [server] stream closed; upstream descriptor released")

@app.get("/chat")
async def chat(request: Request):
    return StreamingResponse(
        token_stream(request),
        media_type="text/event-stream",
        headers={
            "Cache-Control": "no-cache",     # don't let a cache hold the whole body
            "Connection": "keep-alive",
            "X-Accel-Buffering": "no",       # <-- tells Nginx: do NOT buffer this response
        },
    )

# ---------------------------------------------------------------------------
# $ curl -N -i http://127.0.0.1:8000/chat
# HTTP/1.1 200 OK
# date: Fri, 07 Aug 2026 09:14:02 GMT
# server: uvicorn
# cache-control: no-cache
# connection: keep-alive
# x-accel-buffering: no
# content-type: text/event-stream; charset=utf-8
# transfer-encoding: chunked            <-- no Content-Length: the body has no known size
#
# event: start
# data: {"model": "claude-opus-5"}
#
# data: {"i": 0, "text": "Late "}       <-- arrives at t≈0.4s
#
# data: {"i": 1, "text": "orders "}     <-- t≈0.8s ... one frame every 400 ms, LIVE
#
# ... (frames 2..7) ...
#
# event: done
# data: {"seconds": 3.21}
#
# ---------------------------------------------------------------------------
# $ curl http://127.0.0.1:8000/chat          # WITHOUT -N
# (nothing for 3.2 seconds, then the entire body at once)
#   ^ curl buffered it. Same bytes, same server, ruined UX. This is exactly what a
#     reverse proxy with response buffering on does to your users (§2.6).
#
# $ curl -N http://127.0.0.1:8000/chat       # then press Ctrl-C after 1 second
# data: {"i": 0, "text": "Late "}
# data: {"i": 1, "text": "orders "}
# ^C
#   server log:
#     [server] client disconnected -> aborting upstream generation
#     [server] stream closed; upstream descriptor released
# ---------------------------------------------------------------------------
```

**Why this works, line by line.**

- **`media_type="text/event-stream"` plus `transfer-encoding: chunked`.** There is no `Content-Length` because the length is unknown when the headers are sent — that's the defining property of a streamed response, and it's why every layer in the path must be willing to forward bytes it hasn't finished receiving.
- **Each `yield` becomes a `write()` on the client's socket** (§1.2's uniform interface). The `\n\n` terminator is the SSE frame delimiter and is not optional — a frame without a blank line is buffered by the client waiting for more.
- **`await asyncio.sleep(...)` is the load-bearing line for concurrency.** `await` returns control to the event loop, which goes back to `epoll_wait` (§1.6) and serves other connections. Replace it with a synchronous `time.sleep(0.4)` and *every* concurrent stream stalls for 400 ms per token — the §1.6 analogy-break, reproduced in eight characters. This is the number-one agent-server bug.
- **`X-Accel-Buffering: no` is the Nginx-specific escape hatch** (§2.6): it tells Nginx to disable `proxy_buffering` for this response only. It is a response *header* your app sets, which makes it deployable without touching the proxy config — and it does nothing on other proxies, so it's a mitigation, not a guarantee.
- **`request.is_disconnected()` + the `finally:` block are the money-saving pair.** Without them a closed tab leaves you streaming into a dead socket and, worse, still paying the provider for tokens. Note the `finally` runs on both paths — that's where you cancel the upstream request and close its descriptor (§1.10's "always close").
- **Honesty caveat (Principle 6):** `request.is_disconnected()` in Starlette/FastAPI is best-effort — it reports a disconnect the ASGI server has *noticed*, and a client that vanishes without sending a FIN (network partition, laptop lid closed) may not be noticed until a TCP timeout, which can be minutes. Belt and braces: also set a hard wall-clock cap on generation, and enable TCP keepalives. Second caveat: `curl` without `-N` shows the buffered behaviour — that's a client-side artifact, deliberately included because it is the cheapest possible demonstration of the failure mode.

### Runnable example — the real thing: streaming from Claude, and counting tokens before you send

```python
# claude_stream.py — any OS.
#   pip install anthropic
#   export ANTHROPIC_API_KEY=...        (or: ant auth login)
#   python3 claude_stream.py
#
# Requires a real API key and will spend a small amount of money.
import time
import anthropic

client = anthropic.Anthropic()          # reads ANTHROPIC_API_KEY / an `ant auth login` profile

MESSAGES = [{"role": "user", "content":
             "In two sentences, explain why fsync is slow."}]

# ---------- 1) count tokens BEFORE sending: budget the request (§2.4 / Day 4 §2.1) ----------
count = client.messages.count_tokens(
    model="claude-opus-5",
    messages=MESSAGES,
)
print(f"input tokens (measured by the API, not estimated): {count.input_tokens}")

# ---------- 2) stream: first token fast, socket held open until done ----------
t0 = time.perf_counter()
first_token_at = None
chunks = 0

with client.messages.stream(
    model="claude-opus-5",
    max_tokens=1024,
    messages=MESSAGES,
    # Adaptive thinking is the current mechanism; there is no budget_tokens on this model.
    thinking={"type": "adaptive", "display": "summarized"},
    output_config={"effort": "low"},          # low|medium|high|xhigh|max
) as stream:
    for text in stream.text_stream:           # each iteration = bytes off the socket
        if first_token_at is None:
            first_token_at = time.perf_counter() - t0
        chunks += 1
        print(text, end="", flush=True)       # flush=True: §1.3, or you buffer your own output
    final = stream.get_final_message()

total = time.perf_counter() - t0
print(f"\n\n  time to first token : {first_token_at*1000:.0f} ms")
print(f"  total wall clock    : {total:.2f} s")
print(f"  text chunks received: {chunks}")
print(f"  usage               : in={final.usage.input_tokens} out={final.usage.output_tokens}")
print(f"  stop_reason         : {final.stop_reason}")
print(f"\n  -> the socket to the provider was held open for {total:.1f}s to deliver "
      f"{final.usage.output_tokens} tokens: ~{total/max(final.usage.output_tokens,1)*1000:.0f} ms/token")

# --- representative output (exact text and timings vary per call) ---
# input tokens (measured by the API, not estimated): 21
# fsync is slow because it must force dirty pages out of the kernel's page cache to the
# storage device and then issue a cache-flush barrier that the drive must acknowledge...
#
#   time to first token : 742 ms
#   total wall clock    : 3.18 s
#   text chunks received: 47
#   usage               : in=21 out=63
#   stop_reason         : end_turn
#
#   -> the socket to the provider was held open for 3.2s to deliver 63 tokens: ~50 ms/token
```

**Why this works, line by line.**

- `anthropic.Anthropic()` with no arguments resolves credentials from the environment (`ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN`, or a stored `ant auth login` profile) — don't hardcode a key.
- **`client.messages.count_tokens(...)` is the only correct way to count tokens for Claude.** `tiktoken` is OpenAI's tokenizer and undercounts Claude by roughly 15–20% on prose and far more on code — an estimate you cannot budget against. This call is how you enforce §2.4's per-request token budget *before* paying for a request that will be rejected or truncated.
- **`client.messages.stream(...)` as a context manager** is the SDK helper: it opens the HTTPS connection, parses the SSE frames the API sends (`message_start`, `content_block_delta`, `message_delta`, `message_stop`), and exposes the assembled text via `text_stream`. Each iteration corresponds to bytes arriving on that socket — the same `read()`-until-`EAGAIN` machinery from §1.5/§1.6, wrapped in `httpx`.
- **`flush=True` on the print is §1.3, applied to your own process.** Without it, if your stdout is a pipe (a container!), your beautifully streamed tokens sit in Python's 8 KB buffer and appear all at once — you'd have reproduced the very bug you're avoiding upstream.
- **Honesty caveats (Principle 6), all real:**
  - **`temperature`, `top_p`, `top_k`, and `budget_tokens` are removed on current Claude models and return a `400 BadRequestError`.** Don't add them "to be safe" — steer with prompting and `output_config.effort` instead. This is a genuine divergence from what most tutorials show.
  - Thinking blocks stream with **empty text by default** (`display` defaults to `"omitted"`), which in a UI looks like a long pause before output starts. `display: "summarized"` is set explicitly above for that reason.
  - `stream.get_final_message()` must be called **inside** the `with` block, before the connection closes.
  - Model IDs, parameters, and pricing drift — **verify against the current provider docs before relying on any of this.**
- **Provider-neutral note.** The shape is identical across vendors and only the field names change: OpenAI's Responses API streams with `client.responses.stream(...)` and counts tokens with `tiktoken`; Google's `genai` streams with `generate_content_stream` and counts with `client.models.count_tokens`. What does *not* change is the systems fact this section is about: **one long-lived descriptor per stream, held for the length of the generation, with a buffering hazard at every hop.**

---

## 2.3 Durability for agents — the run log, and "checkpoint before you ack"

**Depth: [WORKING]**

### Intuition

An agent turn is expensive (§2.1: ~$0.08 and 19 seconds) and *not idempotent* — its tool calls may have sent an email, charged a card, or opened a ticket. So the question §1.7 asked about orders applies with more force to agents: **if the worker process dies mid-turn, what exactly have you lost, and what will you wrongly redo?**

Three levels of answer, mapping directly onto §1.4's policies:

| Design | On crash you lose | On restart you might | Fits |
|---|---|---|---|
| **Nothing persisted** (state in RAM) | the whole conversation | start over from the user's message, re-running every tool call | Toy demos only |
| **Append to a run log, no fsync** | the last ~30 s (page cache, §1.3) | re-run the last few steps | Most agents. Correct trade. |
| **Append + fsync before acting** | nothing | resume exactly at the boundary | Agents whose tools have side effects |

The pattern for the third row *is* §1.7's WAL, relabelled: **before you execute a tool call with an external side effect, append a record describing it, fsync, and only then execute.** After the tool returns, append the result. On restart, replay the log; if you find a "about to call `charge_card`" record with no matching result, you know a charge may or may not have happened, and you can reconcile deliberately (query the payment provider by idempotency key) rather than blindly retrying or blindly skipping.

That framing gives you the rule that separates a demo from a system: **the durability boundary must sit before the side effect, not after it.** And the thing that makes reconciliation possible at all is an **idempotency key** generated *before* the fsync and passed to the external API — so a retry is provably the same operation.

### Analogy — a surgeon's checklist, written down before each step

A surgical team records each step *before* performing it. If the team is interrupted, the record says what was about to happen, so the next team can check rather than guess. It would be useless to write the record afterwards — that's exactly the case where the record is missing precisely when it's needed.

**Where the analogy breaks:** the checklist can't tell you whether the interrupted step actually completed; neither can your log. What closes the gap is the **external system's** ability to answer "did you already see idempotency key X?" A log gives you the *question*; the idempotency key gives you the *answer*. Design both, or the log is a diary of ambiguity. Second break: surgeons check the record once, at handover. An agent replays its log on every restart, so the log must be *machine-replayable* — self-describing records with CRCs and lengths (§1.7), not prose.

### Runnable example — an agent run log that survives `kill -9`, and resumes exactly

```python
# agent_run_log.py — Linux/any OS. stdlib only (no LLM needed).
# Run:  python3 agent_run_log.py           <- first run, crashes deliberately mid-turn
#       python3 agent_run_log.py           <- second run, RESUMES from the log
import json, os, sys, zlib, time

LOG = "/tmp/day5_agent_run.log"

# ---------------- the log: length-prefixed, CRC'd records (§1.7) ----------------
def append(fd, record: dict, durable: bool):
    payload = json.dumps(record, sort_keys=True).encode()
    frame = len(payload).to_bytes(4, "big") + zlib.crc32(payload).to_bytes(4, "big") + payload
    os.write(fd, frame)                       # -> page cache (§1.3)
    if durable:
        os.fsync(fd)                          # -> device (§1.4). THE boundary.

def replay(path):
    """Read records until EOF or the first torn/corrupt one. That point IS the boundary."""
    records, torn = [], False
    if not os.path.exists(path):
        return records, torn
    with open(path, "rb") as f:
        blob = f.read()
    off = 0
    while off + 8 <= len(blob):
        n = int.from_bytes(blob[off:off+4], "big")
        crc = int.from_bytes(blob[off+4:off+8], "big")
        body = blob[off+8:off+8+n]
        if len(body) != n or zlib.crc32(body) != crc:
            torn = True                       # partial write from a crash mid-append
            break
        records.append(json.loads(body))
        off += 8 + n
    return records, torn

# ---------------- the "agent" ----------------
STEPS = [
    ("think",        {"side_effect": False}),
    ("sql_query",    {"side_effect": False}),
    ("charge_card",  {"side_effect": True}),   # <- must be durable BEFORE it runs
    ("send_email",   {"side_effect": True}),
    ("final_answer", {"side_effect": False}),
]

def run_tool(name):
    time.sleep(0.05)
    return {"ok": True, "tool": name}

done, torn = replay(LOG)
completed = {r["step"] for r in done if r["phase"] == "result"}
pending   = [r for r in done if r["phase"] == "intent" and r["step"] not in completed]

print(f"=== startup: replayed {len(done)} records from {LOG} ===")
print(f"  completed steps : {sorted(completed) or '(none)'}")
if torn:
    print("  torn trailing record detected -> truncated at the last valid boundary")
if pending:
    p = pending[0]
    print(f"  ⚠ AMBIGUOUS: '{p['step']}' was durably intended but has no result.")
    print(f"    idempotency_key={p['idempotency_key']}")
    print(f"    -> RECONCILE with the external system by this key. Do NOT blindly retry.")

fd = os.open(LOG, os.O_WRONLY | os.O_CREAT | os.O_APPEND, 0o644)
crash_after = os.environ.get("CRASH_AFTER", "charge_card")

for step, meta in STEPS:
    if step in completed:
        print(f"  skip  {step:<13} (already done in a previous run)")
        continue
    if pending and pending[0]["step"] == step:
        print(f"  skip  {step:<13} (pending reconciliation, not auto-retried)")
        continue

    key = f"run1-{step}"
    append(fd, {"phase": "intent", "step": step, "idempotency_key": key},
           durable=meta["side_effect"])       # <-- fsync ONLY before side effects
    marker = "fsync'd" if meta["side_effect"] else "buffered"
    print(f"  intent {step:<13} ({marker})")

    if step == crash_after and "--resumed" not in sys.argv and not done:
        print(f"\n  💥 simulating kill -9 immediately after the durable intent for '{step}'")
        print(f"     (the intent survives; the result never happened)")
        os._exit(137)                          # like SIGKILL: no flush, no cleanup (Day 2)

    result = run_tool(step)
    append(fd, {"phase": "result", "step": step, "result": result},
           durable=meta["side_effect"])
    print(f"  result {step:<13} {result}")

os.close(fd)
print("\n=== turn complete ===")
print(f"  (delete {LOG} to start over)")

# --- actual output, FIRST run ---
# === startup: replayed 0 records from /tmp/day5_agent_run.log ===
#   completed steps : (none)
#   intent think         (buffered)
#   result think         {'ok': True, 'tool': 'think'}
#   intent sql_query     (buffered)
#   result sql_query     {'ok': True, 'tool': 'sql_query'}
#   intent charge_card   (fsync'd)
#
#   💥 simulating kill -9 immediately after the durable intent for 'charge_card'
#      (the intent survives; the result never happened)
#
# --- actual output, SECOND run (same command) ---
# === startup: replayed 5 records from /tmp/day5_agent_run.log ===
#   completed steps : ['sql_query', 'think']
#   ⚠ AMBIGUOUS: 'charge_card' was durably intended but has no result.
#     idempotency_key=run1-charge_card
#     -> RECONCILE with the external system by this key. Do NOT blindly retry.
#   skip  think         (already done in a previous run)
#   skip  sql_query     (already done in a previous run)
#   skip  charge_card   (pending reconciliation, not auto-retried)
#   intent send_email    (fsync'd)
#   result send_email    {'ok': True, 'tool': 'send_email'}
#   intent final_answer  (buffered)
#   result final_answer  {'ok': True, 'tool': 'final_answer'}
#
# === turn complete ===
```

**Why this works, line by line.**

- **The record frame is `[4-byte length][4-byte CRC32][payload]` — §1.7's torn-write defence, verbatim.** `replay` stops at the first frame whose body is short or whose CRC fails, which is exactly how a real WAL finds its durability boundary after a crash.
- **`durable=meta["side_effect"]` is the design decision made explicit.** Pure-thought steps are appended to the page cache and cost ~2 µs; steps with external consequences pay the ~1 ms fsync. This is §1.4's policy table applied per-record rather than per-file, and it's why the agent isn't uniformly slow.
- **`os._exit(137)` is the honest crash.** It bypasses `atexit`, `finally` blocks, and buffer flushing — the same semantics as `SIGKILL` (Day 2 §termination). The *only* reason the `charge_card` intent survives is that we fsync'd it; the two earlier buffered records survive only because the page cache outlives the process (§1.3 — they'd be gone on a power cut, which is the correct trade for pure-thought steps).
- **The second run's `⚠ AMBIGUOUS` branch is the whole point.** A durable intent with no result means "this may or may not have happened." The system does not guess. It surfaces an idempotency key and defers to the external system of record. Blind retry double-charges; blind skip silently drops a charge; **reconciliation by key is the only correct third option**, and it's only available because the key was written down *before* the call.
- **Honest limits of this demo:** it's single-writer with no concurrency control (a real agent server needs one log per run, or per-run offsets in a shared log); it never truncates or checkpoints, so the log grows forever (§1.7's checkpointing step); and it doesn't fsync the containing directory on first creation (§1.4's step 4). Those are exercises, and two of them are in "Build This."

---

## 2.4 The descriptor and token budget of an agent server

**Depth: [WORKING]**

### Intuition

Two budgets govern an agent server, and both are the same *shape* of problem as Day 4's memory budget: a hard ceiling, a per-unit cost, and a leak that grows quietly until it doesn't.

**Budget 1 — descriptors.** Per in-flight agent turn, at peak:

| Descriptor | Held for | Notes |
|---|---|---|
| Client connection (SSE stream) | whole turn (~20 s) | §2.2 |
| Upstream connection to the model provider | whole turn | Pooled by `httpx`, but one in-flight request pins one |
| Database connection | milliseconds, from a pool | Pool size is a fixed FD cost, not per-request |
| Per-tool HTTP connections | seconds | Pooled; **cap the pool or it becomes per-request** |
| Vector store / cache / queue connections | pool | Fixed cost |

So ≈ **2 long-lived descriptors per concurrent turn**, plus a fixed pool overhead. 5,000 concurrent turns ⇒ ~10,000 descriptors ⇒ your `RLIMIT_NOFILE` of 1024 fails at ~500 users, and the failure mode (§1.1) is `EMFILE` on `accept()` — the service stops accepting while reporting 3% CPU and looking perfectly healthy. Set `LimitNOFILE=65536` and compute the number rather than guessing.

**Budget 2 — tokens.** Day 4 §2.1/§2.3 established the context window as a RAM-like budget and unbounded history as the agent equivalent of a memory leak. Today adds the I/O consequence: **every token in your history is re-sent over a socket on every turn.** A conversation that grows to 100k tokens doesn't just approach the window limit and cost more; it makes each turn's *request body* larger, its time-to-first-token longer, and its bandwidth bill real. The two budgets multiply: long conversations hold descriptors longer, and holding descriptors longer means more concurrent conversations, which means more descriptors.

**The three controls, all of which you must set explicitly:**

1. **A hard concurrency cap** (a semaphore) sized to `(RLIMIT_NOFILE − pool overhead − slack) / 2`. Reject or queue beyond it, with a fast 429 — a rejected request is vastly better than an `EMFILE` that breaks every request.
2. **A wall-clock timeout on every upstream call**, and a total turn deadline. No exceptions. The default `httpx`/SDK timeout is generous (the Anthropic SDK defaults to 10 minutes), which is right for a long generation and catastrophic as a leak bound.
3. **A token budget per turn**, measured with `count_tokens` before sending (§2.2's example), with history trimmed or compacted when it's exceeded.

### Worked example — sizing a box

Target: 3,000 concurrent agent turns per process.

```
  descriptors:
      3,000 client SSE sockets
    + 3,000 in-flight upstream requests
    +   100 DB pool
    +   200 tool HTTP pool
    +    50 misc (log files, epoll, eventfds, metrics scrape)
    ─────────
      6,350 descriptors at peak   → set LimitNOFILE=16384 (2.5× headroom)

  memory (Day 4):
      3,000 × (socket buffers ~64 KB + per-request Python state ~80 KB
               + accumulated history ~150 KB)
    ≈ 3,000 × ~300 KB ≈ 900 MB      → container limit ≥ 2 GB

  CPU (§2.1):
      3,000 turns × ~4 ms CPU / ~20 s wall ≈ 0.6 cores   → 2 cores is plenty

  tokens/cost:
      3,000 concurrent × (1 turn / 20 s) = 150 turns/s × $0.08 ≈ $12/second
                                                          ≈ $43,000/hour
```

Read that last line twice. **The binding constraint on an agent service is almost never the machine.** It is the token bill, and the second constraint is the provider's rate limit. Descriptors and memory are what make the box *fall over*; tokens are what make the business fall over. Size the box correctly and then put a budget guard in front of it.

---

## 2.5 System design — an agent gateway for 5,000 concurrent streaming users

**The problem.** Build the service that sits between a chat UI and a model provider. Requirements: (a) 5,000 concurrent streaming conversations; (b) time-to-first-token under 1 s at p95; (c) a user closing the tab must stop the spend within a second; (d) a provider outage or rate limit must degrade gracefully, not cascade; (e) per-user and per-org spend caps enforced *before* the request; (f) full transcripts durable enough to answer "what did the agent do?" a week later.

**Key decision 1 — event loop, one process per core, hard concurrency cap.** (§1.6, §2.1, §2.4.) Async Python (or Go/Node) with `SO_REUSEPORT` and N processes = N cores. Each process holds a semaphore sized from its FD limit. Beyond it: **429 with `Retry-After` immediately** — never queue unboundedly, because a queue of long-lived requests is a descriptor leak with extra steps. Do not size workers by CPU; the CPU is idle (§2.1).

**Key decision 2 — never buffer the stream, and prove it.** (§1.3, §2.2, §2.6.) `proxy_buffering off` at the reverse proxy (or `X-Accel-Buffering: no` per response), gzip **disabled** for `text/event-stream`, no CDN in the path for streaming routes, and `PYTHONUNBUFFERED=1`. Add a synthetic check that asserts the *inter-frame arrival time* through the full production path, not just that the response eventually arrives — buffering is invisible to a check that only measures total latency.

**Key decision 3 — the ack boundary is "durably logged," not "sent to the model."** (§1.7, §2.3.) Per turn: append `{turn_id, user, org, prompt_hash, tools_enabled}` to a run log, fsync (group-committed across concurrent turns, §1.4 policy C), and only then call the provider. Tool calls with side effects get their own intent+result records with idempotency keys. Transcripts ship to object storage via the §1.8 pipeline. This satisfies (f) and makes (c) auditable.

**Key decision 4 — cancellation propagates both directions.** (§2.2.) Client disconnect → cancel the upstream request (close its descriptor) → release the semaphore → record a `cancelled` run-log entry with tokens-so-far. Conversely, an upstream failure → close the client stream with a terminal SSE error frame, not a silent hang. Every long-lived descriptor needs an owner that will close it on *every* path.

**Key decision 5 — spend caps in front of the request, using measured tokens.** (§2.4.) `count_tokens` on the assembled request, check it against the org's remaining budget in Redis, decrement optimistically, reconcile with actual usage from the response. Enforcing *after* the call means you find out you're over budget by reading the invoice.

**Failure modes:**

| Failure | Symptom | Mitigation |
|---|---|---|
| `EMFILE` at ~500 users | `accept()` fails, service looks idle and healthy | `LimitNOFILE=16384`; concurrency semaphore; alert on `process_open_fds/process_max_fds` (§1.10) |
| A blocking call in a handler | p99 collapses, p50 fine, CPU ~3% | `PYTHONASYNCIODEBUG=1`, slow-callback logging; move to a thread pool |
| Proxy buffers SSE | "streaming doesn't work in prod but works locally" | §2.6; inter-frame-timing synthetic check |
| Provider 429/529 | Every user's stream fails at once | Per-provider circuit breaker + jittered backoff + a fallback model; shed load with 429 rather than piling up |
| Zombie streams (client vanished, no FIN) | FD count climbs; spend continues | TCP keepalive, hard generation deadline, periodic sweep of streams older than N minutes |
| One org floods the gateway | Everyone's latency degrades | Per-org concurrency quota, not just a global cap |
| Unbounded history | Rising cost, rising TTFT, eventual context overflow | Trim/compact; measure with `count_tokens` (Day 4 §2.3) |

**Why this over the alternatives.** *Thread-per-request* is simpler and works to a few hundred streams, then dies on stacks and scheduling (§2.1). *WebSockets* buy bidirectional messaging you mostly don't need and add proxy/scaling complexity. *Polling for partial results* (client asks "any more?" every 500 ms) removes the long-lived descriptor but multiplies request count by ~40× and makes TTFT worse — it trades an FD problem for a request-rate problem, badly. *No gateway at all* (browser talks to the provider directly) leaks your API key, removes spend control, and makes (f) impossible.

---

## 2.6 Case studies

### ① Reverse-proxy response buffering silently destroys streaming — the "works locally, broken in prod" classic

**What happens.** Nginx's `proxy_buffering` directive defaults to **`on`**. With buffering on, Nginx reads the upstream response into its own buffers (`proxy_buffer_size`, `proxy_buffers`, spilling to `proxy_temp_path` on disk for large responses) and forwards to the client on its own schedule. For a normal HTML page this is a *feature*: it frees the upstream worker as fast as possible and shields it from slow clients. For an SSE stream it is a catastrophe — the proxy holds your token frames and releases them in lumps, or at the end.

The signature is unmistakable and the diagnosis is usually slow because the symptom is not an error: **streaming works perfectly against the dev server and appears completely broken behind the production proxy, with a 200 status, correct content, and no log line anywhere.** Teams routinely spend days on the application before suspecting the middle.

Nginx provides two documented fixes: set `proxy_buffering off;` for the streaming location, or have the upstream send the response header `X-Accel-Buffering: no`, which disables buffering for that response only. The second is the one to reach for first, because it lives in your application code and ships with your deploy (§2.2's example sets it). The same class of bug exists with any intermediary that buffers: gzip/compression middleware (it *must* accumulate to compress — exclude `text/event-stream`), some CDN configurations, corporate proxies, and AWS ALB/CloudFront settings.

**Engineering lessons.**

1. **§1.3's buffering stack does not end at your process.** It extends across every hop: your language's buffer → kernel → your proxy's buffer → the CDN → the browser's buffer. Any one of them can convert "streaming" into "batch," and none of them will tell you.
2. **A latency check that measures total response time cannot detect this.** The total is unchanged; only the *distribution over time* changed. Monitor **inter-frame arrival time** end-to-end through the real production path, or you are not monitoring streaming at all.
3. **Defaults are chosen for the common case, and you are not the common case.** `proxy_buffering on` is right for 99% of HTTP traffic. Every time you adopt a technology for a workload it wasn't defaulted for — streaming over a page-oriented proxy, an agent over a request/response framework — audit the defaults rather than inheriting them.

**Primary sources (verify current):** Nginx documentation for `ngx_http_proxy_module` — the `proxy_buffering`, `proxy_buffer_size`, `proxy_buffers` and `proxy_read_timeout` directives, and the `X-Accel-Buffering` response header (documented under `ngx_http_upstream_module`); the WHATWG HTML specification's Server-Sent Events section for the wire format; MDN's "Using server-sent events" page.

**Honest gap (Principle 7):** this is a pervasive, well-documented *configuration* failure rather than a single named company postmortem, and I'm not going to attach it to an invented incident. The evidence is the vendor documentation plus the fact that it reproduces on demand — you can produce it yourself in five minutes by putting a default Nginx in front of §2.2's server, which is a better teacher than a story.

### ② MCP's stdio transport — when a `print()` corrupts the protocol

**What it is.** The Model Context Protocol (MCP) is an open protocol for connecting agents to tools and data sources. One of its two standard transports is **stdio**: the client launches the server as a subprocess and speaks JSON-RPC over the child's **stdin and stdout** — descriptors 0 and 1 (§1.1), inherited across `fork`/`exec` (Day 2). Messages are newline-delimited JSON; the protocol specification is explicit that the server **must not write anything to stdout that is not a valid MCP message**, and that anything it wants to log should go to **stderr** (descriptor 2).

**Why this bites everyone who writes an MCP server.** Every language's most convenient debugging tool writes to stdout. A stray `print("loaded 42 tools")`, a library that logs to stdout by default, a warning banner, a progress bar, a Python `DeprecationWarning` routed to stdout — any of these interleaves non-JSON bytes into the message stream and **corrupts the protocol**. The failure is not a clean error: the client's parser hits a line that isn't JSON, and depending on implementation you get a cryptic parse error, a dropped message, or a hung session. The fix is trivial once you know it (log to stderr, `logging.StreamHandler(sys.stderr)`) and mystifying until you do.

**Engineering lessons — three, all straight out of Part 1.**

1. **Descriptors 0/1/2 are a convention with real semantics, and stdout is a *data channel* when someone says it is** (§1.1). The moment a program's stdout is a protocol stream rather than a human-readable log, every incidental write to it is a wire-format bug. This is the same discipline as a Unix filter: `grep`'s stdout is data, its diagnostics go to stderr — MCP just made the stakes higher.
2. **The parent/child descriptor plumbing is Day 2's `fork`/`exec` gap, and it is a security boundary too.** The child inherits whatever the parent gives it. Give it the pipes it needs and nothing else (`close_fds=True`, `O_CLOEXEC`), or a subprocess ends up holding your database socket.
3. **The buffering rules from §1.3 apply with a vengeance.** The child's stdout is a **pipe**, so it is block-buffered by default in Python and C — an MCP server that writes a response and doesn't flush will appear to hang while its reply sits in an 8 KB userspace buffer. This is exactly §1.3's tty-vs-pipe trap, wearing a protocol hat: correct code either flushes after every message or runs unbuffered.

**Primary sources (verify current — MCP is evolving quickly):** the Model Context Protocol specification's **Transports** page (modelcontextprotocol.io) for the stdio transport rules, including the requirement that the server not write non-MCP content to stdout and the guidance to use stderr for logging; the reference SDK implementations under github.com/modelcontextprotocol; and `pipe(7)` plus `setvbuf(3)` for the buffering mechanics underneath.

---

## 2.7 In production (Part 2, condensed per the [WORKING] tier)

**Key practices.**

- **Set `LimitNOFILE` and a concurrency semaphore from an actual calculation** (§2.4). Alert on `process_open_fds / process_max_fds > 0.8`.
- **Every upstream call gets a timeout; every turn gets a deadline.** Log and count both separately — a rise in deadline hits is your earliest signal of provider degradation.
- **Disable buffering end-to-end for SSE routes and monitor inter-frame timing**, not just total latency (§2.6).
- **Never block the event loop.** Run with `PYTHONASYNCIODEBUG=1` in staging; alert on slow-callback warnings. Wrap any synchronous SDK in `asyncio.to_thread`.
- **`PYTHONUNBUFFERED=1` in the image; log to stdout as structured JSON; let the platform rotate** (§1.8 design B).
- **Cancel upstream on client disconnect**, and record cancellations with tokens-spent — otherwise your spend and your delivered value diverge invisibly.
- **Measure tokens with the provider's own counter before sending** (§2.2), and enforce budgets before the call, not after the invoice.

**Top failure mode, named plainly: descriptor exhaustion from un-timed-out streams.** It starts as a slow climb in open FDs during an incident with the provider, holds steady for hours, and then the service stops accepting connections entirely while every dashboard shows low CPU, low memory, and healthy processes. The detection is one metric (open FDs vs limit); the prevention is one policy (a hard deadline on every held descriptor); the diagnosis without them is a very long night.

---

# PART 3 — THE BRIDGE

> No new concepts here. This Part is about what happens when Part 1's mechanisms and Part 2's workload are **the same process**, sharing **one descriptor table**, **one page cache**, and **one event loop thread**.

## 3.1 Where the layers meet: one table, three claimants

Your agent service is a single Linux process. Inside it, three logically separate concerns compete for the same three scarce, process-wide resources:

```
                     ONE PROCESS
  ┌───────────────────────────────────────────────────────────────┐
  │  ONE descriptor table (RLIMIT_NOFILE — §1.1)                  │
  │  ONE page cache share + RSS (Day 4)                           │
  │  ONE event-loop thread (§1.6)                                 │
  ├───────────────┬───────────────────┬───────────────────────────┤
  │ HTTP server   │ agent runtime     │ storage / logging         │
  │ (§1.6, §2.2)  │ (§2.1–2.4)        │ (§1.3, §1.4, §1.7, §1.8)  │
  │               │                   │                           │
  │ N client SSE  │ N upstream model  │ run-log fd (fsync'd)      │
  │ sockets       │ sockets           │ app-log fd (buffered)     │
  │ 1 listen sock │ DB/tool pools     │ transcript spool          │
  │ 1 epoll fd    │                   │ shipper's tail fds        │
  └───────────────┴───────────────────┴───────────────────────────┘
             every one of these is a descriptor from the SAME 1024/65536
```

The bridge is that **none of these subsystems knows about the others' consumption**, but the kernel enforces the limit against their sum. Three specific couplings follow, and each is a real incident shape:

**Coupling 1 — the FD budget is shared, so an agent-side leak breaks the HTTP server.** A provider outage leaves 4,000 upstream sockets hanging with no timeout (§2.4). The count crosses `RLIMIT_NOFILE`. The next `accept()` on the *listening* socket fails with `EMFILE` — so **new users cannot connect at all**, and the health check (an HTTP request!) fails too, so the orchestrator restarts the pod, dropping the 4,000 in-flight conversations that were about to recover. One missing timeout in the agent layer took down the web layer *and* triggered a self-inflicted outage.

**Coupling 2 — the event loop is shared, so a storage stall freezes every stream.** Your run log's `fsync` (§1.4, ~1 ms normally) hits a device stall — noisy neighbour, cloud burst credits exhausted, failing drive — and takes 800 ms. If that `fsync` is called directly from an `async def` handler, **the event-loop thread is blocked for 800 ms** and all 3,000 in-flight SSE streams stop emitting tokens (§1.6's analogy break #2). Users see 3,000 simultaneously stuttering chats caused by one disk. The fix is structural, not tuning: fsync happens on a dedicated writer thread/process with a queue, and the request path `await`s a future.

**Coupling 3 — the page cache is shared, so logging competes with everything.** Your app writes 1 GB/hour of logs into the page cache (§1.8), which is Day 4's dirty-page budget. Under `vm.dirty_ratio` throttling the kernel *stalls the writing process* — which is the same process serving the agents. Meanwhile the accumulated conversation history of 3,000 streams (Day 4 §2.3) is RSS competing with that page cache for the container's memory limit, and the OOM killer resolves the dispute with a `SIGKILL` that flushes nothing (Day 2).

## 3.2 The shared failure mode, traced end to end

One incident, one timeline, showing every section of today firing in sequence:

```
t+0:00  Model provider starts returning 529 (overloaded), then stops responding entirely.
        Your upstream sockets stay ESTABLISHED. No FIN, no RST. (§1.5: silence ≠ failure)

t+0:30  Retry logic re-issues requests. Old sockets have a 10-minute SDK default timeout
        and are still held. Open FDs: 1,200 → 4,800.               (§2.4: no deadline)

t+1:10  process_open_fds crosses 16,384. accept() returns EMFILE.  (§1.1)
        New users get connection refused. Existing streams keep working — so the
        dashboard looks *fine*: CPU 4%, memory 60%, error rate on existing traffic ~0.

t+1:12  The liveness probe is an HTTP GET. It cannot connect. Kubernetes marks the pod
        unhealthy and sends SIGTERM, then SIGKILL after the grace period.   (Day 2)

t+1:13  SIGKILL: the userspace log buffer is discarded (§1.3) — the last ~8 KB of logs,
        i.e. exactly the diagnostic evidence, is lost. Buffered run-log records that
        were never fsync'd are in the page cache and DO survive (§1.3/§1.4);
        fsync'd side-effect intents survive too, and on restart the run log correctly
        flags them as AMBIGUOUS for reconciliation.                          (§2.3)

t+1:14  The replacement pod starts, retries the same provider, and repeats the cycle.
        Now with a cold prompt cache and 3,000 users reconnecting simultaneously.
```

**Every mitigation is something from today:** a hard deadline on every upstream call (§2.4) stops the FD climb; a concurrency semaphore (§2.5) makes the service shed load with 429s instead of dying; alerting on `process_open_fds/process_max_fds` (§1.10) catches it at t+0:40 instead of t+1:12; `PYTHONUNBUFFERED=1` (§1.3) preserves the logs through the SIGKILL; a circuit breaker turns the retry storm into fast failures; and the run log's fsync boundary (§2.3) is what makes the post-incident reconciliation possible instead of guesswork.

## 3.3 The dependency map

**What the agent layer calls into the backend for:**

| Agent needs | Backend provides | Breaks when |
|---|---|---|
| Hold a user's stream open (§2.2) | A descriptor + an event loop that never blocks (§1.6) | Any blocking call → *all* streams stutter |
| Talk to the model provider (§2.1) | A socket, a connection pool, a timeout | No timeout → FD leak → `EMFILE` (§1.1) |
| Deliver tokens as they arrive (§2.2) | An unbuffered path through every hop (§1.3) | Proxy buffering → batch delivery (§2.6) |
| Not redo a side effect after a crash (§2.3) | `append + fsync` before the effect (§1.4, §1.7) | fsync skipped → double charge, or silent drop |
| Answer "what did it do?" a week later (§2.5) | Log rotation + shipping (§1.8) | `rm` instead of rotate → disk full, no evidence |
| Not exceed its budget (§2.4) | An FD limit set from a calculation (§1.10) | Default 1024 → fails at ~500 users |

**What the backend needs back from the agent layer:**

| Backend needs | Agent must | Breaks when |
|---|---|---|
| Bounded concurrency | Cap in-flight turns with a semaphore (§2.5) | Unbounded acceptance → `EMFILE`, OOM |
| Bounded per-connection state | Trim/compact history (Day 4 §2.3) | Growth → RSS → OOM kill (Day 4) |
| Descriptors released promptly | Cancel upstream on disconnect (§2.2) | Zombie streams → FD climb + spend |
| The event loop kept free | `await` everything; thread-pool the rest | One `time.sleep` → global stall (§1.6) |
| Predictable fsync load | Batch run-log writes; group-commit (§1.4) | Per-record fsync → 1,000 turns/s ceiling |
| Bounded log volume | Structured, size-capped lines to stdout (§1.8) | 200 MB stack dumps → dirty-page stalls |

**And the shared resources neither owns:**

```
  RLIMIT_NOFILE  ──►  client sockets + upstream sockets + pools + log fds + epoll
  page cache/RSS ──►  dirty log pages + socket buffers + conversation histories
  event loop     ──►  accept + SSE writes + agent orchestration + (wrongly) fsync
  the disk       ──►  run log fsyncs + app log writes + transcript spool + shipper reads
```

Every row is a place where a change in one layer silently degrades the other, and every one of them is invisible in a per-layer test. That is the real content of "the bridge": **the layers are only separate in your architecture diagram; in the kernel they are one process sharing three counters.**

## 3.4 The one sentence

*An agent server is a program that holds thousands of descriptors open for minutes at a time while computing almost nothing — so its correctness is decided by when you flush, its cost by when you fsync, its capacity by how you wait, and its failure by which of those three you forgot to bound.*

---

# Topic-wide wrap-up

## Cheat Sheet

**The three-level model (§1.1)**
```
fd (per-process int) ──► open file description (offset + flags) ──► inode (the object)
   dup/fork share ────────────────┘ (shared offset)      separate open() = new description
```

**The buffering stack, and what each crash level costs (§1.3)**

| Layer | Escape hatch | Survives `kill -9`? | Survives power loss? |
|---|---|---|---|
| Userspace buffer (Python/stdio) | `flush()` | ❌ | ❌ |
| Kernel page cache (dirty pages) | `fsync()` | ✅ | ❌ |
| Device write cache | drive FLUSH (issued by fsync) | ✅ | ✅ (if the drive is honest) |

**Durable replace, all four steps (§1.4)**
```python
f.write(data); f.flush(); os.fsync(f.fileno())   # 1,2: data durable
os.replace(tmp, final)                            # 3: atomic swap
dfd = os.open(dirname, os.O_RDONLY | os.O_DIRECTORY); os.fsync(dfd); os.close(dfd)  # 4: NAME durable
```

**fsync policy (§1.4)** — never for logs/caches; per-record only if you must; **group commit** otherwise (batch → one fsync → ack all). On fsync **error: crash, never retry** (§1.9).

**Waiting models (§1.5, §1.6)**

| Model | Cost per idle connection | Ceiling | Use when |
|---|---|---|---|
| Blocking, thread per conn | a thread + 8 MB vaddr | ~hundreds | few conns, CPU-heavy handlers |
| Non-blocking + spin | **100% of a core** | never | never |
| Non-blocking + `epoll` | ~socket buffers only | ~`RLIMIT_NOFILE` | many mostly-idle conns |

**Errno decoder ring**

| Error | Means | Do |
|---|---|---|
| `EAGAIN`/`EWOULDBLOCK` (11) | nothing ready right now | wait for readiness, retry. **Not a failure** |
| `EMFILE` (24) | *this process* is out of FDs | raise `LimitNOFILE`, find the leak (`lsof -p`) |
| `ENFILE` (23) | *the system* is out of FDs | `fs.file-max`; find the guilty process |
| `ESPIPE` (29) | seeking a pipe/socket | you can't; it's a stream |
| `EBADF` (9) | FD closed or never valid | usually a use-after-close race |
| `ENOSPC` (28) | disk full | check `lsof +L1` for deleted-but-open files |
| `EPIPE` / `SIGPIPE` | wrote to a closed pipe/socket | the peer hung up; stop writing |

**Diagnostics you should be able to type from memory**
```bash
ls /proc/<pid>/fd | wc -l            # descriptors held now
cat /proc/<pid>/limits | grep files  # the limit for that process
lsof -p <pid> | awk '{print $5}' | sort | uniq -c   # what KIND of fd is leaking
lsof +L1                             # deleted-but-open files eating disk
grep -E 'Dirty|Writeback' /proc/meminfo   # unflushed data = your risk window
iostat -x 1                          # await, %util: is the device the bottleneck?
strace -f -e trace=openat,close,fsync,epoll_wait -p <pid>   # what is it actually doing
ss -s                                # socket counts by state
```

**Agent-specific (Part 2)** — ~2 long-lived FDs per concurrent turn; `LimitNOFILE` from a calculation, not a default; a deadline on every upstream call; `proxy_buffering off` / `X-Accel-Buffering: no` for SSE; `PYTHONUNBUFFERED=1`; cancel upstream on client disconnect; `count_tokens` before you send.

---

## Build This

**Goal: a crash-proof append-only log service, and proof that it is.** ~2–3 hours. Linux (WSL2/container).

1. **`logd.py`** — a single-threaded `selectors`-based TCP server (§1.6) on port 9099. Each line a client sends is one record. Register the listening socket, accept non-blocking, handle partial reads by buffering per connection until `\n`.
2. **Framing** — append each record as `[4-byte length][4-byte CRC32][payload]` (§1.7, §2.3) to `data.log`, opened with `O_APPEND`.
3. **Three durability modes**, chosen by CLI flag: `--fsync=never`, `--fsync=always`, `--fsync=group` (batch for up to 5 ms or 100 records, then one fsync, then reply `OK <seq>` to *all* clients in the batch — never before the fsync).
4. **`replay.py`** — reads `data.log`, verifies every CRC, stops at the first torn record, prints the count of valid records and whether a torn trailer was found.
5. **The experiment.** For each mode: start the server, have a client send 20,000 records as fast as it can while printing every acked sequence number to its own file, then `kill -9` the server mid-stream. Run `replay.py`. Compare **records acked to the client** against **records recovered from the log**.
6. **The FD experiment.** With the server running, lower its `RLIMIT_NOFILE` to 64 and open connections until `accept()` fails. Confirm the errno is `EMFILE`, confirm the server *survives* (it must not crash — it should log and keep serving existing connections), then fix it properly by raising the limit and closing idle connections.
7. **The rotation experiment.** While the server is running, (a) `rm data.log` and watch `df` diverge from `du`, recovering the data via `/proc/<pid>/fd/N`; then (b) do it correctly: `mv data.log data.log.1`, send `SIGHUP`, and confirm in the handler that reopening loses no records (§1.8).

**Definition of done — all five must hold:**

- ✅ `--fsync=always` and `--fsync=group`: **recovered ≥ acked, always.** Not one acked record is ever missing after `kill -9`. (`kill -9` doesn't lose page-cache data — so also test with a container hard-stop or a VM reset if you can, and note in your README which failure you actually tested.)
- ✅ `--fsync=never`: reproduce a case where recovered < acked, and state exactly how many records were lost and why.
- ✅ `replay.py` reports a torn trailing record at least once, and truncates cleanly rather than crashing.
- ✅ You have a table of throughput (records/sec) for all three modes on your hardware, and the group-commit number is at least 10× the always number.
- ✅ Your README states the durability guarantee in one sentence, naming the exact failure it survives (process kill vs power loss) and the exact window it does not.

**Stretch:** add checkpointing + segment truncation (§1.7); add a `--tail` client using `epoll` on an `inotify` fd; measure and plot `epoll` vs a thread-per-connection version at 100, 1,000, and 5,000 connections.

---

## Active Recall & Self-Test (answer from memory, then check)

1. Draw the three-level FD model. What does `dup()` share that a second `open()` does not?
2. Name every buffer between `f.write("x")` and the flash cells. For each, say what destroys it.
3. Why is `fsync` slow? Name the four things it does. Which one is the barrier?
4. Write out the four-step durable file replace, and name the failure each step prevents.
5. What is `EAGAIN` and why is returning it not an error? What is a short write?
6. Why is `epoll` O(1) where `poll` is O(n)? What data structure changed?
7. What is the difference between level-triggered and edge-triggered, and what bug does ET cause?
8. Your service dies with "Too many open files" after six hours. List three causes and the command that distinguishes them.
9. Why does `O_NONBLOCK` do nothing for regular files? What do real systems use instead?
10. What did fsyncgate teach, and what is the correct response to an fsync error?
11. What is Redis's `appendfsync everysec` actually promising, and what is it not promising?
12. An agent turn takes 19 seconds and 4 ms of CPU. What resource are you actually spending, and how should you size the box?
13. Streaming works locally and appears broken behind the production proxy, with a 200 status. First hypothesis? Two fixes?
14. Where exactly does the durability boundary go in an agent that charges a credit card, and what must you write down *before* the fsync?
15. Name three process-wide resources the HTTP layer and the agent layer share, and one incident shape for each.

**60-second teach-back.** Explain to someone who has never heard the term: *what a file descriptor is, why `write()` doesn't put bytes on the disk, and why one thread can serve ten thousand connections.* You must use no jargon they haven't been given, and you must include one number.

---

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| What is a file descriptor? | A small integer indexing a per-process table of open kernel objects (files, sockets, pipes, epoll, …). The kernel always returns the lowest free number. |
| fd vs open file description vs inode? | fd = per-process int; description = offset + flags (shared by `dup`/`fork`); inode = the object. Separate `open()` = new description = independent offset. |
| Does `write()` put bytes on disk? | No. Into your language's buffer, then (after `flush`) the kernel page cache. Only a returned `fsync` reaches the device. |
| `flush()` vs `fsync()`? | `flush` survives `kill -9`; `fsync` survives power loss. |
| Why is `fsync` slow? | It's a barrier: data + metadata + journal + a device cache-FLUSH the drive must acknowledge. ~0.1–10 ms depending on hardware. |
| Group commit? | Batch N records, one `fsync`, then ack all N. Trades ≤1 ms latency for 10–100× throughput at identical durability. |
| Correct fsync-failure handling? | Crash. Never retry — pre-4.13 Linux cleared the error and returned success on the second call (fsyncgate). |
| Four steps to durably replace a file? | write+flush+fsync(tmp) → `rename` → fsync(parent directory). |
| `EAGAIN` means? | "Nothing ready right now." Control flow, not an error. |
| Short write means? | The kernel accepted only part of your buffer. Normal on pipes/sockets. Loop on the remainder. |
| Why is `epoll` O(1)? | FDs are registered once; the kernel attaches wait-queue callbacks that push ready items onto a ready list. No scanning per call. |
| Level- vs edge-triggered? | LT re-reports while data remains; ET reports only on transition. ET without draining to `EAGAIN` hangs connections. |
| `EMFILE` vs `ENFILE`? | Per-process FD limit vs system-wide. |
| Why is `O_NONBLOCK` useless for files? | Files are always "ready"; the read blocks on the page fault anyway. Use a thread pool or `io_uring`. |
| Redis `appendfsync everysec`? | The default: fsync once a second, so up to ~1 second of writes is lost on power failure. |
| Why did `rm app.log` fill the disk? | The writer still holds the fd; blocks free only when the last descriptor closes. `lsof +L1` finds it. |
| Why no logs from a Python container? | stdout is a pipe → block-buffered. `PYTHONUNBUFFERED=1`. |
| CPU utilization of an agent turn? | ~0.02%. It's a switchboard, not a factory: size by concurrency and descriptor-seconds. |
| Descriptors per concurrent agent turn? | ~2 long-lived (client stream + upstream model) plus fixed pools. |
| Why does streaming break behind Nginx? | `proxy_buffering` is on by default. Fix: `proxy_buffering off` or `X-Accel-Buffering: no`. |
| Durability boundary for a side-effecting tool? | Append intent + idempotency key, fsync, *then* call. Ambiguity on restart is reconciled by key, never blind-retried. |
| Why must an MCP stdio server never `print()`? | stdout is the JSON-RPC wire. Diagnostics go to stderr (fd 2). |

---

## Primary Sources (verify against these)

**Man pages (the actual contracts — read the RETURN VALUE and ERRORS sections):** `open(2)`, `read(2)`, `write(2)`, `close(2)`, `lseek(2)`, `dup(2)`/`dup2(2)`, `fcntl(2)`, `stat(2)`, `pipe(2)` and `pipe(7)`, `socket(2)` and `socket(7)`, `fsync(2)`, `fdatasync(2)`, `sync(2)`, `rename(2)`, `getrlimit(2)`/`setrlimit(2)`, `select(2)`, `poll(2)`, `epoll(7)` (read it end to end, including "Questions and answers"), `epoll_create1(2)`, `epoll_ctl(2)`, `epoll_wait(2)`, `proc(5)`, `setvbuf(3)`, `io_uring(7)`.

**Kernel documentation:** `Documentation/admin-guide/sysctl/vm.rst` (`dirty_ratio`, `dirty_background_ratio`, `dirty_expire_centisecs`); `Documentation/filesystems/ext4/`; `include/linux/fs.h` (`struct file`, `struct file_operations`); `include/linux/errseq.h` (the post-fsyncgate error-reporting mechanism).

**fsyncgate:** the pgsql-hackers thread "PostgreSQL's handling of fsync() errors is unsafe and risks data loss at least on XFS" (Craig Ringer, March 2018); the PostgreSQL wiki page **"Fsync Errors"**; LWN.net *"PostgreSQL's fsync() surprise"* (Jonathan Corbet, April 2018); *"Can Applications Recover from fsync Failures?"* (Rebello, Patel, Alagappan, Arpaci-Dusseau × 2, USENIX ATC 2020); PostgreSQL docs for `data_sync_retry`.

**Redis:** the "Redis persistence" documentation page; annotated `redis.conf`; Salvatore Sanfilippo, *"Redis persistence demystified"* (2012).

**Servers and streaming:** Dan Kegel's **C10K** page (1999) for the historical framing; Nginx `ngx_http_proxy_module` docs (`proxy_buffering`, `proxy_buffers`, `proxy_read_timeout`) and `ngx_http_upstream_module` (`X-Accel-Buffering`); the WHATWG HTML specification, Server-Sent Events section; MDN "Using server-sent events".

**Python:** the `selectors`, `asyncio`, `io`, `os`, and `socket` module documentation; PEP 3116 (the new I/O layer) for why `flush()` and the buffering hierarchy exist.

**Agentic:** the Model Context Protocol specification, **Transports** page (stdio rules); the Anthropic API documentation for streaming, token counting, and current model IDs (**verify — model IDs, parameters, and pricing drift fast; `temperature`/`top_p`/`top_k`/`budget_tokens` are removed on current Claude models and return 400**).

---

## Layered explanations

**10 seconds.** A file descriptor is a numbered ticket the kernel gives you for anything you can read or write — a file, a socket, a pipe. Writing to one doesn't put bytes on the disk; it puts them in a buffer. Making them actually durable is a separate, slow call called `fsync`. And because waiting for I/O is nearly free while threads are not, one thread with `epoll` can watch ten thousand connections at once.

**1 minute.** Every open thing in a Unix process is named by a small integer index into a per-process table — file descriptors 0, 1, 2 are stdin/stdout/stderr, and the kernel always hands out the lowest free number, which is how shell redirection works. The same `read`/`write` calls work on files, sockets, pipes, and devices, which is why Unix pipelines compose. But `write()` only copies your bytes into a buffer: first your language's (~8 KB, lost on `kill -9`) then the kernel's page cache (lost on power failure). Getting them to the device requires `fsync`, which is a hard barrier costing 0.1–10 ms — often the true ceiling on a database's transactions per second, which is why every durable system uses a **write-ahead log** (append + one batched fsync + only then acknowledge) instead of updating data in place. Waiting is the other half. A blocking `read` parks your thread with ~0% CPU but costs a whole thread; setting `O_NONBLOCK` gives you `EAGAIN` instead of waiting but a spin loop burns a full core; the answer is `epoll`, which registers descriptors once and wakes you only for the ready ones in O(1). That is the engine inside asyncio, Nginx, Redis, and Node — and it's exactly what an LLM agent server needs, because an agent turn spends 19 seconds waiting and 4 milliseconds computing.

**5 minutes.** Start with the ticket: an FD is an index into a per-process array of `struct file` pointers. Three levels matter — the descriptor (per-process int), the *open file description* (offset and flags, shared by `dup` and inherited across `fork`), and the inode (the object). Confusing them explains half of Unix's surprises: why `2>&1` interleaves correctly, why a forked child keeps your database socket alive after you close it, why a deleted-but-open file still consumes disk. The uniform `read`/`write` interface over files, sockets, pipes, and devices is a vtable in the kernel (`file_operations`) and is what makes shell pipelines and process substitution possible; it stops at `lseek`, which fails with `ESPIPE` on streams. Durability is a stack: your language buffers ~8 KB in userspace (lost on `kill -9`, and block-buffered rather than line-buffered when stdout is a pipe — the reason your container appears to have no logs), then the kernel holds dirty pages in the page cache (lost on power loss, visible as `Dirty:` in `/proc/meminfo`, written back after ~30 s or under memory pressure), then the drive holds them in its own DRAM. `fsync` walks all of that plus a filesystem journal commit and a device cache-flush barrier; it costs 0.1 ms on enterprise NVMe and 10 ms on a spinning disk, which sets a hard ceiling on serial commits. The universal escape is group commit, and the universal architecture is the WAL: append a checksummed record, fsync once per batch, acknowledge only after the fsync returns, apply to the real data structure lazily, checkpoint and truncate. The two hard-won rules are that a rename needs the parent directory fsync'd too, and that a failed fsync must be fatal — because before Linux 4.13 the kernel cleared writeback errors after reporting them once, so PostgreSQL's retry saw success over permanently lost data (fsyncgate, 2018). On the waiting side: blocking is thread-expensive and CPU-free; non-blocking gives `EAGAIN` and partial writes; `epoll` registers descriptors once and returns only ready ones via a kernel-maintained ready list, solving C10K and becoming the engine under asyncio, Nginx, Redis, Node, and Go's netpoller — with the caveat that one blocking call on the loop thread stalls every connection, and edge-triggered mode silently hangs connections you don't drain to `EAGAIN`. Layer an LLM agent on top and every one of these becomes operational: a turn is 99.98% waiting, so you size by descriptor-seconds and tokens rather than CPU; streaming holds a client SSE descriptor and an upstream descriptor open for the whole generation, so the buffering stack becomes a correctness concern (a default-configured Nginx will batch your tokens) and un-timed-out streams become `EMFILE`, which stops `accept()` and kills the health check while every dashboard says the service is idle and healthy; and any tool call with an external side effect needs its intent plus an idempotency key appended and fsync'd *before* the call, so a crash leaves an answerable question rather than an unanswerable one.

**Expert summary.** The descriptor is a capability: an integer index into a per-task `files_struct`, dereferenced to a `struct file` carrying `f_pos`, `f_flags`, and an `f_op` vtable, dereferenced again to an inode — a three-level indirection whose middle layer's sharing semantics (`dup`/`CLONE_FILES` share the description; independent `open` does not) determine offset aliasing, `O_NONBLOCK` propagation, and fd-inheritance security. The uniform `read`/`write`/`poll` surface is subtype polymorphism over `file_operations`, with `llseek` stubbed to `-ESPIPE` for non-seekable objects, and it is the substrate of Unix composability. Durability is a layered write-back hierarchy — userspace `stdio` buffer, kernel page cache governed by `dirty_ratio`/`dirty_background_ratio`/`dirty_expire_centisecs`, volatile device cache — collapsed only by `fsync`/`fdatasync`, which serialize data writeback, metadata, a journal commit under `data=ordered`, and a `REQ_OP_FLUSH`/`REQ_FUA` barrier; its latency, not its bandwidth, is the binding constraint, making group commit the canonical amortization and the WAL (checksummed append, single batched barrier, ack-after-fsync, lazy apply, checkpoint-and-truncate) the convergent design across relational engines, LSM stores, consensus logs, and filesystem journals. Correctness of that design depends on two non-obvious invariants — directory-entry durability requires an explicit parent fsync, and writeback-error semantics must be per-observer-sticky (`errseq_t`, ≥4.13) rather than consumed-and-cleared, the pre-fix behaviour having rendered fsync-retry a silent-data-loss path. Concurrency is orthogonal: blocking I/O trades thread-stack and scheduler pressure for a linear programming model; `O_NONBLOCK` yields `EAGAIN` and short-transfer semantics without solving readiness discovery; `epoll` moves readiness to a kernel wait-queue-callback-driven ready list, giving O(1) per-call cost in registered descriptors and making one-thread-per-N-idle-connections the dominant server architecture — with level-triggered as the safe default and edge-triggered requiring drain-to-`EAGAIN` discipline, and with regular-file I/O structurally excluded (hence thread pools, or `io_uring`'s SQ/CQ submission model). Agentic workloads are this machinery under an extreme parameter regime: microsecond-scale CPU against tens-of-seconds wall clock, two long-lived descriptors per logical request, incremental delivery over SSE whose end-to-end correctness is a property of every buffering layer in the path including intermediaries you do not control, and side-effecting tool invocations whose crash-consistency requires an intent record with an idempotency key made durable strictly before the effect — so that recovery yields a reconcilable question rather than an unrecoverable ambiguity. The unifying constraint is that the descriptor table, the dirty-page budget, and the event-loop thread are process-global and un-partitioned: an unbounded lease in any layer is a denial of service in every other.




