# Day P4 — Git & the Shell (and the P-phase checkpoint)

> **What this covers:** the Unix shell you will live in every day (`cd`, `ls`, `pwd`, `cat`, `mkdir`, `rm`, pipes `|`, redirection `>`, `man`) inside WSL2/Ubuntu, and **git from absolute zero** — what version control *is*, the core loop (`init`/`status`/`add`/`commit`/`log`/`diff`), the staging area, branches, merging, remotes and GitHub, `.gitignore`, and the habit of committing early and often. It ends with the **P-phase checkpoint** that proves you're ready for Day 1.
> **Prerequisite:** Days P1–P3 (you can write a type-hinted Python program with classes, read/write JSON, and use a virtual environment).
> **The ONE idea that unites this day:** *the shell and git are not "developer chores" — they are the operating surface of all software work, and therefore the operating surface of every AI agent that does software work. An agentic coding tool is a program that types shell commands and reads git diffs, exactly like you're about to.* Learn these tools as **your** hands today; recognize them as **the agent's** hands in Part 2.
>
> **How to read this:** every concept runs the ladder **intuition → analogy (with where it breaks) → concrete worked example → diagram → under-the-hood**. Every shell/git concept has a **runnable transcript**: the exact commands you type (`$ ...`) and the real output they produce — type along in your WSL2 terminal; do not just read.
>
> **Depth tiers:** **[CORE]** open every box · **[WORKING]** use it correctly, know the tradeoffs · **[AWARE]** know it exists.
>
> *(P-days are pure-fundamentals on-ramp material: per the course rules, the `### System design`, `### Case studies`, and `### In production` blocks are intentionally omitted here — they begin at Day 1. Everything else — ladder, runnable examples, recall — applies in full.)*

---

# PART 1 — BACKEND: The Shell and Git From Zero

## Overview & motivation — why a text window from 1978 still runs the world

**The problem before it.** You already know how to *use* a computer: click an icon, a window opens, click a button, something happens. That's a **GUI** (Graphical User Interface). GUIs are wonderful for discovery — everything you can do is visible — and terrible for three things that dominate real engineering work:

1. **Precision** — "click roughly there" is not a repeatable instruction. "Run exactly this command" is.
2. **Automation** — you cannot save a sequence of mouse clicks in a file, review it, share it, and run it again next week. You *can* do that with commands.
3. **Composition** — GUI programs are sealed boxes; you can't feed the output of one into another. Shell programs are open pipes; you can chain them freely.

The **shell** is a program that reads *text commands* you type, runs them, and prints their *text output*. It trades discoverability for precision, automation, and composition — the exact three properties engineering (and, as Part 2 shows, AI agents) cannot live without. This is why an interface designed in the 1970s is still the primary control surface of every server, every cloud machine, every CI pipeline, and every agentic coding tool in 2026.

**Why git in the same day?** Because git is a *command-line program* — you drive it from the shell — and because the two together answer the two most fundamental questions of software work: *"how do I make the machine do things?"* (shell) and *"how do I keep a trustworthy history of what I did?"* (git). Every single "Build" task in the next 100 days ends with a git commit. This day installs the muscle memory.

## First principles & core terminology **[CORE]**

Before any command, five words that must be precise, because people confuse them constantly:

- **Terminal (terminal emulator)** — the *window* that displays text and captures your keystrokes (Windows Terminal, the "Ubuntu" app). It draws characters; it does not understand commands. Historically a physical device (a screen + keyboard wired to a big shared computer); today a program that pretends to be one — hence "emulator."
- **Shell** — the *program running inside* the terminal that interprets what you type. On Ubuntu the default shell is **bash** ("Bourne-Again SHell"). The terminal is the phone handset; the shell is the person you're talking to. *Why it exists / what came before:* before interactive shells, you submitted punch cards or job files and got printed output hours later; the shell (1971, Ken Thompson's Unix shell) made the computer a conversation instead of a mail correspondence.
- **Prompt** — the text the shell prints when it's ready for your next command, e.g. `vinay@LAPTOP:~/projects$`. It typically shows *who* you are, *which machine* you're on, and *where* you are (`~` = your home directory). The `$` means "ordinary user" (a `#` would mean root/administrator — a convention worth knowing because tutorials use it).
- **Command** — the first word you type is a *program name* (`ls`, `git`); everything after it is **arguments** passed to that program. **Flags/options** are arguments starting with `-` (short: `-l`) or `--` (long: `--help`) that modify behavior. This is exactly the `def f(*args)` idea from P1, spelled with spaces instead of parentheses.
- **WSL2 (Windows Subsystem for Linux 2)** — a real Ubuntu Linux running inside Windows in a lightweight virtual machine. You use it because the entire backend/agents world (servers, Docker, deployment, the tools in this course) runs on Linux; WSL2 gives you a genuine Linux environment without leaving Windows. Inside WSL2, your Linux files live under `/home/<you>` and your Windows `C:` drive is visible at `/mnt/c` — keep course projects on the Linux side (`~/projects/...`), because file operations there are dramatically faster and tooling behaves correctly.

**Worked example — anatomy of one command.** When you type:

```
ls -l /home/vinay/projects
│  │  └────────┬─────────┘
│  │           └ argument: WHICH directory to list
│  └ flag: "-l" = long format (permissions, size, date)
└ program name: "list directory contents"
```

the shell (1) splits the line on spaces into `["ls", "-l", "/home/vinay/projects"]`, (2) finds the program file `ls` by searching the directories listed in its `PATH` variable (`/usr/bin/ls` — check with `which ls`), (3) starts it with those arguments, (4) waits for it to finish, and (5) prints the prompt again. Nothing magical: a word → a program, the rest → its arguments.

**Under the hood (opened once, then treated as a black box until Day 19).** Step (3) is two ancient, load-bearing system calls: the shell **forks** (the operating system clones the shell process) and the clone **execs** (replaces itself with the `ls` program). When `ls` exits, it hands back a number — the **exit code**: `0` means success, anything else means failure. The shell stores it in `$?`. Remember exit codes: they are how programs report success/failure *to other programs*, and Part 3 shows they play exactly the role for an agent that HTTP status codes will play for API clients from Day 8 onward.

---

## 1. Moving around: `pwd`, `ls`, `cd` **[CORE]**

**Depth: [CORE]** — you will run these hundreds of times a day, and every other command depends on understanding *where you are*.

### Intuition — the filesystem is a tree, and you are always standing somewhere

Everything on a Unix system lives in one big upside-down tree of **directories** (folders) starting at the **root**, written `/`. A **path** is directions through the tree: `/home/vinay/projects/journey` means "from the root, into `home`, into `vinay`, into `projects`, into `journey`."

The crucial mental model: **the shell always has a current location**, called the *working directory*. Every command you run is interpreted relative to it. Most beginner confusion ("file not found!?") is really "I'm not standing where I think I'm standing."

- `pwd` — **p**rint **w**orking **d**irectory: "where am I?"
- `ls` — **l**i**s**t: "what's here?"
- `cd` — **c**hange **d**irectory: "go there."

Two kinds of path:
- **Absolute** — starts with `/`, full directions from the root: `/home/vinay/projects`. Unambiguous, works from anywhere.
- **Relative** — no leading `/`, directions from *where you are*: `projects/journey`. Two special names exist in every directory: `.` (this directory) and `..` (the parent). `~` expands to your home directory (`/home/vinay`).

### Analogy — a city with one address system (and where it breaks)

The filesystem is a city: root `/` is the city itself, directories are streets and buildings, files are rooms. An absolute path is a full postal address ("India, Hyderabad, Road 12, Flat 4"); a relative path is directions from where you stand ("two doors down, then left"). `pwd` is checking the street sign; `cd` is walking. *Where the analogy breaks:* in a city you can only be reached by one route; in a filesystem a file can have *multiple* valid paths pointing at it (via `..`, `~`, symlinks), and moving yourself (`cd`) is instant and free — no distance exists.

### Runnable example — a full navigation session (type along in WSL2)

```
$ pwd
/home/vinay

$ ls
projects  notes.txt

$ ls -l
total 8
drwxr-xr-x 3 vinay vinay 4096 Jul 22 09:14 projects
-rw-r--r-- 1 vinay vinay  213 Jul 21 18:02 notes.txt

$ cd projects
$ pwd
/home/vinay/projects

$ ls -a
.  ..  journey

$ cd journey
$ cd ..            # back up one level (to projects)
$ cd ~             # jump home from anywhere
$ pwd
/home/vinay

$ cd /nonexistent
bash: cd: /nonexistent: No such file or directory
$ echo $?
1
```

**Why this works, line by line.** `pwd` prints the absolute path of where the shell currently stands. Bare `ls` lists names only; `ls -l` (long) shows one line per entry — the leading `d` in `drwxr-xr-x` marks `projects` as a **d**irectory while the `-` marks `notes.txt` as a plain file, followed by permissions, owner, size in bytes, and modification time. `ls -a` (**a**ll) reveals entries whose names start with `.` — including the ever-present `.` and `..`, and (later, crucially) git's hidden `.git` directory. `cd projects` used a *relative* path — it worked only because we stood in `/home/vinay`; `cd ..` climbs to the parent; `cd ~` teleports home via an absolute shortcut. The failed `cd` demonstrates two things at once: errors are printed as text (to a separate error stream — see the redirection section), and the exit code `$?` flipped from `0` to `1` — the machine-readable "that failed" signal.

### Visual — the tree you just walked

```
/                          ← root
└── home
    └── vinay              ← "~", your home
        ├── notes.txt
        └── projects
            └── journey    ← cd projects, cd journey walked down this spine;
                              cd .. climbed back up one edge
```

### Under the hood **[CORE]**

The "current location" is not shell trickery — every process on Linux carries a *current working directory* attribute maintained by the kernel. `cd` is one of the very few commands that is **built into the shell itself** rather than a separate program: if it were a separate program, it would change *its own* working directory and then exit, leaving the shell's unchanged (fork/exec, above — the child's changes die with it). This is a beautiful first proof that the fork/exec model has real consequences: `which ls` prints `/usr/bin/ls`, but `type cd` prints `cd is a shell builtin`.

---

## 2. Reading, creating, destroying: `cat`, `mkdir`, `rm` **[CORE]**

**Depth: [CORE]** for daily fluency; the *dangers* of `rm` are the load-bearing part.

### Intuition

- `cat` — con**cat**enate and print: dump a file's contents to the screen. (Its real purpose is joining files — `cat a.txt b.txt` prints both — but 95% of usage is "show me this file.")
- `mkdir` — **m**a**k**e **dir**ectory. `mkdir -p a/b/c` creates the whole chain of parents at once (`-p` = parents).
- `rm` — **r**e**m**ove. `rm file` deletes a file; `rm -r dir` deletes a directory and everything inside it (**r**ecursive); `rm -rf` adds **f**orce (no prompts, no complaints about missing files).

**The thing to burn in now:** `rm` does **not** move things to a Trash/Recycle Bin. It unlinks them immediately and permanently. There is no undo. `rm -rf` on the wrong path is the classic career-scarring mistake — and one of the two reasons git exists in your life (a committed file can always be recovered; an `rm`-ed uncommitted one cannot).

### Analogy — and where it breaks

