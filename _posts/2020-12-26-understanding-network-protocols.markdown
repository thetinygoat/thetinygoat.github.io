---
layout: post
title: "Understanding Network Protocols"
date: 2020-12-26 00:00:00 +0530
categories: systems networking
---

## Introduction

You might have heard the word protocol and if you are like me, chances are that you got confused about what that means. Let’s understand what it means 🥳

## What is a communication protocol?

Let’s look at what the Wikipedia definition says.

_A communication protocol is a system of rules that allow two or more entities of a communications system to transmit information via any kind of variation of a physical quantity. The protocol defines the rules, syntax, semantics, and synchronization of communication and possible error recovery methods._

Okay, so this one isn’t as bad as the usual CS jargon that is present on Wikipedia, let’s break it down.

Let’s imagine you are making a server, and you don’t know what clients are going to connect to your server, and the client is going to send some data to the server to process and the server will return the result of the computation. Now a mobile app is very different from a web app which is a lot different from an IOT device and so on, there are so many different types of clients and they are only going to increase. So what do you do as the server developer? Do you write custom logic to handle different kinds of devices? That doesn’t sound like a good use of time and resources.

Let’s think of a solution. What if there were a set of rules that all clients follow, that way the server developer would not need to write custom logic for every kind of device. The server just needs to know about the rules. The data will be sent and received in a way that is specified by the rules. This is what a protocol is, let’s read the definition again, it should make a lot more sense now, also while we're at it let’s rewrite the definition in a non-jargony way.

_A communication protocol is a system of rules that allow two or more entities of a communications system to transmit information in a standard way. The protocol defines the rules, syntax, semantics, and synchronization of communication and possible error recovery methods._

Phew, that was hard!

Now that we know what a communication protocol is, let’s look at a real-world example. I’m assuming you have done some web programming and you have some idea about HTTP.

Let’s take a look at HTTP. In my definition above I said that a protocol is a set of rules that allows clients and servers to communicate in a standard way, does HTTP follow that definition?

Yes!
HTTP has headers, body, compression and so much more. All these things are specified in the HTTP specification which every web server must implement, thus providing a standard way for clients and the server to communicate.

## TCP and UDP

TCP stands for _Transfer Control Protocol_ and UDP stands for _User Datagram Protocol_.
So what are these? TCP and UDP are low-level protocols implemented by your OS, other protocols such as HTTP, build on top of TCP, or UDP.

So what’s the difference?
I won’t go into a lot of detail but if you are interested in knowing more, [this](https://www.amazon.in/Computer-Networks-5e-5th-Tanenbaum/dp/9332518742) is a great book.

TCP is a connection-oriented protocol, which means that there is a connection between the sender and the receiver, there is an acknowledgment for every message, it has error correction and so much more that is needed for a reliable communication protocol. This is the reason why HTTP is built on top of TCP.
But all these checks and guarantees come with a latency cost, this is where UDP comes in, it doesn’t have all the checks and guarantees that TCP provides but it is very fast. This is why DNS uses UDP to serve DNS queries.
It all depends on what you are building, if you don’t care about a few packets getting lost but want lower latency, you should choose UDP. If you require your packets to be in order and want reliable communication go for TCP.

## Let's write a simple TCP protocol

We are going to write a very simple protocol that parses strings.

### Designing the protocol

Our protocol will parse strings sent over the wire. But we have a problem, how do we know when a string ends and the next string begins?
One solution is to use a delimiter to mark the end of the string, but what if that delimiter is present in the string itself?
Let's take an example
Let's say the user is sending a string `hello\nword`, and if we use `\n` as our delimiter we will get an incomplete string. So what's the solution?
How about we encode the length of the string with the string itself, that way we will know how long the string is.
Let's take an example
Let's encode our string like this `10\r\nhello\nworld\r\n` , don't worry if it looks confusing, let's break it down.

```go
10 				// length of the string
\r\n			// delimiter
hello\nworld	// our string
\r\n			// delimiter
```

The `\r\n` is called CRLF, where `\r` is the **C**arriage **R**eturn and `\n` is **L**ine **F**eed, these characters go a long way back, and discussing their history is out of the scope of this post. But these characters can be treated as the end of line characters HTTP uses CRLF to mark EOL. To remove any possibility of corrupted data we are also encoding the length of the string with itself.

### Let's get coding

The following code will be in Go if you're not familiar with the language don't worry it isn't that hard to understand. I'll also try my best to explain the code.

Okay, let's start!

```go
func main() {
	ln, err := net.Listen("tcp", ":8080")
	if err != nil {
		panic(err)
	}

	for {
		conn, err := ln.Accept()
		if err != nil {
			panic(err)
		}
		go process(conn)
	}
}
```

We are setting up a _TCP_ server on port 8080 and then in the `for` loop we are listening for connections and accepting new connections as they come.
The `go` keyword tells the Go runtime to run the `process` function in a separate goroutine. A goroutine is just a lightweight thread.

Now let's take a look at the `process` function

```go
func process(conn net.Conn) {
	r := bufio.NewReader(conn)
	for {
		// handle errors
		buf, err := r.ReadBytes('\n')
		if err == io.EOF {
			break
		}
		if err != nil {
			panic(err)
		}
		// calculate size of the message
		size := 0
		for _, c := range buf {
			if c == '\r' {
				break
			}
			size += size*10 + int(rune(c)-'0')
		}
		// read size bytes from the buffer
		data := make([]byte, size)
		buf = data
		for len(buf) > 0 {
			n, err := r.Read(buf)
			if err != nil {
				panic(err)
			}
			buf = buf[n:]
		}
		fmt.Println(string(data))
	}
	fmt.Println("closing connection")
	conn.Close()
}
```

Okay, a lot is going on here, let's break it down.
First, we are doing some basic error handling, like checking if the connection is closed, etc.
Then we are calculating the size of the string by using some basic mathematics.
Lastly, we are reading size bytes from the buffer.

## Some useful links

1. [Redis protocol specification](https://redis.io/topics/protocol)
2. I worked on a small in-memory database, the code is much smaller and easier to read, this can be a good starting point: [https://github.com/thetinygoat/parabola](https://github.com/thetinygoat/parabola)
