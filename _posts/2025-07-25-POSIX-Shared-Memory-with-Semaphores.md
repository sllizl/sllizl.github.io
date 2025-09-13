---
title:  POSIX Shared Memory with Semaphores
date: 2025-07-25 17:30:23 +0900
categories: [Blog, IPC]
tags: [posix ipc/shm/semaphore]     # TAG names should always be lowercase
author: sllizl
comment: false
---

# Shared Memory

When two processes need to exchange data efficiently, one of the best options on Linux/Unix systems is **shared memory**. Instead of sending data through pipes or sockets (which require copies between kernel and user space), shared memory allows processes to map the same region of memory and communicate directly.

# Using POSIX Shared Memory with Semaphores in C

Linux systems provide two separate Apis for shared memory: **the legacy
system V Api and the more recent POSIX one.** **these Apis should never be 
mixed in a single application, however.** A downside of the posix approach 
is that features are still in development and dependent upon the 
installed kernel version, which impacts code portability. For example, 
the POSIX API, by default, implements shared memory as a memory-mapped 
file: for a shared memory segment, the system maintains a backing file 
with corresponding contents. shared memory under posix can be configured 
without a backing file, but this may impact portability. my example uses 
the posix Api with a backing file, which combines the benefits of memory 
access (speed) and file storage (persistence).

- But there’s a catch: **synchronization**. If one process is writing while the other is reading, race conditions can occur. That’s where POSIX semaphores come in.

We’ll walk through a simple C program that demonstrates how to set up a shared memory segment with semaphores to synchronize read/write access.

The shared-memory example has two programs, named `writer.c` and `reader.c`,
and uses a semaphore to coordinate their 
access to the shared memory.
Whenever shared memory comes into the picture 
with a writer, whether in 
multi-processing or multi-threading, so does 
the risk of a memory-based 
race condition; hence, the semaphore is used
to coordinate (synchronize) 
access to the shared memory.

## The Header File: shm_sem.h

Before diving into the writer and reader code, let’s define a shared header file that contains all the constants, includes, and data structures needed for both processes.

```c
// shm_sem.h
#ifndef SHM_SEM_H
#define SHM_SEM_H

#include <fcntl.h>
#include <sys/mman.h>
#include <semaphore.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#define SHM_NAME "/my_shm"
#define SEM_EMPTY_NAME "/sem_empty"
#define SEM_FULL_NAME  "/sem_full"
#define SHM_SIZE 1024

typedef struct {
    char buffer[SHM_SIZE];
} shm_data;

#endif
```

### Key Points:

1. **Header guards**
   The `#ifndef ... #define ... #endif` pattern ensures the file is included only once during compilation.

2. **System headers**

   * `<fcntl.h>`: for `O_CREAT`, `O_RDWR`, etc.
   * `<sys/mman.h>`: for `mmap`/`shm_open`.
   * `<semaphore.h>`: for POSIX semaphores.
   * `<unistd.h>`, `<stdio.h>`, `<stdlib.h>`, `<string.h>`: standard C library headers for I/O, memory, and utility functions.

3. **Definitions**

   * `SHM_NAME`: the identifier for the shared memory object. It must start with a `/` and will typically live under `/dev/shm/`.
   * `SEM_EMPTY_NAME` and `SEM_FULL_NAME`: unique names for the two semaphores.
   * `SHM_SIZE`: the size of the shared memory buffer (1 KB in this example).

4. **Shared data structure**
   The `shm_data` struct holds the buffer where messages are written. Both processes will map this structure into their address spaces and use it for communication.


---
## Writer

To write some data to a shared memory space, basic steps are as follows:

### Step 1: Shared Memory Setup

We start by creating (or opening) a shared memory object with `shm_open`:

```c
shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
```

* `SHM_NAME` is a string identifier (e.g., `"/my_shm"`).
* `O_CREAT | O_RDWR` means "create if it doesn’t exist, and open for read/write".
* `0666` sets the permissions (read/write for everyone).

After creating it, we must set the size:

```c
ftruncate(shm_fd, sizeof(shm_data));
```

Finally, we map it into our process’s address space:

```c
data = mmap(0, sizeof(shm_data), PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0);
```

At this point, both processes can access the same memory region.

---

### Step 2: Creating Semaphores

We use two semaphores to implement a **producer-consumer model**:

```c
sem_t *sem_empty = sem_open(SEM_EMPTY_NAME, O_CREAT, 0666, 1);
sem_t *sem_full  = sem_open(SEM_FULL_NAME,  O_CREAT, 0666, 0);
```

* `sem_empty` starts at 1 → the buffer is empty, so the writer can write.
* `sem_full` starts at 0 → nothing is written yet, so the reader must wait.

This ensures:

* The **writer** waits if the buffer is full.
* The **reader** waits if the buffer is empty.

---

### Step 3: Writing Data

Inside the main loop, the writer does:

```c
sem_wait(sem_empty);  // wait until buffer is empty
snprintf(data->buffer, SHM_SIZE, "Message %d from writer", i);
printf("Writer: wrote '%s'\n", data->buffer);
sem_post(sem_full);   // signal that data is available
```

