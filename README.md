# Threads in C — POSIX pthreads

> Session resource — deep-dive into POSIX threads, concurrency, and mutexes

A detailed reference on **POSIX threads** (`pthreads`) in C — covering what threads are, how they work under the hood at the OS level, how to create them, and how to protect shared data with mutexes.

🔗 **[View the full resource page →](https://nait-sfi.github.io/Threads-in-C-POSIX-pthreads)**

---

## What is a thread?

A thread is the smallest unit of execution scheduled by the OS. Every process starts with one thread (main). Additional threads **share** the same address space — same code, heap, and globals — but each owns its private:

- **Stack** (typically 8 MB, allocated by pthreads)
- **Program Counter** (where it currently is in the code)
- **Register set** (current CPU state)

```
Index formulas (not applicable here — threads don't use index math like heaps)
But the key mental model:

PROCESS
├── Shared: code (.text), heap, globals, file descriptors
├── Thread 1: stack + PC + registers
├── Thread 2: stack + PC + registers
└── Thread N: stack + PC + registers
```

---

## Under the hood

On Linux, `pthread_create()` calls the `clone()` syscall with flags to **share** the address space instead of copying it. The kernel creates a new `task_struct` (same structure as a process) with its own kernel stack and user-space stack.

```
pthread_create() → glibc wrapper → clone() syscall → kernel: new task_struct → scheduler
```

Modern mutexes are implemented as **futexes** (fast userspace mutexes) — lock/unlock is a cheap atomic memory operation when uncontested, and only falls into the kernel when threads actually have to wait.

---

## Thread functions

| Function | Returns | Description |
|---|---|---|
| `pthread_create()` | 0 / errno | Creates a new thread running a given function |
| `pthread_join()` | 0 / errno | Waits for a thread to finish and reclaims resources |
| `pthread_detach()` | 0 / errno | Auto-release resources on finish (no join needed) |
| `pthread_exit()` | — | Terminates the calling thread with a return value |
| `pthread_self()` | pthread_t | Returns the ID of the calling thread |

---

## Mutex functions

| Function | Returns | Description |
|---|---|---|
| `pthread_mutex_init()` | 0 / errno | Initializes a mutex |
| `pthread_mutex_destroy()` | 0 / errno | Releases mutex resources (must be unlocked first) |
| `pthread_mutex_lock()` | 0 / errno | Acquires mutex — blocks if already held |
| `pthread_mutex_unlock()` | 0 / errno | Releases mutex — wakes up waiting threads |
| `pthread_mutex_trylock()` | 0 / EBUSY | Tries to lock without blocking |

---

## Thread lifecycle

```
NEW → RUNNABLE → RUNNING → TERMINATED
                    ↕
                 BLOCKED (waiting for mutex / I/O)
```

- **NEW** — `pthread_create()` called, not yet scheduled
- **RUNNABLE** — ready, waiting for a CPU slot
- **RUNNING** — executing on a core (can be preempted at any moment)
- **BLOCKED** — sleeping, waiting for a mutex or I/O
- **TERMINATED** — returned or `pthread_exit()` called; freed after `pthread_join()`

---

## Race conditions

A race condition happens when two threads access **shared mutable data** concurrently without synchronization. The result is undefined and changes every run.

```c
// BROKEN — counter++ is 3 CPU instructions (LOAD, ADD, STORE)
// The scheduler can interrupt between any of them
int counter = 0;
void *increment(void *arg) {
    for (int i = 0; i < 1000000; i++)
        counter++;  // data race!
    return NULL;
}
// Expected: 2000000 — Actual: random, less every time
```

---

## Mutex usage

```c
pthread_mutex_t lock;
pthread_mutex_init(&lock, NULL);

// Critical section — only one thread enters at a time
pthread_mutex_lock(&lock);
counter++;
pthread_mutex_unlock(&lock);

pthread_mutex_destroy(&lock);
```

### Deadlock prevention

- Always acquire multiple locks in the **same global order**
- Keep critical sections as **short as possible**
- Never call unknown functions while holding a lock
- One mutex per shared resource

---

## Compile

```bash
gcc -Wall -Wextra -pthread your_file.c -o your_program
```

The `-pthread` flag both sets the correct defines **and** links `libpthread`.

---

## Files

| File | Description |
|---|---|
| `threads_full.c` | Full example — 5 threads + mutex + shared counter |
| `threads-in-c.html` | Standalone resource page (GitHub Pages) |
| `README.md` | This file |