`cat` is holding a document up to the light; `mkdir` is buying a new labeled binder; `rm` is a paper shredder bolted next to your desk — instant, silent, thorough. *Where the analogy breaks:* a real shredder leaves strips someone could reassemble; and physical offices have janitors who empty bins on a schedule. `rm` has neither delay nor bin — the "shredding" is the *removal of the file's directory entry*, and the space is recycled whenever the OS pleases. Treat every `rm -r` as irreversible.

### Runnable example — create, inspect, destroy

```
$ cd ~/projects
$ mkdir -p demo/sub
$ cd demo
$ echo "hello from the shell" > greeting.txt     # ('>' explained fully in §4)
$ cat greeting.txt
hello from the shell

$ ls
greeting.txt  sub

$ rm greeting.txt
$ rm sub
rm: cannot remove 'sub': Is a directory
$ rm -r sub
$ ls
$ cd .. && rm -r demo
```

**Why this works.** `mkdir -p demo/sub` built two nested directories in one call — without `-p`, `mkdir demo/sub` fails if `demo` doesn't exist yet. `echo` printed a string, and `>` *redirected* that output into a file instead of the screen (that operator gets its own section below); `cat` read it back. Plain `rm` refuses directories on purpose — deleting a whole subtree is a bigger decision, so it demands the explicit `-r`. The final line chains two commands with `&&`: run the second **only if** the first succeeded (exit code `0`) — your first taste of programming *with* exit codes rather than merely observing them.

### Visual — what `rm` actually removes

```
directory "demo"                      file contents on disk
┌─────────────────────┐              ┌──────────────────────┐
│ greeting.txt ──────────────────────▶ "hello from the..."  │
└─────────────────────┘              └──────────────────────┘
        │
   rm greeting.txt deletes THIS arrow (the directory entry).
   The bytes linger on disk until overwritten — but nothing
   points to them anymore, so for you they are gone.
```

### Under the hood **[WORKING — named, not fully opened]**

A directory is itself a small file: a table mapping *names → inodes* (an inode is the kernel's record describing a file's actual data blocks). `rm` calls `unlink()`, which deletes the name→inode mapping; when an inode's link count hits zero and no program has it open, the kernel frees its blocks. That's the whole reason recovery tools sometimes work (blocks not yet reused) and why you must never rely on them. Deeper filesystem internals are a Phase-2+ topic — black box deliberately closed here.

---

## 3. Pipes `|` — composing programs like Lego **[CORE]**

**Depth: [CORE]** — pipes are *the* idea of Unix, and (Part 2/3) the structural twin of an agent chaining tool calls.

### Intuition & the problem it solves

Suppose you want "the 5 most recently modified files whose names contain `py`." No single program does that. Before pipes you had two bad options: write a custom program for every such question, or run one command, save its output to a file, run the next command on that file, delete the file — clumsy plumbing by hand.

The 1973 insight (Doug McIlroy, Ken Thompson — one of the most consequential ideas in computing): **make every program read a stream of text in and write a stream of text out, then let the shell connect the output of one directly to the input of the next.** That connector is the pipe, `|`:

```
command1 | command2 | command3
```

Each program stays tiny and single-purpose; arbitrary questions get answered by *composition*. This works only because of a shared convention — every program speaks the same interface, **lines of text** — which is precisely the property that later makes JSON-speaking tools composable for agents.

Three standard streams exist on every process (file descriptors 0, 1, 2):
- **stdin** (0) — the input stream (default: your keyboard)
- **stdout** (1) — the normal-output stream (default: your screen)
- **stderr** (2) — the *error*-output stream (default: also your screen, but separately routable — so errors don't corrupt data flowing down a pipe)

`a | b` connects `a`'s stdout to `b`'s stdin. stderr is *not* piped — errors still reach your eyes even mid-pipeline.

### Analogy — a kitchen assembly line (and where it breaks)

One cook chops, hands to the next who fries, hands to the next who plates. Nobody stores intermediate results in the fridge (no temp files); each station is simple; rearranging stations creates new dishes. *Where it breaks:* in a kitchen, the chopper finishes the whole pile before the fryer starts. In a pipe, **all programs run simultaneously** and data flows through in small chunks — the fryer starts frying the first onion while the chopper is still chopping. That concurrency is why pipes handle gigabytes without gigabytes of memory: nothing ever holds the whole stream.

### Worked example — building a pipeline one stage at a time

Question: *how many files in this directory tree are Python files?* Build it incrementally, inspecting each stage — this is exactly how experienced people write pipelines:

```
$ ls
checkpoint.py  data.json  fizzbuzz.py  notes.md  wordcount.py

$ ls | grep py          # stage 2: keep only lines containing "py"
checkpoint.py
fizzbuzz.py
wordcount.py

$ ls | grep py | wc -l  # stage 3: count the surviving lines
3
```

Trace the data: `ls` writes five lines to stdout → the pipe hands them to `grep py`, which passes through only matching lines (three) → the second pipe hands those to `wc -l` (**w**ord **c**ount, `-l` = count **l**ines), which prints `3`. Each program did one dumb thing; the *pipeline* answered a question none of them knows how to ask.

### Runnable example — a pipeline you'll actually use on repos

Once your journey repo has history (§7), this one-liner answers "what did I work on most recently?":

```
$ git log --oneline | head -3
bb32614 add P3 typed API fetcher
6af1010 add P2 bank account modules
cdfcb5f add P1 fizzbuzz and wordcount

$ git log --oneline | wc -l
7
```

**Why this works.** `git log --oneline` prints one line per commit, newest first, to stdout — it neither knows nor cares what happens next. `head -3` reads stdin and passes through only the first 3 lines; `wc -l` counts them all. git never grew "show only N commits" and "count commits" features into a bloated monolith* — the shell composes them for free. (*It did eventually add flags like `-n`, but the point stands: with pipes it never *needed* to.)

### Visual — the plumbing

```
            stdout→stdin           stdout→stdin
┌────────┐  ╔════════╗  ┌────────┐  ╔════════╗  ┌────────┐
│  ls    │──╢  PIPE  ╟──│ grep py│──╢  PIPE  ╟──│ wc -l  │──▶ screen: 3
└────────┘  ╚════════╝  └────────┘  ╚════════╝  └────────┘
     │                       │                       │
   stderr ─────────────────stderr──────────────────stderr──▶ screen (bypasses pipes)
```

### Under the hood **[CORE]**

A pipe is a **kernel object: a fixed-size in-memory buffer (typically 64 KB) with a write end and a read end.** When you run `ls | grep py`, the shell (1) asks the kernel for a pipe, (2) forks two children, (3) wires `ls`'s stdout (fd 1) to the pipe's write end and `grep`'s stdin (fd 0) to its read end, (4) starts both *at the same time*. Flow control is automatic: if the writer outruns the reader, the buffer fills and the kernel *pauses the writer* until space frees up (backpressure); if the reader outruns the writer, the reader blocks waiting for data. When `ls` exits, the write end closes, `grep` reads end-of-file, finishes, and exits. No temp files, no whole-output-in-memory, no coordination code — the kernel is the coordinator. This "bounded buffer between concurrent producers and consumers" pattern will reappear as message queues in Phase 2; you have now seen the original.

---

## 4. Redirection `>` — sending output somewhere else **[CORE]**

**Depth: [CORE]** — the other half of stream plumbing; agents and scripts capture output with it constantly.

### Intuition

A pipe connects a program to *another program*. **Redirection** connects a program to a *file*:

- `command > file` — write stdout to `file`, **replacing** its contents (creates it if missing).
- `command >> file` — **append** stdout to the end of `file`.
- `command < file` — feed `file` to the command's stdin (rarer, but symmetric).
- `command 2> file` — redirect **stderr** (stream number 2) separately; `2>&1` merges stderr into stdout.

The unifying idea: **programs don't know or care where their output goes.** `ls` writes to "stdout" — an abstract slot. The shell decides before launch whether that slot points at your screen, a pipe, or a file. One mechanism, three destinations. This is the *decoupling of computation from destination*, and it is why the same program needs zero changes to be used interactively, in a pipeline, or in a log-writing script.

### Analogy — the mail chute (and where it breaks)

A program dropping output on stdout is an office worker dropping letters down a mail chute; they never see where the chute leads. The building manager (the shell) re-routes the chute — to the lobby noticeboard (screen), to another office (pipe), or to a filing cabinet (file) — *before the workday starts*. *Where it breaks:* re-routing a real chute mid-day is possible; redirection is fixed when the command launches. And `>` has a sharp edge no mail chute has: pointing it at an existing file **instantly empties the file** — even before the command runs. `>` truncates first, asks never.

### Worked example — the difference between `>` and `>>`, traced

```
state of log.txt after each command:

$ echo "run 1" >  log.txt      →  "run 1"
$ echo "run 2" >  log.txt      →  "run 2"            (run 1 DESTROYED — > replaces)
$ echo "run 3" >> log.txt      →  "run 2" + "run 3"  (>> appends)
```

One character of difference; one destroyed line of data. Scripts that keep logs almost always want `>>`.

### Runnable example — capturing real output, and splitting stderr from stdout

```
$ cd ~/projects/journey
$ ls > files.txt
$ cat files.txt
files.txt
fizzbuzz.py
wordcount.py

$ python fizzbuzz.py > output.txt 2> errors.txt
$ cat output.txt | head -5
1
2
Fizz
4
Buzz
$ cat errors.txt
$                                   # empty — the run had no errors

$ python nonexistent.py > out.txt 2> err.txt
$ cat out.txt
$ cat err.txt
python: can't open file '/home/vinay/projects/journey/nonexistent.py': [Errno 2] No such file or directory
```

**Why this works.** `ls > files.txt` — the shell created/truncated `files.txt`, pointed `ls`'s stdout at it, then ran `ls`; notice `files.txt` lists **itself**, proof the file was created *before* `ls` ran. The Python run demonstrates the two streams being routed to two different files: normal program output landed in `output.txt`, and the diagnostic stream landed in `errors.txt` (empty on success). The failing run inverts it — nothing on stdout, the error message on stderr. This separation is a *contract*: downstream consumers (pipes, files, agents parsing output) get clean data on stdout, while error text takes its own channel. It's the same discipline as an API returning data in the body and signaling failure via a status code — a connection Part 3 makes explicit.

### Visual — one program, three destinations

```
                        ┌──▶ screen        (default)
  program ──▶ stdout ───┼──▶ | next-cmd    (pipe, §3)
                        └──▶ > file.txt    (redirection)

  program ──▶ stderr ── separate channel: still the screen unless 2> routes it
```

### Under the hood **[WORKING]**

File descriptors again: before `exec`ing the program, the shell simply `open()`s the target file and installs it as fd 1 (or fd 2 for `2>`). The program then writes to "fd 1" exactly as always — it is structurally *incapable* of knowing the difference. `>` opens the file with the truncate flag, `>>` with the append flag; that single flag is the entire difference between the two operators. Deliberately stopping here — fd tables, `dup2`, and friends belong to Day 19's concurrency material.

---

## 5. `man` and getting unstuck **[WORKING]**

**Depth: [WORKING]** — you need the habit, not the internals.

### Intuition & worked example

Nobody memorizes flags. The **man**ual pages are the offline, authoritative reference for every standard command, installed on the machine itself: `man ls` opens the full documentation for `ls` in a pager (space = next page, `/word` = search, `q` = quit). Faster for a quick reminder: `ls --help` prints a compact summary. For git specifically, `git help commit` (or `man git-commit`) opens that subcommand's manual.

```
$ man ls          # full manual in a pager; press q to leave
$ ls --help | head -6
Usage: ls [OPTION]... [FILE]...
List information about the FILEs (the current directory by default).
Sort entries alphabetically if none of -cftuvSUX nor --sort is specified.

Mandatory arguments to long options are mandatory for short options too.
  -a, --all                  do not ignore entries starting with .
```

**Why this matters.** `man` pages read like legal documents — precise, exhaustive, unfriendly — but they are *the primary source* (Rule 7): the ground truth blog posts paraphrase and LLMs approximate. The habit to build: guess with `--help`, verify with `man`, and only then search the web. *Analogy:* `--help` is the recipe card taped inside the cupboard; `man` is the full cookbook on the shelf. *Where it breaks:* cookbooks explain *why* dishes work; man pages mostly document *what* the flags do — for the "why," you have these notes and the primary-source books listed in the wrap-up.

---

## 6. Version control — what problem git actually solves **[CORE]**

**Depth: [CORE]** — the concept, before any command.

### The pain that motivated it (term necessity, in three eras)

**Era 0 — no tooling at all.** Everyone who has written anything recognizes this directory:

```
thesis.doc
thesis_final.doc
thesis_final_v2.doc
thesis_final_v2_REALLY_FINAL.doc
thesis_final_v2_REALLY_FINAL_advisor_edits.doc
```

This *is* version control — done by hand, and failing in every way: no record of *what changed* between copies or *why*, no way to compare them, names lie ("FINAL" never is), and two people editing simultaneously produce two diverged copies that someone must merge by eyeball. Early software teams did exactly this, plus its networked cousin: **emailing zipped snapshots (tarballs) of the code back and forth**, with one poor "integrator" hand-merging everyone's changes. The Linux kernel itself ran for years on mailed patches applied by one man.

**Era 1 — centralized version control (CVS 1990, Subversion/SVN 2000).** Idea: one *central server* holds the single official history; developers "check out" the latest code, edit, and "commit" changes back over the network. Enormous improvement — one history, real diffs, names replaced by revision numbers. But the *centralization* is a triple flaw: (1) nearly every operation (view history, diff old versions, commit) needs the server, so no network = no version control; (2) the server is a single point of failure — corrupt it and the project's entire history can die; (3) committing publishes to everyone immediately, so people avoid committing half-done work, so commits become huge, rare, and unreviewable.

**Era 2 — distributed version control (git, 2005).** In 2005 the proprietary tool the Linux kernel used (BitKeeper) revoked its free license, and Linus Torvalds wrote git in about two weeks to replace it. The radical design choice: **every developer's clone contains the complete history — every version of every file, ever.** There is no privileged central copy in git's design (GitHub is a *conventional* central meeting point, not a structural one). Consequences, each a direct fix to an SVN flaw: everything except sync is local and instantaneous, so you commit constantly; every clone is a full backup, so history is nearly unkillable; and cheap local branching (§8) changed how people *work*, not just how they save. That is why distributed won.

### What git is, in one sentence each of its three roles

1. **A time machine** — a permanent, browsable, restorable snapshot of your entire project at every point you chose to save. `rm` a file, gut a function, break everything: one command returns any committed state. This deletes the *fear* from experimentation.
2. **A collaboration protocol** — the mechanism by which multiple people (and — Part 2 — multiple *agents*) edit the same codebase concurrently and systematically merge the results, with every change attributed and explained.
3. **Your 100-day proof-of-work** — every day's Build becomes a commit on GitHub: a public, timestamped, tamper-evident ledger of your learning. In interviews, "here is my repo with 100 days of commits" outweighs any certificate. The habit *is* the credential.

### Analogy — the save-point system (and where it breaks)

Git is save points in a video game: hit save (commit) before a hard boss (risky refactor); die (break the code); reload (checkout/restore); nothing is ever truly lost. *Where it breaks — in a way that makes git strictly better:* a game keeps a handful of save slots and overwrites old ones; git keeps **every** save forever, lets you **compare** any two ("what exactly changed between save 12 and save 40?"), lets you **branch** into parallel storylines and later **merge** them, and attaches to every save a *message explaining why*. And unlike autosave, git only saves when *you* say so — which is a feature: each commit is a deliberate, coherent, described unit of work.

### Worked example — the same afternoon, with and without git

*Without:* you refactor `wordcount.py` for an hour, break it, can't remember what the working version looked like, and lose the evening reconstructing it from memory (or a stale copy called `wordcount_backup2.py`).

*With:* you committed the working version at 2 pm (`git commit -m "wordcount handles punctuation"`). At 3 pm the refactor is a disaster. You run `git diff` to see exactly what you changed since 2 pm, salvage the two good ideas, and `git restore wordcount.py` to reset the rest. Total cost of the failed hour: nothing but the hour. **The commit converted an irreversible risk into a reversible one** — this is the single sentence to remember about why version control exists.

---

## 7. The core loop: `init` → `status` → `add` → `commit` → `log` → `diff`, and the staging area **[CORE]**

**Depth: [CORE]** — this loop is your daily heartbeat for the next 100 days; every box gets opened.

### Intuition — three zones, one flow

A git repository has three zones your work moves through:

1. **The working directory** — the ordinary files you see and edit. Nothing special.
2. **The staging area** (also called **the index**) — a *draft of the next snapshot*. `git add` copies a file's current contents into it. Nothing is saved permanently yet.
3. **The repository** (the hidden `.git` directory) — the permanent history. `git commit` seals whatever the staging area holds into an immutable snapshot with a message, an author, and a timestamp.

*Why does the staging area exist at all?* It seems like bureaucracy — why not commit files directly? Because it lets you **compose** commits: you edited four files this afternoon, but two belong to "fix the JSON bug" and two to "add the CLI flag." Stage and commit the first pair, then the second — two clean, single-purpose commits instead of one mixed blob labeled "stuff." The staging area is where you *edit the story you're about to tell*. (SVN had no such thing; every commit was "whatever is dirty right now." The index is one of git's genuinely new ideas.)

