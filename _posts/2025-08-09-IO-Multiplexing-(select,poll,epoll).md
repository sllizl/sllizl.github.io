---
title: IO Multiplexing (select,poll,epoll)
date: 2025-08-04 23:20:33 +0900
categories: [Blog, Tutorial]
tags: [io multiplexing]     # TAG names should always be lowercase
author: sllizl
comment: false
---

# Linux – IO Multiplexing – Select vs Poll vs Epoll
One basic concept of Linux (actually Unix) is the rule that everything in Unix/Linux is a file. Each process has a table of file descriptors that point to files, sockets, devices and other operating system objects.

Typical system that works with many IO sources has an initializaion phase and then enter some kind of standby mode – wait for any client to send request and response it

# IO Multiplexing
The solution is to use a kernel mechanism for polling over a set of file descriptors. There are 3 options you can use in Linux:

- select(2)
- poll(2)
- epoll

All the above methods serve the same idea, create a set of file descriptors , tell the kernel what would you like to do with each file descriptor (read, write, ..) and use one thread to block on one function call until at least one file descriptor requested operation available.

Simple solution is to create a thread (or process) for each client , block on read until a request is sent and write a response. This is working ok with a small amount of clients but if we want to scale it to hundred of clients, creating a thread for each client is a bad idea.

---

## Select System Call
The select() system call provides a mechanism for implementing synchronous multiplexing I/O.

```c
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

A call to `select()` will block until the given file descriptors are ready to perform I/O, or until an optionally specified timeout has elapsed.

The watched file descriptors are broken into three sets:

- **int nfds**: the number of fd.
- **fd_set \*readfds**: set watched to see if data is available for reading.
- **fd_set \*writefds**: set watched to see if a write operation will complete without blocking.
- **fd_set \*exceptfds**: set watched to see if an exception has occurred, or if out-of-band data is available (these states apply only to sockets).
- **struct timeval \*timeout**: life time of select, usaually be NULL.

A given set may be NULL, in which case `select()` does not watch for that event.

On successful return, each set is modified such that it contains only the file descriptors that are ready for I/O of the type delineated by that set.

Example:

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <wait.h>
#include <signal.h>
#include <errno.h>
#include <sys/select.h>
#include <sys/time.h>
#include <unistd.h>

#define MAXBUF 256

void child_process(void)
{
  sleep(2);
  char msg[MAXBUF];
  struct sockaddr_in addr = {0};
  int n, sockfd,num=1;

  srandom(getpid());

  # Create socket and connect to server
  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = inet_addr("127.0.0.1");

  connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));

  printf("child {%d} connected \n", getpid());
  while(1){
        int sl = (random() % 10 ) +  1;
        num++;
     	sleep(sl);
  	sprintf (msg, "Test message %d from client %d", num, getpid());
  	n = write(sockfd, msg, strlen(msg));	/* Send message */
  }

}

int main()
{
  char buffer[MAXBUF];
  int fds[5];

  struct sockaddr_in addr;
  struct sockaddr_in client;
  int addrlen, n,i,max=0;;
  int sockfd, commfd;

  fd_set rset;

  for(i=0;i<5;i++)
  {
  	if(fork() == 0)
  	{
  		child_process();
  		exit(0);
  	}
  }

  sockfd = socket(AF_INET, SOCK_STREAM, 0);
  memset(&addr, 0, sizeof (addr));
  addr.sin_family = AF_INET;
  addr.sin_port = htons(2000);
  addr.sin_addr.s_addr = INADDR_ANY;

  bind(sockfd,(struct sockaddr*)&addr ,sizeof(addr));
  listen (sockfd, 5); 

  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    fds[i] = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    if(fds[i] > max)
    	max = fds[i];
  }
  
  while(1){
	FD_ZERO(&rset);
  	for (i = 0; i< 5; i++ ) {
  		FD_SET(fds[i],&rset);
  	}

   	puts("round again");
	select(max+1, &rset, NULL, NULL, NULL);

	for(i=0;i<5;i++) {
		if (FD_ISSET(fds[i], &rset)){
			memset(buffer,0,MAXBUF);
			read(fds[i], buffer, MAXBUF);
			puts(buffer);
		}
	}	
  }
  return 0;
}
```

We start with creating 5 child processes , each process connect to the server and send messages to the server. The server process uses accept(2) to create a different file descriptor for each client. The first parameter in select(2) should be the highest-numbered file descriptor in any of the three sets, plus 1 so we check the max fd num.

