# POSIX Threads and Mutexes in C

Practical guide to `pthread` basics plus **under-the-hood behavior** (scheduler, atomic lock path, futex wait/wake, and visibility guarantees through lock/unlock).

## Open the guide

<<<<<<< HEAD
- `threads-in-c.html` — full visual resource page
- `README.md` — compact text reference
=======
🔗 **[View the full resource page →](https://nait-sfi.github.io/Threads-in-C-POSIX-pthreads)**
>>>>>>> 874aad398b31204621ceb10b6a1770507d90e1cc

## Core model

Inside one process:

- **Shared across threads:** code, heap, globals/statics, file descriptors
- **Per-thread private state:** stack, registers, program counter, thread-local data

Linux note: `pthread_create()` goes through libpthread/glibc and reaches a `clone()`-family kernel path to create a schedulable thread that shares process memory.

## Thread APIs

| API | Purpose | Under the hood | Common mistake |
|---|---|---|---|
| `pthread_create` | start new thread | stack/TCB setup + kernel registration | passing pointer to short-lived stack data |
| `pthread_join` | wait for thread exit | caller blocks until target terminates | join cycles / joining same thread twice |
| `pthread_detach` | auto-reclaim on exit | marks thread non-joinable | detach then trying to join |

## Why races happen

`counter++` is logically **load + add + store**, not one indivisible operation.
If two threads interleave those steps, one update can be overwritten.

```c
int counter = 0; // shared
// Unsynchronized counter++ in multiple threads => data race
```

## Mutex APIs

| API | Purpose | Under the hood | Common mistake |
|---|---|---|---|
| `pthread_mutex_init` / `destroy` | setup/teardown lock | initializes lock state + attributes | destroying while still in use |
| `pthread_mutex_lock` / `unlock` | protect critical section | fast userspace atomic path; contended path sleeps via futex | missing unlock on error path |
| `pthread_mutex_trylock` | non-blocking lock attempt | immediate atomic attempt only | busy-looping without backoff |

Memory ordering: successful lock/unlock creates synchronization so writes in one critical section become visible to the next lock holder.

## Minimal safe pattern

```c
pthread_mutex_t lock = PTHREAD_MUTEX_INITIALIZER;

pthread_mutex_lock(&lock);
/* critical section touching shared state */
pthread_mutex_unlock(&lock);

pthread_mutex_destroy(&lock);
```

## Build

```bash
gcc -Wall -Wextra -pthread threads.c -o threads
```

`-pthread` enables the right thread-related defines and links the pthread runtime.