### Analogy — the photographer's table (and where it breaks)

Your working directory is the messy studio. The staging area is the table where you *arrange exactly what will be in the photo*. The commit is pressing the shutter: the arrangement is captured permanently in the album, and the table is ready for the next shot. `git status` is glancing at the table; `git diff` is comparing the studio to the last photo. *Where it breaks:* a photo flattens reality into pixels, but a commit stores a perfect, restorable copy of every staged file — you can climb back *into* any photo. Also, `git add` copies the file *as it is at that moment*: edit the file again after adding it, and the table still holds the old version until you `add` again. This trips up every beginner once; now it won't trip you.

### Runnable example — a complete first session (the transcript to internalize)

```
$ cd ~/projects
$ mkdir journey && cd journey
$ git init
Initialized empty Git repository in /home/vinay/projects/journey/.git/

$ ls -a
.  ..  .git

$ echo "# 100 Days of Backend & Agentic AI" > README.md
$ git status
On branch main
No commits yet
Untracked files:
  (use "git add <file>..." to include in what will be committed)
	README.md
nothing added to commit but untracked files present (use "git add" to track)

$ git add README.md
$ git status
On branch main
No commits yet
Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   README.md

$ git commit -m "add README: journey begins"
[main (root-commit) f4a2c1e] add README: journey begins
 1 file changed, 1 insertion(+)
 create mode 100644 README.md

$ git log
commit f4a2c1e8b9d0f3a7c6e5d4b3a2f1e0d9c8b7a6f5 (HEAD -> main)
Author: Vinay Vemula <vemula.vinay@genpact.com>
Date:   Wed Jul 22 10:15:03 2026 +0530

    add README: journey begins
```

**Why this works, command by command.** `git init` created the hidden `.git` directory — the entire repository lives in there; delete `.git` and the project becomes a plain folder again (history gone), while everything *outside* `.git` stays untouched. The first `git status` reports `README.md` as **untracked**: git sees a file it has never been told about — git tracks nothing by default; you opt files in. `git add README.md` copied the file's current contents into the staging area; `status` now shows it under "Changes to be committed" — on the photographer's table. `git commit -m "..."` sealed the snapshot: the output names the branch (`main`), notes this is the very first commit (`root-commit`), and shows the snapshot's ID `f4a2c1e` — the first 7 characters of a 40-character fingerprint explained under the hood below. `-m` supplies the message inline (omit it and git opens an editor). `git log` replays history newest-first: full ID, author (from your one-time `git config --global user.name/user.email` setup), date, message — and the decoration `(HEAD -> main)`, decoded below.

### Runnable example — the edit cycle and `git diff`'s two faces

```
$ echo "Day P4: learned shell + git." >> README.md
$ git status --short
 M README.md

$ git diff
diff --git a/README.md b/README.md
index 3e5a7f2..9c4b1d8 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,2 @@
 # 100 Days of Backend & Agentic AI
+Day P4: learned shell + git.

$ git add README.md
$ git diff                # now prints NOTHING
$ git diff --staged       # ...because the change moved here
diff --git a/README.md b/README.md
index 3e5a7f2..9c4b1d8 100644
--- a/README.md
+++ b/README.md
@@ -1 +1,2 @@
 # 100 Days of Backend & Agentic AI
+Day P4: learned shell + git.

$ git commit -m "log Day P4 progress"
[main 8d1e5b2] log Day P4 progress
 1 file changed, 1 insertion(+)
```

**Why this works — and the trap it disarms.** `>>` appended a line (not `>`, which would have wiped the README — §4 paying rent already). `status --short` shows ` M` = modified, unstaged. Read the `git diff` output like this: lines starting with `+` were added, `-` removed, and `@@ -1 +1,2 @@` is a **hunk header** — "this excerpt covered 1 line of the old file starting at line 1, and covers 2 lines of the new file." Then the crucial move: after `git add`, plain `git diff` shows *nothing*. It isn't broken — plain `git diff` compares **working directory ↔ staging area**, and they are now identical. `git diff --staged` compares **staging area ↔ last commit** and shows your pending change. Two diffs, two boundaries, one mental picture:

```
 working directory ──── git diff ────▶ staging area ──── git diff --staged ────▶ last commit (HEAD)
        ▲                                   ▲                                        │
        └── your editor writes here    git add copies here                git commit seals here
```

### Runnable example — the time machine in action: undoing all three zones

The whole point of §6 was that commits convert irreversible risk into reversible risk. Prove it — deliberately damage each of the three zones, then undo each damage with the matching command:

```
$ cat README.md
# 100 Days of Backend & Agentic AI
Day P4: learned shell + git.

# DAMAGE 1: ruin the working directory (unstaged edit)
$ echo "GARBAGE LINE" >> README.md
$ git status --short
 M README.md
$ git restore README.md              # throw away the unstaged edit
$ cat README.md
# 100 Days of Backend & Agentic AI
Day P4: learned shell + git.

# DAMAGE 2: stage something you didn't mean to
$ echo "half-finished thought" >> README.md
$ git add README.md
$ git restore --staged README.md     # pull it back OFF the table...
$ git status --short
 M README.md                          # ...edit survives, just unstaged again
$ git restore README.md              # ...and now discard it entirely

# DAMAGE 3: delete a committed file outright
$ rm README.md
$ ls
$ git restore README.md              # resurrected from the last commit
$ cat README.md
# 100 Days of Backend & Agentic AI
Day P4: learned shell + git.
```

