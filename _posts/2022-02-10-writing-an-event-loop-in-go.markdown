---
layout: post
title: "Writing an Event Loop in Go"
date: 2022-02-10 00:00:00 +0530
categories: go systems
---

Event loops have existed for a long time, from [Nginx](https://www.nginx.com/) to [Node.js](https://nodejs.org/en/) to [Redis](https://redis.io/) event loops are present everywhere and for good reason, event loops provide an efficient way to implement high-performance systems.

In this post, we will be implementing a simple single-threaded TCP echo server that uses an event loop to handle multiple connections.

I am assuming you are familiar with what an event loop is(you should be if you're reading this post :P) and have some basic understanding of network programming.

# Async I/O

The gist of async I/O is that it does I/O in a non-blocking way, the server won’t block while waiting for data on a socket. The kernel provides us with a bunch of primitives to achieve async I/O.

## `kqueue`

`kqueue` is an event notification API provided by the kernel and is supported in macOS and other BSDs. Linux and windows have similar APIs namely `epoll` and `IOCPs` respectively, they have different API than that of `kqueue` but achieve a similar result.

Before `kqueue` and `epoll` we had `poll` and `select` but they are not as efficient as the new APIs.

## The `kqueue` API

We have 2 syscalls to interact with the kqueue API.

1. `kqueue()` creates a new queue and returns a file descriptor.
2. `kevent()` has two uses, it can be used to register new events and to poll for new events.

An event has the following structure:

```c
struct kevent {
    uintptr_t   ident;		/* identifier for this event */
    short     filter;		/* filter for event */
    u_short   flags; 		/* action flags for kqueue */
    u_int     fflags; 		/* filter flag value */
    int64_t   data; 		/*   filter data value */
    void      *udata;		/* opaque user data identifier */
    uint64_t  ext[4];		/* extensions */
};
```

We use the above structure to create a new event and then register that event using the `kevent()` syscall.

Let’s go over different fields of the event structure:

1. `ident` - Holds the file descriptor which we want to monitor.
2. `filter` - Used to describe the event we want to listen to, eg:- read, write, etc.
3. `flags` - Used to describe what we want to do with the event, eg:- add to queue or remove from the queue, etc.
4. `fflags` - Filter specific flags.
5. `udata` - Arbitrary used defined data.
6. `ext` - Extended data passed to and from the kernel.

If you want detailed documentation on kqueue, the [official man page](https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2)
is a great resource.

# Writing the Event Loop

**Note**: Because I am using a mac, I will be using kqueue to implement the event loop and as a result, the code will not work on Linux or Windows systems.

Full code can be found [here](https://github.com/thetinygoat/kqueue-event-loop).

```go
type Socket struct {
    fd int
}

type Server struct {
    socket *Socket
}
```

We have a `Socket` type that holds the file descriptor to the socket and we have a `Server` type that holds the socket on which the server will listen for new connections.

```go
func NewServer(host string, port int) (*Server, error) {
    fd, err := syscall.Socket(syscall.AF_INET, syscall.SOCK_STREAM, syscall.IPPROTO_TCP)
    if err != nil {
        return nil, err
    }

    socket := &Socket{fd: fd}
    addr := syscall.SockaddrInet4{
        Port: port,
    }
    copy(addr.Addr[:], net.ParseIP(host))
    syscall.Bind(socket.fd, &addr)

    err = syscall.Listen(socket.fd, syscall.SOMAXCONN)

    if err != nil {
        return nil, err
    }

    return &Server{socket: socket}, nil
}
```

Here we are doing a few things, first, we are creating a new socket, `syscall.AF_INET` means that we are using ipv4, `syscall.SOCK_STREAM` means that we want a connection-oriented socket, `syscall.IPROTO_TCP` marks that the socket will be a TCP socket.

`syscall.Socket` returns a file descriptor to the new socket, we then bind the socket to the host and port passed by the client.

`syscall.Listen` might look like we are listening on the newly created socket but instead it marks the socket as a passive socket, meaning that this socket will be used for accepting new connections.

We also have a few helper methods for the `Server` and `Socket` types which are pretty self-explanatory.

```go
// Close closes the server socket
func (s *Server) Close() error {
    return syscall.Close(s.socket.fd)
}

func (s *Socket) Fd() int {
    return s.fd
}
```

## The `EventLoop` Type

```go
type EventLoop struct {
    kqueueFd int
    sockFd   int
}
```

The `EventLoop` type holds the file descriptors for the kqueue and the server socket.

```go
func NewEventLoop(sockFd int) (*EventLoop, error) {
    kqueueFd, err := syscall.Kqueue()
    if err != nil {
        return nil, err
    }

    loop := &EventLoop{kqueueFd: kqueueFd, sockFd: sockFd}

    socketEvent := syscall.Kevent_t{
        Ident:  uint64(loop.sockFd),
        Filter: syscall.EVFILT_READ,
        Flags:  syscall.EV_ADD | syscall.EV_ENABLE,
    }

    r, err := syscall.Kevent(loop.kqueueFd, []syscall.Kevent_t{socketEvent}, nil, nil)

    if err != nil {
        return nil, err
    }

    if r == -1 {
        return nil, errors.New("failed to register socket with kqueue")
    }

    return loop, nil
}
```

There are a lot of things happening here, let’s go step by step.

1. First we are creating a new kqueue using `syscall.Kqueue` which will return the file descriptor this new kqueue.
2. Then we are instantiating a new kqueue event using the `syscall.Kevent_t` struct.
3. Then we are registering the above event with the kqueue using `syscall.Kevent`.

The Important thing to note is that `syscall.Kevent` is used to both register events and poll for events as we will see in a moment.

`syscall.Kevent` has the following signature:

```go
syscall.Kevent(kqueueFileDescriptor, listOfEventsToRegister, listOfReadyEvents, timeout)
```

```go
func (e *EventLoop) Start() {
    for {
        events := make([]syscall.Kevent_t, 1)
        numEvents, err := syscall.Kevent(e.kqueueFd, nil, events, nil)

        if err != nil {
            continue
        }

        for i := 0; i < numEvents; i++ {
            event := events[i]
            eventFd := int(event.Ident)

            if event.Flags&syscall.EV_EOF != 0 {
                syscall.Close(eventFd)
            } else if eventFd == e.sockFd {
                sockFd, _, err := syscall.Accept(eventFd)
                if err != nil {
                    continue
                }

                sockEvent := syscall.Kevent_t{
                    Ident:  uint64(sockFd),
                    Filter: syscall.EVFILT_READ,
                    Flags:  syscall.EV_ADD | syscall.EV_ENABLE,
                }

                r, err := syscall.Kevent(e.kqueueFd, []syscall.Kevent_t{sockEvent}, nil, nil)

                if err != nil || r == -1 {
                    continue
                }
            } else if event.Filter&syscall.EVFILT_READ != 0 {
                buf := make([]byte, 1024)

                n, err := syscall.Read(eventFd, buf)
                if err != nil {
                    continue
                }
                syscall.Write(eventFd, buf[:n])
            }
        }
    }
}
```

This is the main event loop, let’s go through the code step by step.

1. We create a new loop and use `syscall.Kevent` to poll for new events. The `events` slice will be populated by new events and the number of new events will be returned by `syscall.Kevent`.
2. We then loop over all the events in the `events` slice and process them one by one.
3. If the event is for a file descriptor that has been closed by ANDing `event.Flags` and `syscall.EV_EOF` and we close the file descriptor.
4. If the `eventFd` is equal to the `sockFd` this means that there is an event on our server socket and since we marked our server socket as passive using `syscall.Listen` this means that there is a new connection. We then accept the new connection using `syscall.Accept` and register this file descriptor with the kqueue.
5. Lastly if there is a file descriptor that is ready to be read we read the data into a buffer and write it back, hence creating an echo server.

![ezgif.com-gif-maker.webp](https://cdn.hashnode.com/res/hashnode/image/upload/v1644478985716/lZvoYMSnF.webp)

This is a basic event loop that only listens for connections on a single socket. Production-ready event loops can listen to many types of events(file I/O, timers, etc.) but the basic concept remains the same under the hood.

# References

- [https://wiki.netbsd.org/tutorials/kqueue_tutorial/](https://wiki.netbsd.org/tutorials/kqueue_tutorial/)
- [https://dev.to/frosnerd/writing-a-simple-tcp-server-using-kqueue-cah](https://dev.to/frosnerd/writing-a-simple-tcp-server-using-kqueue-cah)
