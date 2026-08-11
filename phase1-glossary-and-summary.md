# Phase 1: Terms & What We Built

## The core terms

**Namespace** — A way to give a process its own private view of something the
whole system normally shares. A PID namespace gives it its own process list.
A UTS namespace gives it its own hostname. Namespaces don't limit *how much*
a process can use — just what it can *see*.

**`unshare()`** — The system call that creates a new namespace. Important
quirk we ran into: `unshare(CLONE_NEWPID)` doesn't move the current process
into the new PID namespace — only processes it creates *after* the call land
in it. That's why the sandbox has to `fork()` right after unsharing.

**`fork()` / `exec()`** — `fork()` splits the current process into two
identical copies (parent and child). `exec()` then replaces one of those
copies with a different program entirely. Together: "make a copy, then turn
the copy into something else." This is how the sandbox hands off to
`/bin/bash` (or later, an actual PR's build command).

**User namespace** — A namespace specifically for user/group IDs. It lets an
unprivileged user appear as "root" (UID 0) *inside* the namespace, while
still being a nobody outside it. This is what let us drop `sudo` for the
namespace setup itself.

**Capabilities** — Linux splits "what root can do" into ~40 individual
permissions (mount filesystems, trace other processes, bind low ports,
etc.) instead of one all-or-nothing switch. A process can hold some, all, or
none of them, independent of whether its UID is 0.

**seccomp** — A filter on which *system calls* a process is even allowed to
make, regardless of what capabilities it holds. Think of capabilities as
"what you're allowed to do" and seccomp as "which tools you're allowed to
pick up at all."

**cgroups** ("control groups") — Kernel-enforced limits on *resources*:
how much memory, how many processes, how much CPU. Unlike the three above,
cgroups don't care what a process can see or do — only how much of the
machine it can consume.

## What we built, in order

Each layer was added on top of the last, and each was independently proven
to work — not just written and assumed correct.

1. **Namespaces (raw `unshare()` calls, no libraries)**
   PID, mount, UTS, IPC, and user namespaces, built from scratch via
   `ctypes`. Verified: inside the sandbox, `hostname` shows `sandboxed` and
   `ps aux` shows only the sandbox's own two processes — nothing from the
   host.

2. **Capabilities (via the `python-prctl` library)**
   Strips every Linux capability from the process right before it hands off
   to the real command. Verified: even as `root` inside the sandbox, `mount`
   fails with "Operation not permitted."

3. **seccomp (via the `pyseccomp` library)**
   Loads an allow-list of syscalls, built by tracing what a real bash
   session actually uses. Anything not on the list is blocked outright.
   Verified: `ptrace` — a syscall real debugging/tracing tools rely on —
   fails immediately, before capabilities even get checked.

4. **cgroups (via `systemd-run`, no code — a wrapper around the whole thing)**
   Caps memory, process count, and CPU. Verified: a fork-bomb-style loop
   hit the process cap cleanly, and a 300MB allocation under a 256MB memory
   cap got killed once swap was also capped (initially it "succeeded" by
   quietly swapping to disk — a real gotcha, not a bug in our code).

## The shape of it

Each layer restricts what's available to the next one. That's why order
matters: namespaces first (nothing else makes sense without isolation),
then capabilities and seccomp (both must apply *before* the real command
runs), with cgroups wrapped around the outside the whole time.

Phase 1 is done. Next up: Phase 2, the GitHub webhook receiver.