The main infinite loop creates a set of all file descriptors, call select and on return check which file descriptor is ready for read. For simplicity I didn’t add error checking.

On return , select changes the set to contains only the file descriptors that is ready so we need to build the set again every iteration.

The reason we need to tell select what is the highest-numbered file descriptor is the inner implementation of fd_set. Each fd is declared by a bit so fd_set is an array of 32 integers (32 *32bit = 1024 bits). The function check any bit to see if its set until it reach the maximum. This means that if we have 5 file descriptors but the highest number is 900 , the function will check any bit from 0 to 900 to find the file descriptors to watch. There is a posix alternative to select – pselect which add a signal mask while waiting (see man page).

### Select – summary:

1. We need to build each set before each call
2. The function check any bit up to the higher number – O(n)
3. We need to iterate over the file descriptors to check if it exists on the set returned from select
4. The main advantage of select is the fact that it is very portable – every unix like OS has it

---

## Poll System call
Unlike `select()`, with its inefficient three bitmask-based sets of file descriptors, `poll()` employs a single array of nfds pollfd structures. the prototype is simpler:

```c
int poll (struct pollfd *fds, unsigned int nfds, int timeout);
```

The structure pollfd has a different fields for the events and the returning events so we don’t need to build it each time:

```c
struct pollfd {
      int fd;
      short events; 
      short revents;
};
```

For each file descriptor build an object of type pollfd and fill the required events. after poll returns check the revents field.

To change the above example to use poll:

```c
  for (i=0;i<5;i++) 
  {
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    pollfds[i].fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    pollfds[i].events = POLLIN;
  }
  sleep(1);
  while(1){
  	puts("round again");
	poll(pollfds, 5, 50000);

	for(i=0;i<5;i++) {
		if (pollfds[i].revents & POLLIN){
			pollfds[i].revents = 0;
			memset(buffer,0,MAXBUF);
			read(pollfds[i].fd, buffer, MAXBUF);
			puts(buffer);
		}
	}
  }
```

Like we did with select , we need to check each pollfd object to see if its file descriptor is ready but we don’t need to build the set each iteration.

### Poll vs Select
- `poll()` does not require that the user calculate the value of the highest- numbered file descriptor +1.
- `poll()` is more efficient for large-valued file descriptors. Imagine watching a single file descriptor with the value 900 via `select()`—the kernel would have to check each bit of each passed-in set, up to the 900th bit.
- `select()`’s file descriptor sets are statically sized.
- With `select()`, the file descriptor sets are reconstructed on return, so each subsequent call must reinitialize them. The `poll()` system call separates the input (events field) from the output (revents field), allowing the array to be reused without change.
- The timeout parameter to `select()` is undefined on return. Portable code needs to reinitialize it. - This is not an issue with `pselect()`.
- `select()` is more portable, as some Unix systems do not support `poll()`.

---

## Epoll System Calls
While working with select and poll we manage everything on user space and we send the sets on each call to wait. To add another socket we need to add it to the set and call select/poll again.

`epoll()` system calls help us to create and manage the context in the kernel. We divide the task to 3 steps:

- create a context in the kernel using epoll_create
- add and remove file descriptors to/from the context using epoll_ctl
- wait for events in the context using epoll_wait

Lets change the above example to use epoll:

```c
 struct epoll_event events[5];
  int epfd = epoll_create(10);
  ...
  ...
  for (i=0;i<5;i++) 
  {
    static struct epoll_event ev;
    memset(&client, 0, sizeof (client));
    addrlen = sizeof(client);
    ev.data.fd = accept(sockfd,(struct sockaddr*)&client, &addrlen);
    ev.events = EPOLLIN;
    epoll_ctl(epfd, EPOLL_CTL_ADD, ev.data.fd, &ev); 
  }
  
  while(1){
  	puts("round again");
  	nfds = epoll_wait(epfd, events, 5, 10000);
	
	for(i=0;i<nfds;i++) {
			memset(buffer,0,MAXBUF);
			read(events[i].data.fd, buffer, MAXBUF);
			puts(buffer);
	}
  }
```

We first create a context (the parameter is ignored but has to be positive). When a client connect we create an epoll_event object and add it to the context and on the infinite loop we are only waiting on the context.

### Epoll vs Select/Poll
- We can add and remove file descriptor while waiting
- `epoll_wait()` returns only the objects with ready file descriptors
- epoll has better performance – O(1) instead of O(n)
- epoll can behave as level triggered or edge triggered (see man page)
- epoll is Linux specific so non portable