**Why this works — one command, three directions, one rule.** `git restore <file>` copies the file *out of the index* back into the working directory — that's why it undid both the garbage edit and the `rm`: in each case the index still held the last-committed content, and restore re-materialized it. `git restore --staged <file>` moves in the other direction — it copies the *last commit's* version back into the index, un-staging your change without touching the working directory (your edit survives on disk). Map it onto §7's three-zone diagram: `add` moves work right, `commit` seals it, and the two `restore` forms are the left-pointing arrows. The rule that falls out: **anything committed is recoverable in one command; anything never committed has no net under it** — `rm` on an uncommitted file (§2) is still gone forever. One more level of time travel for completeness: `git checkout <hash> -- <file>` restores a file from *any historical commit*, not just the latest — try `git checkout f4a2c1e -- README.md` to see the very first README return (then `git restore --staged README.md && git restore README.md` to come back to the present). Whole-repo resets (`git reset --hard`) exist and are **[AWARE]** for now: powerful, occasionally destructive, learn them when a real mess demands it.

### Under the hood — the object model: what a commit *actually is* **[CORE]**

This is the box most tutorials never open, and opening it makes every confusing git behavior predictable. Inside `.git/objects` git stores exactly four kinds of object, each identified by the **SHA-1 hash of its own content**:

- **blob** — the bytes of one file's contents. No filename, no metadata — just contents.
- **tree** — one directory listing: rows of *(mode, type, hash, name)* pointing at blobs (files) and other trees (subdirectories). Filenames live *here*, not in blobs.
- **commit** — a tiny text record: the hash of one top-level **tree** (= the complete snapshot), the hash(es) of its **parent commit(s)**, author, timestamp, message.
- *(tag — a named pointer with a message; **[AWARE]**, ignore until you release software.)*

**Content-addressing** is the load-bearing idea: an object's ID *is* the SHA-1 hash of its content — a 40-hex-character fingerprint with the property that different content yields (for all practical purposes) different hashes. Three enormous consequences: **integrity** — if any stored byte is corrupted, its hash no longer matches its name, so git detects it instantly; **deduplication** — two identical files (or the same unchanged file across a thousand commits) hash identically, so the content is stored *once*; **history tamper-evidence** — each commit's hash covers its parent's hash, so rewriting any ancient commit changes every hash after it. History isn't merely stored — it's *sealed*, blockchain-style (git had the idea first).

**See it with your own eyes** — `cat-file -p` pretty-prints any object by hash:

```
$ git log --oneline
8d1e5b2 (HEAD -> main) log Day P4 progress
f4a2c1e add README: journey begins

$ git cat-file -p 8d1e5b2                      # the COMMIT object
tree a91f0c37d2e6b5a4c3f2e1d0b9a8c7f6e5d4c3b2
parent f4a2c1e8b9d0f3a7c6e5d4b3a2f1e0d9c8b7a6f5
author Vinay Vemula <vemula.vinay@genpact.com> 1753159230 +0530
committer Vinay Vemula <vemula.vinay@genpact.com> 1753159230 +0530

log Day P4 progress

$ git cat-file -p a91f0c3                      # the TREE it points at
100644 blob 9c4b1d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a3b2c	README.md

$ git cat-file -p 9c4b1d8                      # the BLOB the tree points at
# 100 Days of Backend & Agentic AI
Day P4: learned shell + git.
```

**Why this works / what you just proved.** A commit is ~6 lines of text: a pointer to one tree, a pointer to its parent, identity, message. The tree maps the name `README.md` to a blob hash; the blob is the file bytes, verbatim. So *"a commit is a snapshot"* is literal: commit → tree → blobs reaches every file's exact contents. And *"history is a chain"* is literal too: `parent f4a2c1e...` links this commit to the previous one. Follow parents backward and you have the whole history; a commit with **two** parents is a merge (§8). Because commits can share parents and later rejoin, the full structure is a **DAG** — a *directed acyclic graph*: arrows point backward in time (directed), and no commit can be its own ancestor (acyclic).

**What HEAD and branches actually are — the demystification.** Ready: **a branch is a 41-byte file containing a commit hash.** Nothing more.

```
$ cat .git/refs/heads/main
8d1e5b2c7a9f4e3d2b1a0c9d8e7f6a5b4c3d2e1f

$ cat .git/HEAD
ref: refs/heads/main
```

`main` is a file (a **ref**) holding the hash of the branch's latest commit. **HEAD** is one more tiny file answering "where am I right now?" — usually by naming a branch. When you commit: git writes the new commit object, then updates `refs/heads/main` to the new hash. That's the whole ceremony. This is why branches are instant and free (§8): creating one writes a 41-byte file; nothing is copied. And **the index/staging area** is likewise just a real file, `.git/index` — a binary table of *(path, blob-hash, metadata)* rows describing the snapshot-in-draft; `git add` writes the blob into `.git/objects` and updates the file's row; `git commit` converts the table into tree objects and wraps a commit around them. No black boxes remain: **objects (blobs/trees/commits) + refs (branch files + HEAD) + the index — that is the entire machine.**

```
 .git/HEAD ──▶ refs/heads/main ──▶ commit 8d1e5b2 ──parent──▶ commit f4a2c1e ──▶ (root)
                                        │                          │
                                      tree a91f0c3              tree 3b7d9e1
                                        │                          │
                                      blob 9c4b1d8              blob 3e5a7f2
                                     (README v2)               (README v1)
```

*(One honesty note, per Rule 7: SHA-1 is cryptographically broken for adversarial collisions, so newer git uses a hardened variant and a SHA-256 mode exists; for integrity-checking your own work the model above is exactly right. Verify against the Pro Git book, ch. 10, if you go deeper.)*

---

## 8. Branches and merging: `git branch`, `git checkout -b`, `git merge` **[CORE]**

**Depth: [CORE]** — cheap branching is *the* feature that made git change how people work.

### Intuition & the problem it solves

You're mid-way through a risky experiment when an urgent fix is needed on the working code. Without branches you're stuck: your working directory holds a half-broken hybrid. **A branch is an independent line of development** — start one for the experiment, and `main` stays untouched and shippable; switch between them freely; when the experiment succeeds, **merge** it back. In SVN, branching copied the whole tree on a server and was so painful teams avoided it. In git — §7 just showed you why — a branch is a 41-byte pointer file, so creating one is instant, and the workflow "new branch for every piece of work" became the industry norm.