This means:

1. Wait until the buffer is ready.
2. Write a new message into shared memory.
3. Signal the reader that data is available.

We also add a `sleep(1)` to simulate some delay.


> #### **What `empty` and `full` do**
>    In the classic **single-slot producer-consumer problem**:
>
>    | Semaphore | Purpose                                                                   |
>    | --------- | ------------------------------------------------------------------------- |
>    | `empty`   | Indicates the buffer is **empty / ready for writing**. Initial value = 1. |
>    | `full`    | Indicates the buffer is **full / ready for reading**. Initial value = 0.  |
>
> *#### How they work*
>
>    **Writer (Producer):**
>
>    ```c
>    sem_wait(sem_empty);  // wait until buffer is empty
>    write_to_buffer();
>    sem_post(sem_full);   // signal buffer now has data
>    ```
>
>    **Reader (Consumer):**
>
>    ```c
>    sem_wait(sem_full);   // wait until buffer has data
>    read_from_buffer();
>    sem_post(sem_empty);  // signal buffer is now empty
>    ```
>
>
>
> *#### Why this works*
>
>    * **`empty`** ensures the writer **does not overwrite** unread data.
>
>    * If the buffer is already full (`empty = 0`), `sem_wait(empty)` blocks the writer until the reader consumes it and calls `sem_post(empty)`.
>
>    * **`full`** ensures the reader **does not read garbage**.
>
>    * If the buffer is empty (`full = 0`), `sem_wait(full)` blocks the reader until the writer produces data and calls `sem_post(full)`.
>
>    * Together, they **coordinate access** to a shared single-slot buffer safely across processes.
>
>    ---
>
> *#### Timing Diagram (Single-slot buffer)*
>
>    ```
>    Time → 
>
>    Writer:  | write M0 | wait for empty | write M1 | wait for empty | write M2 ...
>    Buffer:  |  M0      | blocked?       | M1       | blocked?       | M2
>    Reader:  | wait full| read M0        | wait full| read M1        | wait full | read M2
>    Sem empty: 1 -------->0------>1-------->0-------->1
>    Sem full:  0 -------->1------>0-------->1-------->0
>    ```
>
>    **Explanation of the arrows:**
>
>    1. Initially, `empty = 1`, `full = 0`.
>    2. Writer does `sem_wait(empty)` → `empty-- = 0`, writes data.
>    3. Writer calls `sem_post(full)` → `full++ = 1`, signals reader.
>    4. Reader does `sem_wait(full)` → `full-- = 0`, reads data.
>    5. Reader calls `sem_post(empty)` → `empty++ = 1`, signals writer.
>
>    * This **ping-pong** continues for every data item.
>
>    ---
>
> *#### Note*
>
>    * These semaphores **don’t hold the data**, only control access and order.
>
>    * Single-slot model: `empty = 1` → ready to write, `full = 1` → ready to read.
>
>    * Multi-slot (buffer with N slots):
>
>    * `empty = N` initially, `full = 0` initially.
>    * Writer consumes `empty`, produces `full`.
>    * Reader consumes `full`, produces `empty`.
>



---

### Step 4: Cleanup

After finishing, we clean up resources:

```c
munmap(data, sizeof(shm_data));
close(shm_fd);
sem_close(sem_empty);
sem_close(sem_full);
```

> ⚠️ Note: In production, you would also call `shm_unlink()` and `sem_unlink()` to remove the named shared memory and semaphores once you are sure no process will use them anymore. This prevents resource leaks.

---

### Full Example (Writer)

```c
#include "shm_sem.h"

int main() {
    int shm_fd;
    shm_data *data;

    shm_fd = shm_open(SHM_NAME, O_CREAT | O_RDWR, 0666);
    ftruncate(shm_fd, sizeof(shm_data));

    data = mmap(0, sizeof(shm_data), PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0);

    sem_t *sem_empty = sem_open(SEM_EMPTY_NAME, O_CREAT, 0666, 1);
    sem_t *sem_full  = sem_open(SEM_FULL_NAME,  O_CREAT, 0666, 0);

    for (int i = 0; i < 5; i++) {
        sem_wait(sem_empty); // down

        snprintf(data->buffer, SHM_SIZE, "Message %d from writer", i);
        printf("Writer: wrote '%s'\n", data->buffer);

        sem_post(sem_full); // post
        sleep(1);
    }

    munmap(data, sizeof(shm_data));
    close(shm_fd);
    sem_close(sem_empty);
    sem_close(sem_full);

    // remove objects
    shm_unlink(SHM_NAME);
    sem_unlink(SEM_EMPTY_NAME);
    sem_unlink(SEM_FULL_NAME);

    return 0;
}
```

---

### Conclusion

This small example demonstrates how **POSIX shared memory** can be combined with **semaphores** to build safe, efficient inter-process communication (IPC).

* Shared memory enables fast, direct data exchange.
* Semaphores ensure proper synchronization between producer and consumer.

With these building blocks, you can implement more complex IPC systems, such as message queues, logging systems, or real-time data streams.


---

## Reader

