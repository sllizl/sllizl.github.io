---
title: Mutex lock for Linux Thread Synchronization (user space)
date: 2025-08-04 23:20:33 +0900
categories: [Blog, Tutorial]
tags: [mutex (user space)]     # TAG names should always be lowercase
author: sllizl
comment: false
---

# Mutex lock for Linux Thread Synchronization (user space)

Thread synchronization ensures that multiple threads or processes can safely access shared resources without conflicts. The critical section is the part of the program where a shared resource is accessed, and only one thread should execute it at a time. Without synchronization, race conditions may occur, causing unpredictable results—for example, two threads updating the same counter simultaneously may produce incorrect values.

**Example of Synchronization Problem:**
```c
#include <pthread.h>
#include <stdio.h>

pthread_t tid[2];
int counter;

void* simple_task(void* arg) {

    counter += 1;

    printf("Thread %d started\n", counter);
    for (unsigned long i = 0; i < 0xFFFFFFFF; i++);
    printf("Thread %d finished\n", counter);

    return NULL;

}

int main() {

    for (int i = 0; i < 2; i++)
        pthread_create(&tid[i], NULL, simple_task, NULL);
    
    pthread_join(tid[0], NULL);
    pthread_join(tid[1], NULL);

    return 0;
}
```

**Problem Observed:**
Output may show repeated or missing job logs, e.g.,
```bash
Job 1 started
Job 2 started
Job 2 finished
Job 2 finished
```

This happens because the shared counter variable is accessed by both threads simultaneously. Its solution is mutex.

## Mutex
A mutex (mutual exclusion) ensures that only one thread accesses a shared resource at a time. Threads must lock the mutex before entering the critical section and unlock it after finishing.

### Working of a mutex
- Suppose one thread has locked a region of code using mutex and is executing that piece of code.
- Now if scheduler decides to do a context switch, then all the other threads which are ready to execute the same region are unblocked.
- Only one of all the threads would make it to the execution but if this thread tries to execute the same region of code that is already locked then it will again go to sleep.
- Context switch will take place again and again but no thread would be able to execute the locked region of code until the mutex lock over it is released.
- Mutex lock will only be released by the thread who locked it.
- So this ensures that once a thread has locked a piece of code then no other thread can execute the same region until it is unlocked by the thread who locked it.
- Hence, this system ensures synchronization among the threads while working on shared resources.

**Example with Mutex:**
```c
#include <pthread.h>
#include <stdio.h>

pthread_t tid[2];
int counter;
pthread_mutex_t lock;

void* simple_task(void* arg) {

    pthread_mutex_lock(&lock); # lock

    counter += 1;
    printf("Thread %d started\n", counter);
    for (unsigned long i = 0; i < 0xFFFFFFFF; i++);
    printf("Thread %d finished\n", counter);

    pthread_mutex_unlock(&lock); # unlock

    return NULL;

}

int main() {

    pthread_mutex_init(&lock, NULL); # lock init

    for (int i = 0; i < 2; i++)
        pthread_create(&tid[i], NULL, simple_task, NULL);
    
    pthread_join(tid[0], NULL);
    pthread_join(tid[1], NULL);

    pthread_mutex_destroy(&lock); # lock destroy

    return 0;

}
```
**In the above code:**
- A mutex is initialized in the beginning of the main function.
- The same mutex is locked in the ‘trythis()’ function while using the shared resource ‘counter’.
- At the end of the function ‘trythis()’ the same mutex is unlocked.
- At the end of the main function when both the threads are done, the mutex is destroyed.
**Output:**
```bash
Job 1 started
Job 1 finished
Job 2 started
Job 2 finished
```