- `git branch` — list branches (`*` marks the current one)
- `git checkout -b name` — create branch `name` *and* switch to it (modern equivalent: `git switch -c name`; `checkout` is what you'll see in most docs, so learn it first)
- `git merge name` — fold branch `name`'s work into the branch you're standing on

### Analogy — parallel drafts of a document (and where it breaks)

You have a solid essay draft (`main`). To try a restructure without endangering it, you duplicate the file and hack on the copy (`experiment`). Success → fold the changes back (merge); failure → delete the copy, the original never flinched. *Where it breaks, twice:* (1) duplicating a large folder costs time and disk; a git branch copies *nothing* (pointer file — the object store is shared, and identical content is deduplicated by hashing anyway). (2) With manual copies, folding changes back means eyeballing two documents; git merges *automatically* whenever the two lines touched different parts, and pinpoints the exact overlapping lines when they didn't (conflicts, below).

### Runnable example — branch, change, switch back, merge

```
$ cd ~/projects/journey
$ git branch
* main

$ git checkout -b experiment
Switched to a new branch 'experiment'

$ echo "An idea I am not sure about yet." > idea.md
$ git add idea.md
$ git commit -m "sketch risky idea"
[experiment 4c7f9a1] sketch risky idea
 1 file changed, 1 insertion(+)
 create mode 100644 idea.md

$ git checkout main
Switched to branch 'main'
$ ls
README.md                      # idea.md is GONE from the working directory...

$ git checkout experiment
$ ls
README.md  idea.md             # ...and back. Files follow the branch.

$ git checkout main
$ git merge experiment
Updating 8d1e5b2..4c7f9a1
Fast-forward
 idea.md | 1 +
 1 file changed, 1 insertion(+)

$ git branch -d experiment
Deleted branch experiment (was 4c7f9a1).
```

**Why this works.** `checkout -b experiment` wrote a new ref file pointing at the current commit and moved HEAD to it — instant. The commit on `experiment` advanced only that branch's pointer; `main` stayed at `8d1e5b2`. Switching branches rewrote the **working directory** to match the target branch's snapshot — that's why `idea.md` vanished on `main` and reappeared on `experiment`: your files on disk are a *view of the current branch*, materialized from the object store. The merge output says **`Fast-forward`**: since `main` had no new commits of its own, its history was a strict prefix of `experiment`'s, so git merely *slid the `main` pointer forward* to `4c7f9a1` — no new commit needed. Finally `branch -d` deleted the 41-byte pointer; the commits it pointed at remain, now reachable from `main`.

### Worked example — the real merge (both sides moved) and the conflict

Fast-forward is the easy case. When **both** branches gained commits, git must create a **merge commit** — a commit with **two parents** (the DAG earning its name):

```
        A───B───C   main
             \
              D───E   feature

$ git checkout main && git merge feature

        A───B───C───M   main        M = merge commit, parents C and E
             \     /
              D───E
```

git finds the common ancestor (`B`), computes what each side changed since `B`, and combines both — automatically, when the changes touch different lines. When both sides edited the **same lines**, git refuses to guess and declares a **conflict**:

```
$ git merge feature
Auto-merging README.md
CONFLICT (content): Merge conflict in README.md
Automatic merge failed; fix conflicts and then commit the result.

$ cat README.md
# 100 Days of Backend & Agentic AI
<<<<<<< HEAD
Day P4: shell + git mastered.
=======
Day P4: learned the terminal and version control.
>>>>>>> feature
```

**How to read and fix it.** git wrote *both* versions into the file: between `<<<<<<< HEAD` and `=======` is your current branch's version; between `=======` and `>>>>>>> feature` is the incoming one. Resolution is manual and honest: open the file, keep what's right (either side, or a blend), **delete the three marker lines**, then `git add README.md` and `git commit`. A conflict is not an error — it is git correctly declining to make a semantic judgment only a human (or, someday, a reviewing agent) can make. Don't panic; read the markers; decide; add; commit.

### Runnable example — manufacture a conflict on purpose, then resolve it end-to-end

Never meet your first conflict under deadline pressure. Create one deliberately right now, in a scratch repo, so the shape is familiar forever:

```
$ cd ~/projects && mkdir conflict-lab && cd conflict-lab && git init -q
$ echo "The deadline is Friday." > plan.txt
$ git add plan.txt && git commit -qm "initial plan"

$ git checkout -b urgent                      # side A changes the line...
Switched to a new branch 'urgent'
$ echo "The deadline is Wednesday." > plan.txt
$ git commit -qam "pull deadline in to Wednesday"

$ git checkout main                           # ...and side B changes the SAME line
$ echo "The deadline is next Monday." > plan.txt
$ git commit -qam "push deadline to next Monday"

$ git merge urgent
Auto-merging plan.txt
CONFLICT (content): Merge conflict in plan.txt
Automatic merge failed; fix conflicts and then commit the result.

$ git status
On branch main
You have unmerged paths.
  (fix conflicts and run "git commit")
Unmerged paths:
  (use "git add <file>..." to mark resolution)
	both modified:   plan.txt

$ cat plan.txt
<<<<<<< HEAD
The deadline is next Monday.
=======
The deadline is Wednesday.
>>>>>>> urgent

$ echo "The deadline is Wednesday (urgent wins)." > plan.txt   # decide + remove markers
$ git add plan.txt
$ git commit -m "merge urgent: Wednesday deadline stands"
[main 7b3e0c4] merge urgent: Wednesday deadline stands

$ git log --oneline --graph
*   7b3e0c4 merge urgent: Wednesday deadline stands
|\
| * 5a1d9f2 pull deadline in to Wednesday
* | c8e4b21 push deadline to next Monday
|/
* 2f0a6d3 initial plan

$ cd .. && rm -r conflict-lab                 # lab over
```

**Why this works, beat by beat.** Both branches edited the *same line* of the *same file* since their common ancestor (`initial plan`), which is the one case three-way merge cannot decide — so `merge` stopped mid-flight and put the repo in a special "merging" state: `git status` shows `Unmerged paths`, and the file on disk contains both candidates fenced by markers. Resolution was three ordinary moves you already know: edit the file to the version you actually want (here: a blend — urgent's date, plus a note), `git add` it (which is how you *tell git* "this path is resolved" — add's second job), and `git commit` (message pre-filled if you omit `-m`) to complete the merge. The `--graph` log then shows the diamond: history genuinely forked and rejoined, and the merge commit `7b3e0c4` sits at the join with **two parents** — run `git cat-file -p 7b3e0c4` and you'll see two `parent` lines, the §7 object model predicting exactly this. If mid-conflict you panic and want out: `git merge --abort` returns everything to the pre-merge state — even conflict resolution is reversible.

### Under the hood **[WORKING — interface + mechanism named]**

`git merge` runs a *three-way merge*: base = the common-ancestor snapshot (found by walking the DAG), plus the two tips. For each file it compares hunks: changed on one side only → take it; changed identically → take once; changed differently on the same lines → conflict markers. The merge commit then records both tips as parents, which is how history remembers *that* two lines of work converged and *what* each contained. Merge *strategies* and rebasing are deliberately left closed until you need them.

---

## 9. Remotes and GitHub: `clone`, `remote`, `push`, `pull` **[CORE]**

**Depth: [CORE]** for the four commands and the mental model; GitHub-the-product features are **[AWARE]**.

### Intuition & terminology

Everything so far lived on your laptop — which dies with your laptop and helps no collaborator. A **remote** is another copy of the repository, somewhere else, that your local repo knows by name and can synchronize with. **GitHub** is a company hosting git repositories with a web UI, access control, and collaboration features on top (pull requests, issues) — the de-facto public square. Precision matters here: *git is the tool; GitHub is a place to host repositories.* (Alternatives: GitLab, Bitbucket. Interchangeable for this course.)

- `git clone <url>` — copy a remote repository to your machine: full history, all objects, working directory checked out, and the remote pre-registered under the conventional name **`origin`**.
- `git remote -v` — list configured remotes and their URLs.
- `git push` — upload your new local commits to the remote and advance its branch pointer.
- `git pull` — download new commits *from* the remote and merge them into your current branch. (Strictly `pull` = `fetch` — download objects and update your remote-tracking refs — *then* `merge`. Knowing that decomposition **[WORKING]** explains every weird `pull` behavior you'll ever see.)

Because every clone is complete (Era 2, §6), your laptop and GitHub each hold the entire history — sync is exchanging *missing objects and updated pointers*, not shipping the project wholesale.

### Analogy — cloud-synced with a manual sync button (and where it breaks)

`origin` is your cloud drive for code — except sync is **explicit**: `push` = upload, `pull` = download, and nothing moves until you say so. *Where it breaks, importantly:* Dropbox syncs *files* continuously and resolves concurrent edits by making sad duplicates ("conflicted copy (2)"); git syncs *commits* — whole reasoned snapshots — and resolves concurrent edits with a real merge (§8). Manual-ness is a feature: you push only coherent, committed, message-carrying units, never a half-saved file. And unlike a cloud drive, *every* participant has the full history — GitHub going down loses you nothing but the meeting point.

### Runnable example — `git clone`: the whole history arrives in one command

Before wiring up your own repo, feel what "every clone is complete" means by cloning someone else's — here, git's own repository of sample docs (any public repo works; this one is small):

```
$ cd ~/projects
$ git clone https://github.com/octocat/Hello-World.git
Cloning into 'Hello-World'...
remote: Enumerating objects: 13, done.
remote: Total 13 (delta 0), reused 0 (delta 0), pack-reused 13
Receiving objects: 100% (13/13), done.

$ cd Hello-World
$ ls -a
.  ..  .git  README

$ git log --oneline
7fd1a60 (HEAD -> master, origin/master) Merge pull request #6 from Spaceghost/patch-1
7629413 New line at end of file. --Signed off by Spaceghost
5531221 first commit

$ git remote -v
origin	https://github.com/octocat/Hello-World.git (fetch)
origin	https://github.com/octocat/Hello-World.git (push)
```

**Why this works.** One command produced three things: a working directory checked out at the latest commit (`README` on disk), the **entire history** in `.git` (`git log` walks all three commits without ever touching the network — try it with Wi-Fi off), and a pre-configured remote named `origin` pointing back where it came from. Nothing was "downloaded on demand"; the full object store crossed the wire once (13 objects — blobs, trees, commits, §7's machine). This is the Era-2 promise made tangible: you now hold a complete, independent, offline-capable copy — if GitHub vanished tonight, this repo would survive on your disk, history and all. **For the Build:** if you create the GitHub repo *first* (with its auto-generated README), this same command is your starting move — `git clone git@github.com:<you>/journey.git` — and `origin` comes pre-wired, letting you skip the `remote add` in the next transcript.

### Runnable example — the day's actual workflow: repo on GitHub → connect → push

The other starting order (local work exists first). On github.com: **New repository** → name `journey` → *no* README/license (you have local commits; an empty remote avoids an immediate merge). GitHub then shows the "push an existing repository" commands — decoded here:

```
$ cd ~/projects/journey
$ git remote add origin git@github.com:vinayvemula/journey.git
$ git remote -v
origin	git@github.com:vinayvemula/journey.git (fetch)
origin	git@github.com:vinayvemula/journey.git (push)

$ git push -u origin main
Enumerating objects: 9, done.
Counting objects: 100% (9/9), done.
Writing objects: 100% (9/9), 782 bytes | 782.00 KiB/s, done.
To github.com:vinayvemula/journey.git
 * [new branch]      main -> main
branch 'main' set up to track 'origin/main'.

$ git push          # after future commits: this is all it takes
Everything up-to-date
```

**Why this works.** `remote add origin <url>` wrote one config entry: "the name `origin` means this URL" — no data moved. (The `git@github.com:` form uses **SSH keys** for authentication — generate with `ssh-keygen -t ed25519`, paste `~/.ssh/id_ed25519.pub` into GitHub → Settings → SSH keys; the `https://` form works too and asks for a token. Do the SSH setup once and forget it.) `push -u origin main` uploaded every object reachable from `main` that the remote lacked (all 9 — blobs, trees, commits from §7–8) and set the remote's `main` to your latest commit; `-u` recorded `origin/main` as this branch's **upstream**, which is what lets every later sync be a bare `git push` / `git pull` with no arguments.

```
   your laptop                                GitHub ("origin")
 ┌──────────────────────────┐               ┌──────────────────────────┐
 │ working dir              │    push ───▶  │                          │
 │ staging area             │               │   full history           │
 │ .git (full history) ─────┼── ◀── pull ───│   (a bare repo — no      │
 └──────────────────────────┘               │    working directory)    │
      add/commit are LOCAL                  └──────────────────────────┘
      only push/pull touch the network
```

### Worked example — why `pull` before `push`, traced

You commit on your laptop; meanwhile the remote gained a commit you don't have (a collaborator — or you, from another machine, or an agent). Then:

```
$ git push
 ! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'github.com:vinayvemula/journey.git'
hint: Updates were rejected because the remote contains work that you do not
hint: have locally. ... You may want to first integrate the remote changes
hint: (e.g., 'git pull ...') before pushing again.

$ git pull
Merge made by the 'ort' strategy.
 dayplan.md | 3 +++
 1 file changed, 3 insertions(+)
$ git push
To github.com:vinayvemula/journey.git
   4c7f9a1..b2e8d47  main -> main
```

**The logic:** push is only allowed when it *fast-forwards* the remote branch (§8's easy case). If the remote has commits you lack, blindly moving its pointer to your commit would orphan them — so git refuses and tells you to integrate first. `git pull` fetched the missing commits and merged them into yours (possibly surfacing §8-style conflicts to resolve); now your history *contains* the remote's, the push fast-forwards, everyone's work survives. Rejection isn't failure — it's the protocol protecting other people's commits from you, and yours from them.

---

## 10. `.gitignore` — telling git what NOT to track **[WORKING]**

**Depth: [WORKING]** — one concept, one file format, big daily payoff.

### Intuition & the problem

Projects fill with files that don't belong in history: virtual environments (`.venv/` — thousands of reinstallable third-party files, P3), interpreter caches (`__pycache__/`), editor droppings (`.vscode/`), OS noise (`.DS_Store`), and — critically — **secrets** (`.env` files with API keys). Committing them bloats every clone, buries `git status` in noise so real changes hide, and a pushed secret must be treated as **leaked and rotated immediately** (history is permanent and public — deleting the file in the *next* commit removes nothing from the *previous* one; §7's object model guarantees that).

**`.gitignore`** is a plain text file of patterns; matching **untracked** files become invisible to `git status` and un-addable by accident. It's a bouncer's list at the door — *and where that analogy breaks:* the bouncer only screens people *outside*; a file already tracked (already inside the club) is unaffected by adding its pattern — you must first evict it with `git rm --cached <file>` (removes from tracking, keeps it on disk).

### Runnable example — the noise disappears

```
$ python3 -m venv .venv
$ mkdir __pycache__ && touch __pycache__/junk.pyc .env
$ git status --short
?? .env
?? .venv/
?? __pycache__/

$ cat > .gitignore <<'EOF'
.venv/
__pycache__/
*.pyc
.env
EOF

$ git status --short
?? .gitignore

$ git add .gitignore && git commit -m "ignore venv, caches, secrets"
[main 1f6c3d9] ignore venv, caches, secrets
 1 file changed, 4 insertions(+)
```

**Why this works.** Each line is a pattern: a trailing `/` matches directories (`.venv/`), `*` is a wildcard (`*.pyc` = any compiled-Python file anywhere), plain names match exactly (`.env`). After creating the file, the three noise entries vanish from `status` — only `.gitignore` itself remains, and committing *it* is the point: the ignore rules ship with the repo, so every clone (and every collaborator, and every agent) inherits them. Rule of thumb for what to ignore: **anything regenerable** (installs, caches, build outputs) **and anything secret**. Source code, configs, docs, and lockfiles get committed.

---

## 11. Commit early, commit often — the working rhythm **[WORKING]**

**Depth: [WORKING]** — a practice, argued for properly rather than asserted.

### Intuition — commits are checkpoints, and checkpoints are only useful if they exist

Every property you've built today — time-machine restores (§6), surgical `diff`s (§7), fearless branches (§8), meaningful pushes (§9) — scales with commit **frequency and coherence**. One giant end-of-day "did stuff" commit gives you a time machine with one save slot, a diff too big to read, and a history that explains nothing. The professional rhythm:

- **Commit each coherent unit of work** — a passing test, a working function, a finished section. Guideline: *if you'd be annoyed to lose it, commit it.* For this course: **every day's Build = at least one commit, pushed.**
- **Write messages that answer "why," in the imperative:** `add overdraft guard to BankAccount` (imperative because it completes "this commit will …", the convention git's own history uses) — not `changes`, `fix`, `asdf`. The reader of your message is you-in-three-weeks, mid-debug, running `git log` to find where behavior changed; write for that desperate person.
- **Stage deliberately.** `git add <specific files>` beats reflexive `git add .` — the staging area exists (§7) precisely so unrelated edits become separate commits.

**Worked example — the two histories, three weeks later.** A bug appeared sometime this month. History A: `7f3d2c1 did stuff`, `9e8b4a6 more work`, `c2d1f0e final`. History B: fourteen commits like `parse CLI args before loading config` and `handle missing JSON file in load_accounts`. In B you skim `git log --oneline`, spot the suspicious message, `git diff` that single small commit, and see the bug in minutes; git even ships a tool (`git bisect`, **[AWARE]** — it binary-searches history for the commit that broke things) whose entire usefulness is proportional to how small your commits are. In A you have archaeology. The habit costs seconds; the payoff compounds for the life of the repo — and Part 2 adds the twist: your commit messages become *context an agent reads*.

---

## Part 1 — comparisons, failure modes, mistakes

| | Manual copies (`_final_v2`) | Dropbox-style sync | SVN (centralized) | **git (distributed)** |
|---|---|---|---|---|
| What's saved | whole files, ad hoc | every file save, continuous | changesets, on a server | deliberate snapshots + full local history |
| Why did it change? | ❌ | ❌ | ✅ messages | ✅ messages |
| Works offline | ✅ | partially | ❌ (server for almost everything) | ✅ (everything but push/pull) |
| Concurrent edits | eyeball-merge | "conflicted copy (2)" | locks / server merge | three-way merge, explicit conflicts |
| Branches | copy the folder | ❌ | slow, server-side | 41-byte pointer file, instant |
| Full backup at each participant | ❌ | ❌ | ❌ (server only) | ✅ every clone |

**Failure modes & the classic mistakes (beginner → senior):**
- *Beginner:* editing after `git add` and committing the stale staged version (re-`add`; §7's photographer's-table break) · `>` where `>>` was meant, wiping a file (§4) · `rm -rf` with a mistyped path — **pause and re-read every `rm -rf` before Enter** (§2) · panic at conflict markers instead of reading them (§8).
- *Intermediate:* committing `.venv/` or `.env` because `.gitignore` came too late (§10 — and a pushed secret is a *rotated* secret, not a deleted file) · `git add .` swallowing unrelated changes into one commit · treating a rejected push as an error instead of "pull first" (§9).
- *Senior-flavored:* messages that say *what* the diff already shows instead of *why* · giant long-lived branches that guarantee conflict archaeology at merge time · force-pushing (`push --force`) over teammates' commits — the one command that *can* destroy others' pushed work; treat as **[AWARE]**-and-avoid until Phase 2.

## Interview & practice (Part 1)

1. What exactly does `git add` do, and why does the staging area exist? (Copies content into the index/draft snapshot; enables composing multi-file edits into separate coherent commits.)
2. A branch is what, physically? What is HEAD? (A ref file containing a commit hash; the file saying which ref you're on.)
3. Explain `git diff` vs `git diff --staged` by naming the two boundaries each compares.
4. Why can git detect a corrupted object instantly, and why can't you silently rewrite an old commit? (Content-addressing; each commit hashes its parent's hash.)
5. Walk the double-lost-work scenario `push` rejection prevents, and the pull→merge→push resolution.
6. Why is `|` between two programs better than a temp file between them? (Concurrency, backpressure, no disk, no cleanup.)
7. `command > out.txt 2> err.txt` — what lands where, and why do two streams exist at all?

*Practice — easy:* create a directory tree with one command; list *hidden* files; count the lines of your `.gitignore` with a pipe. *Medium:* stage two files, unstage one (`git status` tells you the command), commit the other; then explain what `git restore --staged` moved between which zones. *Hard:* create a conflict on purpose (two branches editing the same README line), resolve it, then use `git cat-file -p` on the merge commit to show its two parents. *Thought experiment:* design what breaks if two files in one repo hashed to the same SHA-1 — which of git's three guarantees (integrity, dedup, tamper-evidence) fails first?

---

# PART 2 — AGENTIC AI: These Exact Tools Are the Agent's Hands

> Backend from here on is a black box: "the shell runs commands and returns text + an exit code; git stores snapshot history" = everything Part 1 built. Nothing new is introduced about *how* they work — this Part shows **where they reappear** when the operator is a model instead of you. Full agent mechanics (tool schemas, loops, constrained decoding) arrive in Phase 1; today you get the honest preview.

## Overview & motivation — the agent has no hands but yours

An **agentic coding tool** (Claude Code — the tool this course itself is being built with — plus Cursor's agent mode, Copilot Workspace, and kin) is a program that puts a language model in a loop: the model reads your request, decides on an *action*, the harness *executes* that action, and the result is fed back to the model, repeat. The revelation Day P4 equips you for: **the actions are almost entirely the commands you just learned.** The model cannot touch a filesystem, run a test, or make a commit — it can only *emit text*. A harness turns designated bits of that text into real `ls`, `cat`, `python`, `git diff`, `git commit` invocations, exactly as the shell turns your typed text into fork/exec. You spent Part 1 becoming a shell user; the model is *also* a shell user, just one that types via an API. Everything you know about the tools transfers one-to-one to reasoning about, supervising, and debugging agents.

## 1. Agentic coding tools operate *through* the shell and git **[WORKING]**

**Depth: [WORKING]** — the mental model and its consequences; harness internals come Day 22+.

### Intuition

Watch an agentic coding session and you see a loop of Part-1 vocabulary:

1. **Orient** — the agent runs `ls`, `cat app.py`, `git log --oneline`, `git status` to learn where it is and what exists. (Your §1–2 navigation, verbatim: the model also "stands somewhere" and must look around, because it starts each session knowing *nothing* about your machine.)
2. **Act** — it edits files and runs `python -m pytest` or the program itself.
3. **Observe** — the harness captures **stdout, stderr, and the exit code** (§4's three channels) and feeds them back as text. Exit `0` → proceed; nonzero + a stderr traceback → the model reads the traceback (P1's skill!) and fixes.
4. **Checkpoint** — it inspects its own work with `git diff` and, when asked, records it with `git commit -m "..."`.

### Analogy — a remote contractor working over chat (and where it breaks)

You've hired a skilled contractor who can *only* communicate by text message: they text you commands, you run them and text back what the terminal said. Competent contractor + faithful messenger = real work done, sight unseen. *Where it breaks:* a human contractor remembers yesterday, gets suspicious when something feels off, and refuses obviously destructive instructions on their own judgment. The model remembers only what's in the current conversation, believes what stdout tells it, and will run what it decides to run — which is why the *harness* (not the model) enforces permissions, and why the read-only allowlist in the runnable example below is load-bearing rather than decorative.

### Worked example — one real turn of an agentic session, traced

Request: *"the tests are failing, fix it."* The transcript of what the harness actually executes:

```
agent → run: git status --short
harness → " M accounts.py\n?? test_accounts.py"          (exit 0)
agent → run: python -m pytest -q
harness → "F.\n...\nTypeError: withdraw() missing 1 required
           positional argument: 'amount'\n1 failed, 1 passed" (exit 1)
agent → (edits accounts.py — fixes the method signature)
agent → run: python -m pytest -q
harness → "2 passed"                                       (exit 0)
agent → run: git diff --stat
harness → " accounts.py | 2 +-\n 1 file changed..."        (exit 0)
agent → run: git add accounts.py && git commit -m "fix withdraw() signature"
harness → "[main 3e9f2a1] fix withdraw() signature"        (exit 0)
```

Every single line is Part-1 material: `status` to orient, exit code `1` + stderr traceback as the failure signal, re-run to *verify* (agents that don't re-run are agents that lie), `diff` as self-review, a §11-quality commit message as the checkpoint. An agent is not magic — it is **your Part-1 workflow, executed by a model, mediated by a harness.**

## 2. Pipes and redirection = the original tool-chaining **[WORKING]**

**Depth: [WORKING]** — a structural correspondence that will make Phase-1 agent loops feel familiar instead of new.

### Intuition

§3's pipeline discipline — small single-purpose programs, text streams as the universal interface, output of one becoming input of the next — is *structurally identical* to how an agent composes tool calls:

```
shell:   ls            |  grep py        |  wc -l           → answer
agent:   list_files()  →  search(qry)   →  summarize(txt)  → answer
```

Same shape, one pivotal difference: in a shell pipeline **you** design the chain in advance and the shell executes it rigidly; in an agent loop **the model** picks the next "pipe segment" *dynamically, after reading the previous output*. An agent is a pipeline whose plumber is inside the pipeline. That difference is the entire value (it adapts: sees an error, changes course) and the entire risk (it can also chain toward something you didn't intend) — foreshadowing why Part 3 cares about guardrails. The shared precondition is the deeper lesson: pipes work because every program speaks *lines of text*; tool-chaining works because every tool speaks *structured text* (JSON). Composition always rests on a shared interface — carry that sentence to Day 8 (HTTP) and Day 22 (tools).

**Worked example — the same question, both styles.** *"How many commits mention 'fix'?"* Shell (you, static plan): `git log --oneline | grep -c fix` → `4`. Agent (dynamic plan): calls `run_shell_command("git log --oneline")`, *reads* the 40 returned lines, counts mentions itself — or, if the log came back huge, decides to call again with a narrower command. Adaptivity replaced `grep`; the composition principle didn't change. And redirection has an exact echo too: `pytest > result.txt 2> errors.txt` capturing streams into files *is* what every harness does to capture streams into tool-result messages — same plumbing, different sink.

## 3. The git repo is the agent project's memory and audit log **[WORKING]**

**Depth: [WORKING]** — reframing Part 1's artifact as agent infrastructure.

### Intuition — two directions, both load-bearing

**Reading (memory):** a model starts each session amnesiac about your project. The repo is the *only durable context*: file tree (`ls`, `glob`), file contents (`cat`), and — underused by humans, heavily used by agents — **history**: `git log` says what changed recently and *why* (your §11 messages become model-legible context, which quietly upgrades "write good messages" from etiquette to *prompt engineering for future agents*), and `git diff` says what's currently in flight. **Writing (audit log):** every agent edit lands in the working directory where `git diff` exposes it *before* you accept it, and every accepted change becomes a commit — attributable, timestamped, revertable. Agent goes off the rails? `git restore` / reset to the last good commit: Part 1's time machine is precisely the undo button that makes letting an agent touch your code *safe*. You already committed before risky refactors; an agent session **is** a risky refactor.

### Analogy — the site logbook (and where it breaks)

A construction site logbook: every contractor reads it on arrival (what's done, what's pending, why that wall moved) and writes what they did before leaving. It makes workers *interchangeable across time* — exactly what git history does for you-tomorrow, a collaborator, or an agent-in-ten-minutes. *Where it breaks:* a logbook is testimony (trust the writer); git history is *the work itself* — the commit contains the actual snapshot, content-addressed and tamper-evident (§7). The audit log can't drift from reality because it **is** reality.

### Runnable example — a Claude tool-use loop that reads your repo's history *(preview — taken on faith until Day 22)*

> **Honesty caveats first.** This is a **preview**: the tool-use loop below (`tools=`, `stop_reason`, `tool_result`) is Day 22+ material — today, run it, watch it, and take the plumbing on faith; only the *shell/git part* is expected to be fully legible. It needs `pip install anthropic`, an `ANTHROPIC_API_KEY` in your environment, and API calls **cost money** (cents here). The tool is **read-only by allowlist** — deliberately: a model deciding which shell commands to run must not be handed `rm`/`push`/`commit` on day one (that judgment is Part 3's whole point). `shell=False` + `.split()` is a demo-grade injection guard, not production security.

```python
# repo_historian.py — run from inside ~/projects/journey
# pip install anthropic          (and: export ANTHROPIC_API_KEY=sk-ant-...)
import subprocess

import anthropic

# The agent's ONLY capability: run a shell command from this read-only allowlist.
ALLOWED_PREFIXES: tuple[str, ...] = (
    "git log",
    "git status",
    "git branch",
    "git diff --stat",
    "ls",
)


def run_shell_command(command: str) -> str:
    """Execute an allowlisted read-only command; return stdout+stderr as text."""
    if not command.strip().startswith(ALLOWED_PREFIXES):
        # The model reads this and picks an allowed command instead.
        return f"ERROR: command not allowed. Allowed prefixes: {ALLOWED_PREFIXES}"
    proc = subprocess.run(
        command.split(),            # no shell=True: no pipes/redirection = no injection (demo-grade)
        capture_output=True, text=True, timeout=10,
    )
    output = (proc.stdout + proc.stderr).strip()
    return output[:4000] or f"(no output, exit code {proc.returncode})"


TOOLS = [{
    "name": "run_shell_command",
    "description": (
        "Run a READ-ONLY shell command in the user's git repository and get its "
        "text output. Use it to inspect the repo: 'git log --oneline' for history, "
        "'git status' for current state, 'git branch' for branches, "
        "'git diff --stat' for pending changes, 'ls' for files. "
        "Only these commands are permitted; anything else returns an error."
    ),
    "input_schema": {
        "type": "object",
        "properties": {
            "command": {"type": "string", "description": "The exact command, e.g. 'git log --oneline'"},
        },
        "required": ["command"],
    },
}]

client = anthropic.Anthropic()   # reads ANTHROPIC_API_KEY from the environment

messages: list[dict] = [{
    "role": "user",
    "content": "Look at this repository and summarize in a few sentences: what is it, "
               "what has been worked on recently, and is anything uncommitted?",
}]

MAX_ITERS = 8                    # guard: never loop forever on a confused model
for _ in range(MAX_ITERS):
    resp = client.messages.create(
        model="claude-opus-4-8",
        max_tokens=2048,
        thinking={"type": "adaptive"},
        tools=TOOLS,
        messages=messages,
    )
    if resp.stop_reason != "tool_use":
        break                                       # model is done investigating
    messages.append({"role": "assistant", "content": resp.content})
    results = []
    for block in resp.content:
        if block.type == "tool_use":                # the model chose a command...
            print(f"  [agent runs] {block.input['command']}")
            results.append({
                "type": "tool_result",
                "tool_use_id": block.id,
                "content": run_shell_command(block.input["command"]),  # ...we execute it
            })
    messages.append({"role": "user", "content": results})

for block in resp.content:
    if block.type == "text":
        print("\n" + block.text)
```

```
$ python repo_historian.py
  [agent runs] git log --oneline
  [agent runs] git status
  [agent runs] ls

This is "journey", a learning repository for a 100-day backend and agentic-AI
course. Recent commits show steady prerequisite work: a README, Day P4 progress
notes, a .gitignore for the virtual environment and secrets, and a merged
experiment branch. The working tree is clean — nothing is uncommitted — and
all work is on the main branch.
```

**Why this works — the part you *can* fully read today.** `run_shell_command` is Part 1 in a function: it launches a process (`subprocess.run` = Python's doorway to fork/exec, §"first principles"), captures **stdout and stderr** (§4's streams — captured instead of redirected to files), notes the **exit code**, and returns everything as text. The allowlist check is the harness enforcing what the model may do — note it's the *function*, not the model, that guarantees safety; the model just receives an `ERROR:` string and adapts. **The part on faith until Day 22:** `TOOLS` describes the function to the model (name + when-to-use description + a JSON schema for arguments); the API response's `stop_reason == "tool_use"` means *"I'm not answering yet — run this for me"*; we execute, append the result keyed by `tool_use_id`, and call again; the loop ends when the model stops requesting tools and writes prose; `MAX_ITERS` caps a runaway loop. Watch what the model *chose*: `log` → `status` → `ls` — the same orient-first sequence a sensible human runs in an unfamiliar repo, because the repo really is its only memory.

## Interview & practice (Part 2)

1. In one sentence: what does an agentic coding tool add on top of a plain LLM chat? (A harness that executes the model's chosen commands and feeds results back — hands and eyes.)
2. Which three signals from a command execution does a harness feed back to the model, and where did you meet each in Part 1? (stdout, stderr, exit code — §4 and the fork/exec note.)
3. What's the structural difference between `a | b | c` and an agent chaining three tool calls? (Static human-designed plan vs. per-step model-chosen plan; same composition principle.)
4. Why does the allowlist live in `run_shell_command` rather than in the prompt? (Prompts steer; code enforces. The model can be confused or manipulated; the function cannot.)
5. Name the two directions in which a git repo serves an agent, with one command each. (Memory: `git log` for context; audit/undo: `git diff` review + revert to last commit.)

*Practice — easy:* rerun `repo_historian.py` after making (but not committing) a change — does the summary notice? *Medium:* add `"git show --stat"` to the allowlist and extend the tool description; observe whether the model starts using it. *Hard (thought experiment):* the model asks to run `git log; rm -rf ~` — trace exactly which line of the example stops it, and which *second* line would stop it even if the allowlist were fooled (hint: what does `.split()` + no-`shell` prevent?).

---

# PART 3 — THE BRIDGE: One Skill Set, Two Operators

> Everything here references only Parts 1 and 2 — no new concepts, just the dependency map made explicit.

## The dependency map

```
            YOU (Part 1)                        THE AGENT (Part 2)
                │ types commands                     │ emits tool calls
                ▼                                    ▼
        ┌───────────────────────────────────────────────────┐
        │            THE SHELL — one execution surface       │
        │   command in → fork/exec → stdout / stderr / exit  │
        └───────────────────────────────────────────────────┘
                              │ reads & writes
                              ▼
        ┌───────────────────────────────────────────────────┐
        │   GIT — one memory & audit layer                    │
        │   objects (blobs/trees/commits) + refs + index      │
        │   history = context (log/diff) · commits = undo     │
        └───────────────────────────────────────────────────┘
                              │ push / pull
                              ▼
                    GitHub — the shared meeting point
                    (you ⇄ collaborators ⇄ agents)
```

One surface, two operators. Every arrow in the top half is Part 1 mechanics; every claim about the left-vs-right operator is Part 2. The bridge is that **there is nothing agent-specific in the bottom three boxes** — which is exactly why learning them as a human today was not a detour from agentic AI but the first day of it.

## Three load-bearing correspondences

**1. The exit code is the proto-status-code.** Part 1: programs report success/failure to *other programs* via exit codes, and `&&` / the harness branch on them. Part 2: the agent decides "proceed vs. fix" from exit code + stderr. This caller-readable-outcome pattern is precisely what HTTP status codes will formalize for APIs from Day 8 — you have now met the idea in its oldest form. A tool that always "succeeds" textually while failing semantically confuses an agent exactly the way a `200 OK`-with-error-body will confuse an API client.

**2. Composition requires a shared interface — pipes, then tools.** Pipes (Part 1 §3) compose because every program speaks lines of text; agent tool-chaining (Part 2 §2) composes because every tool speaks structured text. Same theorem, two proofs. When Phase 1 introduces JSON tool schemas, recognize them as `|` with a type system.

**3. Git converts irreversible risk into reversible risk — for both operators.** You commit before a risky refactor (Part 1 §11); the *harness's* safety story for letting a model edit your files is the same commit (Part 2 §3): `diff` to inspect, commit to accept, restore to reject. Corollary you can act on today: **commit before every agent session.** The repo is simultaneously the agent's memory (log/messages as context) and your leash on the agent (diff/restore as review-and-undo) — one artifact, both directions.

## How they fail together

- **Agent misreads the shell → git catches it.** Model hallucinates a flag, misparses stderr, edits the wrong file: the blast radius is the working directory, and `git diff` + `git restore` bound it — *provided you committed first*. Uncommitted work has no safety net under either operator (§2's `rm` lesson, generalized).
- **Git state confuses the agent → the shell reveals it.** A repo with a week of uncommitted drift, meaningless messages (`stuff`, `fix`), or a forgotten in-progress merge feeds the model stale or contradictory context, and its plans degrade accordingly. Repo hygiene (§10–11) is no longer just for humans — a clean `status` and honest `log` are the quality of the agent's *inputs*.
- **The harness is the only real boundary.** Prompts request; allowlists and permissions *enforce* (Part 2 §3's example). Everything dangerous in Part 1 — `rm -rf`, `>` truncation, `push --force` — is exactly what an agent must be structurally prevented from reaching until trust is earned, because the shell executes the confused and the malicious with equal cheerfulness for either operator.

---

# Wrap-up

## Cheat Sheet

```bash
# ── shell ────────────────────────────────────────────────
pwd                     # where am I?
ls -la                  # what's here (long + hidden)?
cd path  ·  cd ..  ·  cd ~
cat file                # print a file
mkdir -p a/b            # make dirs (with parents)
rm file  ·  rm -r dir   # DELETE — permanent, no trash
cmd1 | cmd2             # pipe: stdout → stdin, concurrent
cmd > f   ·  cmd >> f   # redirect: replace · append
cmd 2> errs.txt         # stderr separately;  echo $?  = exit code
man cmd  ·  cmd --help  # the manual · quick summary

# ── git: the daily loop ─────────────────────────────────
git init                       # new repo (creates .git/)
git status                     # ALWAYS run when unsure
git add <files>                # working dir → staging area
git diff  ·  git diff --staged # unstaged edits · staged-vs-HEAD
git commit -m "why, imperative"# staging area → history
git log --oneline              # history, compact

# ── branches & remotes ──────────────────────────────────
git checkout -b name           # create + switch (git switch -c)
git merge name                 # fold `name` into current branch
git branch -d name             # delete merged branch pointer
git clone <url>                # remote → full local copy (origin)
git push  ·  git pull          # upload commits · fetch + merge
git push -u origin main        # first push: set upstream

# ── under the hood (the whole machine) ──────────────────
# objects: blob(file bytes) ← tree(dir listing) ← commit(tree+parent+msg)
# IDs = SHA-1 of content → integrity, dedup, tamper-evident DAG
# branch = ref file with a hash · HEAD = "which ref am I on"
# index = .git/index, the draft snapshot   (see: git cat-file -p <hash>)
```

## Build This — Day P4 Build + the P-phase checkpoint

**Part A — the journey repo (the day's Build).**
1. Create a GitHub repo `journey`; set up SSH keys (`ssh-keygen -t ed25519`, add the `.pub` to GitHub).
2. In WSL2: either `git clone` it into `~/projects/`, or `git init` locally and `git remote add origin ...` (Part 1 §9 shows both).
3. Copy in your P1–P3 builds (fizzbuzz, wordcount, BankAccount modules, the typed API-fetcher). Add a `.gitignore` covering `.venv/`, `__pycache__/`, `*.pyc`, `.env` **before** the first `git add`.
4. Make them your first commits — **one commit per day's work** with imperative messages (`add P1 fizzbuzz and wordcount`), then `git push -u origin main`.
5. Branch drill: `git checkout -b improve-readme`, edit README, commit, `git checkout main`, `git merge improve-readme`, delete the branch, push.

*Definition of done (A):* `git log --oneline` shows ≥3 meaningful commits · the GitHub page shows your files and history · `git status` is clean · `git branch` shows only `main` · `git log --oneline | wc -l` works and you can explain every stage of that pipeline.

**Part B — the P-phase checkpoint (prove you're ready for Day 1).**
From a blank file, **without looking anything up** — no notes, no search, no AI: write a type-hinted Python program containing a class and a function that reads a JSON file, handles a missing file gracefully (no traceback — a clean message), and prints a result. Then commit and push it, also from memory. A shape that qualifies (spec, not a template to copy):

```python
# checkpoint.py — e.g.: class Library with load(path: str) -> list[Book];
# reads books.json; FileNotFoundError → friendly message + empty library;
# prints a count/summary. Then:
#   $ python checkpoint.py           → prints a result, no traceback
#   $ mv books.json books.bak && python checkpoint.py   → graceful message
#   $ git add checkpoint.py && git commit -m "..." && git push
```

*Definition of done (B):* runs cleanly with the JSON present; degrades gracefully with it absent; `mypy checkpoint.py` passes; the commit exists on GitHub — **and you did not look anything up.** If you reached for a reference on Python, repeat the weak P-day; if on git, redo Part A tomorrow from memory. If both flowed — you're ready for Day 1.

## Active Recall & Self-Test *(answer from memory — retrieval builds the memory)*

1. Terminal vs. shell — what does each actually do?
2. `>` vs `>>` vs `|` — one sentence each, including what `>` does to an existing file *before* the command runs.
3. Draw the three zones (working dir / staging / history) and label which command moves work across each boundary — then place `git diff` and `git diff --staged` on it.
4. What are the three git object types, what does each contain, and what is an object's name derived from?
5. What is a branch, *physically*? What is HEAD? Why is branching instant?
6. Why did distributed version control beat centralized? Three concrete SVN pains, three git answers.
7. What two channels + one number does a harness feed back to an agent after running a command, and what does the agent do with the number?
8. Why must `.gitignore` exist *before* the first `git add`, and what do you do if a secret was already pushed?

**Teach-back prompt (60 seconds, aloud, to an imaginary colleague):** *"Explain what actually happens inside `.git` when I run `git add file.py` and then `git commit -m 'msg'` — using the words blob, index, tree, commit, ref, and HEAD."* If you stumble, reread §7's under-the-hood and run the `cat-file` transcript again with your own hands.

## Spaced-Repetition Flashcards

| Q | A |
|---|---|
| Command → "where am I / what's here / go there"? | `pwd` / `ls` / `cd` |
| What does `a \| b` connect, and do the programs run in sequence? | a's stdout → b's stdin; no — concurrently, with kernel backpressure |
| `>` vs `>>`? | replace (truncates first) vs append |
| The three standard streams and their fd numbers? | stdin 0, stdout 1, stderr 2 |
| Exit code convention? | 0 = success; nonzero = failure; last one in `$?` |
| `git add` does what, to where? | copies file content into the staging area (the index — `.git/index`) |
| `git diff` vs `git diff --staged`? | working dir↔index vs index↔HEAD |
| A git branch is, physically…? | a ref file containing one commit hash |
| HEAD is…? | the file recording which ref (branch) you're currently on |
| The three object types in `.git/objects`? | blob (file bytes), tree (directory listing), commit (tree + parent + author + message) |
| Why is git history tamper-evident? | IDs are content hashes and each commit hashes its parent — change one, all descendants change |
| Fast-forward merge means…? | target branch had no new commits, so its pointer just slides forward; no merge commit |
| `git pull` decomposes into…? | `git fetch` (download) + `git merge` (integrate) |
| Why was your push rejected? | remote has commits you lack — pull (fetch+merge) first, then push fast-forwards |
| `.gitignore` doesn't affect which files? | already-tracked ones (evict with `git rm --cached`) |
| An agentic coding tool's "hands" are…? | shell commands executed by a harness; results (stdout/stderr/exit code) fed back as text |
| A pipe is to programs what ___ is to agent tools? | chaining tool calls — composition over a shared text interface |
| The safety mechanism that makes agent edits reversible? | commit before the session; `git diff` to review; restore/reset to reject |

## Primary Sources *(verify against these — Rule 7)*

- **Pro Git** (Scott Chacon & Ben Straub) — free at <https://git-scm.com/book>. Ch. 1–3 = today's Part 1 git material; **Ch. 10 "Git Internals"** = §7's object model, from the source.
- **git-scm reference & man pages** — <https://git-scm.com/docs>, or `git help <cmd>` / `man git-commit` offline: the authoritative flag-level truth.
- **GNU Coreutils manual** — <https://www.gnu.org/software/coreutils/manual/> (plus local `man ls`, `man rm`, …): the canonical documentation for the shell utilities.
- **Bash Reference Manual** — <https://www.gnu.org/software/bash/manual/> for pipes, redirection, and exit-status semantics as specified, not folklore.
- **Anthropic tool-use docs** — <https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview> for the Part-2 preview's real contract (revisited properly at Day 22). *Model IDs and API surface drift — verify before relying.*

## Key Takeaways & Summary

**10-second version.** The shell runs text commands and reports via stdout/stderr/exit-code; git snapshots your project as a content-addressed history you can diff, branch, merge, and sync. Agentic coding tools are models driving *these same two tools* through a harness — learn them once, use them as human and supervisor both.

**1-minute version.** The shell trades discoverability for precision, automation, and composition: programs are verbs, pipes chain them concurrently over text streams, redirection points output anywhere, and exit codes make success machine-readable. Git solves "what changed, why, and can I get it back": `add` drafts a snapshot in the staging area, `commit` seals it into an immutable object chain, branches are pointer files so experiments are free, merges combine divergent lines (conflicts = git honestly refusing to guess), and remotes/GitHub sync full histories between complete copies. Under the hood it's three object types + ref files + an index — small enough to hold entirely in your head. Part 2's payoff: an agent is a model emitting these very commands through a permission-enforcing harness, reading the repo as its memory and leaving commits as its audit trail; your git hygiene is now also the agent's context quality, and your pre-session commit is its undo button.

**5-minute version.** Reconstruct the day from five questions. *(1) How do I command a machine precisely?* A shell: reads a line, fork/execs the program, wires three standard streams, collects an exit code — composition via `|` (kernel-buffered, concurrent, backpressured) and `>`/`>>` (same stdout, different sink; `>` truncates first). *(2) How do I keep trustworthy history?* Git: the working-dir→index→history flow; the index existing so commits can be *composed*; `status`/`diff`/`diff --staged` as the three-zone instruments. *(3) Why trust it?* Content-addressing: blob/tree/commit objects named by SHA-1 of their contents → integrity, dedup, and a tamper-evident parent-linked DAG; branches and HEAD are pointer files, hence free branching; verify all of it yourself with `git cat-file -p`. *(4) How do people (and agents) share?* Full-history clones syncing objects+refs via push/pull; rejected pushes protect concurrent work; merge machinery (fast-forward vs. three-way, conflict markers) integrates divergence; `.gitignore` keeps regenerables and secrets out — permanently-public history means a pushed secret is a rotated secret. *(5) What does this have to do with agents?* Everything: the agent's actions are shell commands, its feedback is stdout/stderr/exit-code, its plan is a dynamically-chosen pipeline, its memory is `log`/`diff`, its accountability is the commit, and its limits are the harness's allowlist — one skill set, two operators, which is why P4 is the last prerequisite and not a detour.

**Expert summary.** Unix's `fork/exec` + fd-based stream model gives a compositional execution surface where processes are typed-by-convention text filters and the kernel provides free concurrency and flow control; git layers a content-addressed Merkle DAG (blobs/trees/commits) with mutable refs and a staging index over it, yielding O(1) branching, cryptographic history integrity, and merge-by-common-ancestor. Agentic coding systems are LLM policies emitting actions against exactly this surface, with the harness as the capability boundary and the repository serving dual duty as retrieved context and reversible audit log — so shell/git fluency is the transferable substrate for both operating and supervising them.

## Mental Models

- **The city map** — the filesystem is one tree; you always stand somewhere (`pwd`); paths are directions, absolute or relative.
- **The assembly line with a bounded belt** — pipes: concurrent stations, kernel buffer as the belt, backpressure when full. Recurs as message queues in Phase 2.
- **The mail chute** — programs write to abstract slots (fd 1/2); the shell re-routes the chutes before launch. Computation decoupled from destination.
- **The photographer's table** — working dir = studio, index = arranged shot, commit = shutter click into a permanent album you can climb back into.
- **Save points with infinite slots + branching timelines** — commits remove the fear from experiments; branches are parallel storylines; merge is the timelines reconciling.
- **Everything is a tiny file** — branch = 41-byte hash file, HEAD = one-line pointer, index = a table. When git feels magical, `cat` the file.
- **The remote contractor over text chat** — the agent: capable, literal, memoryless beyond the transcript, and bounded by what the messenger (harness) agrees to relay.

## Mnemonics

- **Daily git heartbeat:** **S-A-D-C-L-P** — *"**S**tatus, **A**dd, **D**iff, **C**ommit, **L**og, **P**ush"* → "**S**mart **A**gents **D**on't **C**ommit **L**azy **P**atches."
- **The object chain:** **B-T-C** — *"**B**ytes, **T**able-of-contents, **C**heckpoint"* (blob → tree → commit), each pointing at the previous.
- **Streams by number:** *"0 in, 1 out, 2 trouble"* (stdin 0, stdout 1, stderr 2).
- **Redirection danger:** *"one arrow annihilates, two arrows append."*
- **Before any agent session:** *"commit, then permit."*

## Common Confusions *(each has bitten thousands before you)*

| Confusion | Resolution |
|---|---|
| terminal = shell | Terminal draws text; the shell (bash) interprets commands. You can swap shells inside the same terminal. |
| git = GitHub | git is the local tool; GitHub is one hosting service for git repos. git works forever with no GitHub. |
| `git add` marks a file "to be included" | It copies the file's contents **at that moment** into the index. Edit again → must `add` again. |
| plain `git diff` shows all my changes | It shows only *unstaged* ones (working↔index). Staged changes: `git diff --staged`. |
| a branch contains/copies files | A branch is a pointer file with one commit hash. The objects are shared; switching branches re-materializes the working dir. |
| `git pull` just downloads | It's fetch **+ merge** — it can create merge commits and surface conflicts. |
| deleting a committed secret in the next commit removes it | History is immutable and content-addressed; the old commit still holds it. Rotate the secret. |
| `rm` is like the Recycle Bin | It unlinks immediately and permanently. Git-committed files are your only real undo. |
| `.gitignore` hides tracked files | It only prevents *untracked* files from being added. Tracked ones need `git rm --cached` first. |
| merge conflict = something broke | It's git *correctly* refusing to guess between two human edits to the same lines. Read markers, decide, add, commit. |
| the model in Part 2 "has access to my computer" | It emits text; the **harness** decides what to execute (allowlist!). Capability lives in code, not in the model. |