### Full Example (Reader)
```c
#include "shm_sem.h"

int main() {
    int shm_fd;
    shm_data *data;

    // Open shared memory (writer already created it)
    shm_fd = shm_open(SHM_NAME, O_RDWR, 0666);
    if (shm_fd == -1) {
        perror("shm_open");
        exit(1);
    }

    // Map it
    data = mmap(0, sizeof(shm_data), PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0);
    if (data == MAP_FAILED) {
        perror("mmap");
        exit(1);
    }

    // Open semaphores
    sem_t *sem_empty = sem_open(SEM_EMPTY_NAME, 0);
    sem_t *sem_full  = sem_open(SEM_FULL_NAME, 0);

    if (sem_empty == SEM_FAILED || sem_full == SEM_FAILED) {
        perror("sem_open");
        exit(1);
    }

    // Read loop
    for (int i = 0; i < 5; i++) {
        sem_wait(sem_full); // wait until data is available
        printf("Reader: read '%s'\n", data->buffer);
        sem_post(sem_empty); // signal that buffer is now empty (writer can write)
    }

    // Cleanup
    munmap(data, sizeof(shm_data));
    close(shm_fd);
    sem_close(sem_empty);
    sem_close(sem_full);

    return 0;
}
```

---

### Step 1 — `shm_open(...)`

* We call `shm_open(SHM_NAME, O_RDWR, 0666)` **without** `O_CREAT`.

  * This means the reader expects the shared memory object to already exist (created by the writer).
  * If the writer hasn’t created it yet, `shm_open` will fail and the program should handle that (here we `perror` + exit).
* `O_RDWR` is used because the reader maps the region with read and write permissions (`PROT_READ|PROT_WRITE`), matching the writer’s usage.

**Practical note:** start the writer first (or handle the failure and retry) so the reader can open the object.

---

### Step 2 — `mmap(...)`

* `mmap(NULL, sizeof(shm_data), PROT_WRITE | PROT_READ, MAP_SHARED, shm_fd, 0)` maps the same physical memory into the reader’s virtual address space.
* `MAP_SHARED` ensures writes from one process are visible to all other processes that mapped the same object.
* The map size must match the size the writer set with `ftruncate()` — otherwise the mapping may be smaller/invalid.

---

### Step 3 — `sem_open(...)`

* We open two named semaphores with `sem_open(name, 0)` (no `O_CREAT`): `sem_empty` and `sem_full`.

  * Opening with `0` attempts to open existing semaphores created by the writer.
* If `sem_open` returns `SEM_FAILED`, it indicates the semaphore does not exist or some other error — the program logs and exits.

**Semantic reminder:**

* `sem_full` signals “there is data to read” (initially 0).
* `sem_empty` signals “buffer is empty, writer may write” (initially 1).

---

### Step 4 — The read loop (`sem_wait` / `sem_post`)

```c
sem_wait(sem_full);             // block until writer has posted
printf("Reader: read '%s'\n", data->buffer);
sem_post(sem_empty);            // allow writer to write again
```

* `sem_wait(sem_full)` blocks the reader until the writer has written data and called `sem_post(sem_full)`.
* After `sem_wait` returns, the reader can safely read `data->buffer`. The semaphore-based synchronization guarantees visibility: the `sem_post` on the writer side and `sem_wait` here act as synchronization points (they provide the necessary memory ordering so you see the most recent write).
* After reading, the reader calls `sem_post(sem_empty)` to let the writer know the buffer is free.

**Why this is safe:** the semaphore operations provide the producer–consumer synchronization so you don’t need extra memory fences or `msync()` for correctness in this pattern.

---

### Step 5 — Cleanup

* `munmap(data, ...)` and `close(shm_fd)` release this process’s mapping and file descriptor.
* `sem_close(...)` closes the semaphore handles in this process.

**Important:** the reader does **not** call `shm_unlink()` or `sem_unlink()` here because unlinking removes the named kernel object system-wide. Typically the creator (writer) or a dedicated cleanup process should `unlink` when the resource must be permanently removed. If readers start unlinking, other processes may fail to open the objects.

---
Run order:

1. start the writer (it creates the shared memory and semaphores),
2. then start the reader.

Example output (when paired with the writer from the tutorial):

```
Reader: read 'Message 0 from writer'
Reader: read 'Message 1 from writer'
...
```

---

### Small improvements & robustness tips

* **Avoid blocking forever:** use `sem_timedwait()` or handle signals (`EINTR`) if you don’t want the reader to hang indefinitely.
* **Check sizes:** confirm `mmap` size matches the writer’s `ftruncate()` size, or exchange a version/size field in the shared struct.
* **Handle startup race:** if the reader may start before the writer, implement a retry loop with backoff for `shm_open`/`sem_open` or create a small handshake protocol.
* **Multiple readers/writers:** the two-semaphore single-slot pattern works for single producer + single consumer. For multiple producers/consumers you’ll need counting semaphores (or a ring buffer with head/tail indices plus atomic operations).
* **Signal handling / cleanup:** add signal handlers to close and unlink resources cleanly on Ctrl-C.

