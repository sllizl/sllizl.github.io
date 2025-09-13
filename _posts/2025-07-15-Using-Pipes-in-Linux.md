---
title:  Using Pipes in Linux
date: 2025-07-15 16:04:35 +0900
categories: [Blog, IPC]
tags: [pipes]     # TAG names should always be lowercase
author: sllizl
comment: false
---


---

# Anonymous Pipes

An **anonymous pipe** is one of the simplest forms of inter-process communication (IPC) in Linux. It provides a unidirectional communication channel between related processes, typically a parent and its child created with `fork()`.

---

## 1. How It Works

* Created with the `pipe()` system call, which returns two file descriptors:

  * `fd[0]`: the **read end**
  * `fd[1]`: the **write end**
* The kernel maintains a temporary buffer. Data written to `fd[1]` can be read from `fd[0]`.
* Pipes do not have a name in the filesystem and disappear once all processes close their ends.

---

## 2. Example Code

```c
#include <unistd.h>
#include <stdio.h>
#include <string.h>

int main() {
    int fd[2];
    pipe(fd);

    if (fork() == 0) {

        // Child: writer
        close(fd[0]); // close unused read end
        const char *msg = "Hello from child";
        write(fd[1], msg, strlen(msg));
        close(fd[1]);
        
    } else {

        // Parent: reader
        char buf[100];
        close(fd[1]); // close unused write end
        int n = read(fd[0], buf, sizeof(buf)-1);
        buf[n] = '\0';

        printf("Parent read: %s\n", buf);
        close(fd[0]);
    }
    return 0;
}
```

---

## 3. Diagram

Here’s a simple illustration of how anonymous pipes connect processes:

```
           pipe(fd)
        ┌───────────────┐
        │   Kernel       │
        │   Buffer       │
        └──────┬────────┘
               │
      ┌────────┴───────────┐
      │                    │
  fd[1] (write)        fd[0] (read)
   Child Process        Parent Process
```

* The **child process** writes data into the pipe using `fd[1]`.
* The **parent process** reads data from the pipe using `fd[0]`.
* The kernel buffer sits in between, acting like a queue.

---

## 4. Some Details

* Pipes are **blocking** by default:

  * `read()` blocks if no data is available.
  * `write()` blocks if the buffer is full.
* Closing the unused ends is essential to avoid deadlocks.
* They are **unidirectional**; if you need two-way communication, you must create two pipes.

---



# Named Pipes (FIFOs)

A **named pipe**, also known as a **FIFO (First In, First Out)**, is a special file in the filesystem that allows processes to communicate with each other. Unlike anonymous pipes, which are limited to parent-child relationships, named pipes enable communication between **unrelated processes**.

---

## 1. How Named Pipes Work

* Created using `mkfifo()` (in code) or the `mkfifo` shell command.
* Appear in the filesystem as a special file (type `p`).
* Processes open the FIFO with `open(path, O_RDONLY)` or `open(path, O_WRONLY)`.
* Data written to the FIFO by one process can be read by another in **FIFO order**.

---

## 2. Lifecycle

1. **Creation**: A FIFO file is created in the filesystem.
2. **Open**: One process opens the FIFO for writing, another for reading.
3. **Data transfer**: Data is written into the kernel buffer, then read by the other process.
4. **Persistence**: The FIFO file remains until explicitly removed with `unlink()` or `rm`.

---

## 3. Example


### Writer

```c
#include <fcntl.h>
#include <unistd.h>
#include <string.h>

int main() {

    if (mkfifo("myfifo", 0666) == -1) { // create FIFO file node
    perror("mkfifo");
    }

    int fd = open("myfifo", O_WRONLY);
    const char *msg = "Hello via FIFO";

    write(fd, msg, strlen(msg));
    close(fd);

    if (unlink("myfifo") == -1) { //remove FIFO file node
    perror("unlink failed");
    return 1;
    }

    return 0;
}
```

### Reader

```c
#include <fcntl.h>
#include <unistd.h>
#include <stdio.h>

int main() {

    char buf[100];

    int fd = open("myfifo", O_RDONLY);
    int n = read(fd, buf, sizeof(buf)-1);
    buf[n] = '\0';

    printf("Reader got: %s\n", buf);
    close(fd);
    return 0;
}
```

---

## 4. Simple Diagram

Here’s a simplified illustration of how a named pipe connects two independent processes:

```
   [ Writer Process ]                [ Reader Process ]
          |                                  ^
          | write("msg")                     |
          v                                  |
     +---------------------------------------------+
     |              Named Pipe (FIFO)              |
     |   (kernel buffer, linked to a file path)    |
     +---------------------------------------------+
          ^                                  |
          | open("myfifo", O_WRONLY)         | open("myfifo", O_RDONLY)
          |                                  |
```

* The FIFO exists as a **special file** in the filesystem (`myfifo`).
* One process writes to it, another reads from it.
* Multiple writers/readers are allowed, but data always flows **first-in, first-out**.

---


# The Producer-Consumer Model

The diagram illustrates a **named pipe (FIFO) in a producer-consumer scenario**:

```
    +----------------+        write()       +----------------+
    | Producer (P1)  | ------------------> |  FIFO Buffer   |
    |  (Writer)      |                     |  (kernel space)|
    +----------------+                     +----------------+
                                               ^
                                               |
                                               | read()
                                               |
    +----------------+                         |
    | Consumer (P2)  | <----------------------+
    |  (Reader)      |
    +----------------+
```

1. **Producer (P1)**

   * This is the process that generates data.
   * It opens the FIFO with `O_WRONLY` and writes messages using `write()`.
   * The messages enter the kernel’s FIFO buffer, not directly into the reader.

2. **FIFO Buffer (Kernel Space)**

   * The FIFO exists as a **special file node** in the filesystem (`mkfifo("myfifo", …)`), but the data itself is stored in the **kernel memory buffer** while being transferred.
   * It ensures **First-In, First-Out (FIFO)** order, so messages are delivered to consumers in the same order they were written.
   * The buffer allows **synchronization**: if the reader is not ready, the writer may block (unless opened with `O_NONBLOCK`).

3. **Consumer (P2)**

   * This process receives data.
   * It opens the FIFO with `O_RDONLY` and calls `read()` to retrieve messages.
   * The consumer does not need to know who the producer is; it only relies on the FIFO path.

4. **Flow of Data**

   * Messages move **from the producer → FIFO kernel buffer → consumer**.
   * Multiple producers or consumers can exist, but the kernel guarantees **message ordering per producer-consumer pair**.

---

✅ **Key Takeaways from the Diagram**

* Named pipes decouple producers and consumers, allowing them to be **completely independent processes**.
* The **filesystem node** provides a shared reference, while the **kernel buffer** handles actual data transfer.
* Processes only need to open the FIFO path; no parent-child relationship is required.



