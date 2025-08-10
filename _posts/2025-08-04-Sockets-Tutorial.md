---
title: Sockets Tutorial
date: 2025-08-04 23:20:33 +0900
categories: [Blog, Tutorial]
tags: [socket]     # TAG names should always be lowercase
author: sllizl
comment: false
---

# Sockets Tutorial
This is a simple tutorial on using sockets for interprocess communication.


## The client server model
Most interprocess communication uses the client server model. These terms refer to the two processes which will be communicating with each other. One of the two processes, the client, connects to the other process, the server, typically to make a request for information. A good analogy is a person who makes a phone call to another person.
Notice that the client needs to know of the existence of and the address of the server, but the server does not need to know the address of (or even the existence of) the client prior to the connection being established.

Notice also that once a connection is established, both sides can send and receive information.

The system calls for establishing a connection are somewhat different for the client and the server, but both involve the basic construct of a socket.
A socket is one end of an interprocess communication channel. The two processes
each establish their own socket.

The steps involved in establishing a socket on the client side are as follows:

1. Create a socket with the `socket()` system call
2. Connect the socket to the address of the server using the `connect()` system call
3. Send and receive data. There are a number of ways to do this, but the simplest is to use the `read()` and `write()` system calls.
4. Close socket with `close()`.

The steps involved in establishing a socket on the server side are as follows:

1. Create a socket with the `socket()` system call
2. Bind the socket to an address using the `bind()` system call. For a server socket on the Internet, an address consists of a port number on the host machine.
3. Listen for connections with the `listen()` system call
4. Accept a connection with the `accept()` system call. This call typically blocks until a client connects with the server.
5. Send and receive data with `write()` and `read()`.
6. Close socket with `close()`.

## Socket Types
When a socket is created, the program has to specify the address domain and the socket type. Two processes can communicate with each other only if their sockets are of the same type and in the same domain.

There are two widely used address domains, the **Unix domain**, in which two processes which share a common file system communicate, and the **Internet domain**, in which two processes running on any two hosts on the Internet communicate. Each of these has its own address format.

The address of a socket in the Unix domain is a character string which is basically an entry in the file system.

The address of a socket in the Internet domain consists of the Internet address of the host machine (every computer on the Internet has a unique 32 bit address, often referred to as its IP address).
In addition, each socket needs a port number on that host.
Port numbers are 16 bit unsigned integers.
The lower numbers are reserved in Unix for standard services. For example, the port number for the FTP server is 21. It is important that standard services be at the same port on all computers so that clients will know their addresses.
However, port numbers above 2000 are generally available.

There are two widely used socket types, stream sockets, and datagram sockets. Stream sockets treat communications as a continuous stream of characters, while datagram sockets have to read entire messages at once. Each uses its own communciations protocol.

Stream sockets use TCP (Transmission Control Protocol), which is a reliable, stream oriented protocol, and datagram sockets use UDP (Unix Datagram Protocol), which is unreliable and message oriented.

**The examples in this tutorial will use sockets in the Internet domain using the TCP protocol.**



## Sample code
C code for a very simple client and server are provided for you. These communicate using stream sockets in the Internet domain. The code is described in detail below. However, before you read the descriptions and look at the code, you should compile and run the two programs to see what they do.

**server.c**
**client.c**

Download these into files called **server.c** and **client.c** and compile them separately into two executables called `server` and `client`.

They probably won't require any special compiling flags, but on some solaris systems you may need to link to the socket library by appending `-lsocket` to your compile command.

Ideally, you should run the client and the server on separate hosts on the Internet. Start the server first. Suppose the server is running on a machine called **cheerios**. When you run the server, you need to pass the port number in as an argument. You can choose any number between 2000 and 65535. If this port is already in use on that machine, the server will tell you this and exit. If this happens, just choose another port and try again. If the port is available, the server will block until it receives a connection from the client. Don't be alarmed if the server doesn't do anything;

It's not supposed to do anything until a connection is made.


Here is a typical command line:
```bash
server 51717
```
To run the client you need to pass in two arguments, the name of the host on which the server is running and the port number on which the server is listening for connections.

Here is the command line to connect to the server described above:
```bash
client cheerios 51717
```
The client will prompt you to enter a message.
If everything works correctly, the server will display your message on stdout, send an acknowledgement message to the client and terminate.
The client will print the acknowledgement message from the server and then terminate.
You can simulate this on a single machine by running the server in one window and the client in another. In this case, you can use the keyword **localhost** as the first argument to the client.

The server code uses a number of ugly programming constructs, and so we will go through it line by line.

---

## Server Code

In this server code section we will go throuh the example code `server.c` .

### 1. Create a Socket

To communicate with a client, we first need to create a socket file descriptor for data transfer. The function prototype for creating a socket is shown below:
```c
int socket(int domain, int type, int protocol);
```
- dmain (protocal family)
    - AF_INET: IPv4 Internet socket
    - AF_UNIX: Unix domain socket
    - AF_INET6: IPv6 Internet socket

- type (type of socket)
    - SOCK_STREAM：connection-oriented (TCP or loacal byte stream) 
    - SOCK_DGRAM: connectionless (UDP or local datagram)

- protocal
    - Usually 0, system automatically chooses a proper protocal.

In our case, it should be:
```c
server_fd = socket(AF_INET, SOCK_STREAM, 0);
```
This `socket()` function returns a new file descriptor, which is a non-negative integer. We will use this fd for several times below. 

The next step is to bind a address to the socket.

### `struct sockaddr`
Before we `bind()`, I want to introduce a critical data structure `struct sockaddr`. We must initialize `struct sockadr_in` before bind to a socket.
```c
struct sockaddr {
    sa_family_t sa_family;  // Protocal familly (AF_INET、AF_UNIX...)
    char        sa_data[14]; // The actual address data (unspecified content, fixed size)
};
```
As a prototype, sockaddr contains two fields. First is `sa_family` for specifing a protocal family, which is 2 bytes long. Second field `sa_data[14]` is for a specific address, which is 14 bytes long.

`struct sockaddr_in` is an address structure specifically designed for IPv4 which we will use, and clearly distinguishing between the port and the IP address. It is defined as follows:
```c
struct sockaddr_in {
    sa_family_t    sin_family;   // AF_INET
    in_port_t      sin_port;     // 16bit port number（network byte order）
    struct in_addr sin_addr;     // 32bit IPv4 address
    unsigned char  sin_zero[8];  // Padding bytes to maintain the same size as sockaddr
};
```
Initialization is as follows:
```c
#define SERVER_PORT 8080
#define SERVER_ADDR "127.0.0.1"

struct sockaddr_in server_addr, client_addr;

memset(&server_addr, 0, sizeof(server_addr));
server_addr.sin_family = AF_INET;             // Protocal familly AF_INET
server_addr.sin_port = htons(SERVER_PORT);    // Port number
server_addr.sin_addr.s_addr = inet_addr(SERVER_ADDR); // IP address
```
The second field of `sockaddr_in` is unsigned short `sin_port`, which contain the port number. However, instead of simply copying the port number to this field, it is necessary to convert this to **network byte order** using the function **htons()** which converts a port number in host byte order to a port number in network byte order.
