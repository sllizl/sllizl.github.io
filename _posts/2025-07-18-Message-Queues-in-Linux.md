---
title:  Message Queues in Linux
date: 2025-07-18 20:29:23 +0900
categories: [Blog, IPC]
tags: [message queues]     # TAG names should always be lowercase
author: sllizl
comment: false
---



# Understanding Message Queues in Linux

In Linux, **message queues** are a powerful **inter-process communication (IPC)** mechanism. Unlike pipes, which transfer raw byte streams, message queues allow processes to send **discrete messages** with a **message type**, giving more flexibility and control.

Linux provides two main APIs for message queues:

1. **System V Message Queues** – older, widely supported (`msgget`, `msgsnd`, `msgrcv`, `msgctl`)
2. **POSIX Message Queues** – newer, more modern (`mq_open`, `mq_send`, `mq_receive`, `mq_close`)

In this post, we focus on **System V message queues**, which are simpler for learning basic concepts.

---

## 1. What is a Message Queue?

A message queue is essentially a **kernel-maintained list of messages**. Each message has:

* A **type** (long integer, > 0)
* A **payload** (arbitrary data, usually a struct or buffer)

Messages are stored in **FIFO order within each type**, and processes can selectively receive messages by type.

---

## 2. Key System Calls

### a) Create or get a queue: `msgget()`

```c
#include <sys/ipc.h>
#include <sys/msg.h>

int msgid = msgget(1234, IPC_CREAT | 0666);
```

* `1234` is a key that identifies the queue (can also be generated using `ftok()`).
* `IPC_CREAT` creates the queue if it does not exist.
* Permissions `0666` allow read/write access for all users.

### b) Send a message: `msgsnd()`

```c
#include <string.h>
#include <sys/msg.h>

struct msgbuf {
    long mtype;      // message type, must be > 0
    char mtext[100]; // message data
};

struct msgbuf msg;
msg.mtype = 1;
strcpy(msg.mtext, "Hello, queue!");

msgsnd(msgid, &msg, strlen(msg.mtext)+1, 0);
```

* Messages are **atomic**, so they are not broken into bytes.
* Multiple processes can send messages concurrently.

### c) Receive a message: `msgrcv()`

```c
struct msgbuf msg;
msgrcv(msgid, &msg, sizeof(msg.mtext), 1, 0); // receive type 1
printf("Received: %s\n", msg.mtext);
```

* The fourth parameter specifies **which message type** to receive:

  * `0` → receive first message in queue regardless of type
  * `>0` → receive first message matching that type
  * `<0` → receive first message with type ≤ absolute value

### d) Delete the queue: `msgctl()`

```c
msgctl(msgid, IPC_RMID, NULL);
```

* Removes the queue from the kernel.

---

## 3. Example: Parent Sends, Child Receives

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/ipc.h>
#include <sys/msg.h>
#include <unistd.h>

struct msgbuf {
    long mtype;
    char mtext[100];
};

int main() {
    int msgid = msgget(1234, IPC_CREAT | 0666);

    if (fork() == 0) {

        // Child: receive
        struct msgbuf msg;
        msgrcv(msgid, &msg, sizeof(msg.mtext), 1, 0);
        printf("Child received: %s\n", msg.mtext);
    } else {

        // Parent: send
        struct msgbuf msg;
        msg.mtype = 1;
        strcpy(msg.mtext, "Hello from parent");
        msgsnd(msgid, &msg, strlen(msg.mtext)+1, 0);
    }

    return 0;
}
```

Output:

```
Child received: Hello from parent
```

---

## 4. Advantages of Message Queues

* **Discrete messages**: easier to manage than raw byte streams.
* **Selective reading by type**: processes can filter messages.
* **Multiple senders/receivers**: kernel handles synchronization.
* **Atomic delivery**: messages are delivered in a single unit.

---

## 5. Comparison with Pipes

| Feature           | Pipe (Anonymous/FIFO)          | Message Queue                       |
| ----------------- | ------------------------------ | ----------------------------------- |
| Data type         | Stream of bytes                | Discrete messages with type         |
| Communication     | Related/unrelated processes    | Related/unrelated processes         |
| Blocking behavior | Block on read/write            | Block on send/receive               |
| Selective reading | No                             | Yes (by message type)               |
| Persistence       | Memory only (FIFO file exists) | Kernel queue persists until removed |

---

## 6. Notes and Best Practices

* **Message size limit**: Linux defines a maximum message size (check `/proc/sys/fs/mqueue/msgsize_max` for POSIX queues or `MSGMAX` for System V).
* **Non-blocking operations**: set `IPC_NOWAIT` flag in `msgsnd` or `msgrcv`.
* **Cleanup**: always remove message queues with `msgctl(IPC_RMID)` to avoid resource leaks.


