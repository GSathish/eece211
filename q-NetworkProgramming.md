---
layout: page
title: Sockets and Network Programming
---

### Objectives

In this reading we examine *client/server communication* over the network using the *socket* abstraction.

Network communication is inherently concurrent, so building clients and servers will require us to reason about their concurrent behavior and to implement them with thread safety.
We must also design the *wire protocol* that clients and servers use to communicate, just as we design the operations that clients of an ADT use to work with it.

Some of the operations with sockets are *blocking*: they block the progress of a thread until they can return a result.
Blocking makes writing some code easier, but it also foreshadows a new class of concurrency bugs we'll soon contend with in depth: deadlocks.

## Client/server design pattern

In this reading, we explore the **client/server design pattern** for communication with message passing.

In this pattern there are two kinds of processes: clients and servers.
A client initiates the communication by connecting to a server.
The client sends requests to the server, and the server sends replies back.
Finally, the client disconnects.
A server might handle connections from many clients concurrently, and clients might also connect to multiple servers.

Many Internet applications work this way: web browsers are clients for web servers, an email program like Outlook is a client for a mail server, etc.

On the Internet, client and server processes are often running on different machines, connected only by the network, but it doesn't have to be that way --- the server can be a process running on the same machine as the client.

## Network sockets

### IP addresses

A network interface is identified by an [IP address](http://en.wikipedia.org/wiki/IP_address).
IPv4 addresses are 32-bit numbers written in four 8-bit parts.
For example (as of this writing):

+ `137.82.130.49` is the IP address of a UBC web server.
  Every address whose [first octet is `137`](http://en.wikipedia.org/wiki/List_of_assigned_/8_IPv4_address_blocks) is on the UBC network.

+ `173.194.123.40` is the address of a Google web server.

+ `127.0.0.1` is the [loopback](http://en.wikipedia.org/wiki/Loopback) or [localhost](http://en.wikipedia.org/wiki/Localhost) address: it always refers to the local machine.
  Technically, any address whose first octet is `127` is a loopback address, but `127.0.0.1` is standard.

You can [ask Google for your current IP address](https://www.google.com/search?q=my+ip).
In general, as you carry around your laptop, every time you connect your machine to the network it will be assigned a new IP address.

### Hostnames

[Hostnames](http://en.wikipedia.org/wiki/Hostname) are names that can be translated into IP addresses.
A single hostname can map to different IP addresses at different times; and multiple hostnames can map to the same IP address.
For example:

+ `www.ubc.ca` is the name for UBC's web server.
  You can translate this name to an IP address yourself using `dig`, `host`, or `nslookup` on the command line, e.g.:

```
$ dig +short www.ubc.ca
137.82.130.49
```

+ `google.com` is exactly what you think it is.
  Try using one of the commands above to find `google.com`'s IP address.
  What do you see?

+ `localhost` is a name for `127.0.0.1`.
  When you want to talk to a server running on your own machine, talk to `localhost`.

Translation from hostnames to IP addresses is the job of the [Domain Name System (DNS)](http://en.wikipedia.org/wiki/Domain_Name_System).
It's super cool, and important, but not part of our discussion in EECE 210 (you can learn about DNS in EECE 358 or CPSC 317).

### Port numbers

A single machine might have multiple server applications that clients wish to connect to, so we need a way to direct traffic on the same network interface to different processes.

Network interfaces have multiple [ports](http://en.wikipedia.org/wiki/Port_(computer_networking)) identified by a 16-bit number from 0 (which is reserved, so we effectively start at 1) to 65535.

A server process binds to a particular port --- it is now **listening** on that port.
Clients have to know which port number the server is listening on.
There are some [well-known ports](http://en.wikipedia.org/wiki/List_of_TCP_and_UDP_port_numbers#Well-known_ports) which are reserved for system-level processes and provide standard ports for certain services.
For example:

+ Port 22 is the standard SSH port.
+ Port 25 is the standard email server port.
+ Port 80 is the standard web server port.
  When you connect to the URL `http://www.ubc.ca` in your web browser, it connects to `137.82.130.49` on port 80.

When the port is not a standard port, it is specified as part of the address.
For example, the URL `http://130.2.39.10:9000` refers to port 9000 on the machine at `130.2.39.10`.

When a client connects to a server, that outgoing connection also uses a port number on the client's network interface, usually chosen at random from the available *non*-well-known ports.

### Network sockets

A [**socket**](http://en.wikipedia.org/wiki/Network_socket) represents one end of the connection between client and server.

+ A **listening socket** is used by a server process to wait for connections from remote clients.
  
  In Java, use [`ServerSocket`](java:java/net/ServerSocket) to make a listening socket, and use its [`accept`](http://docs.oracle.com/javase/8/docs/api/java/net/ServerSocket.html#accept--) method to listen to it.

+ An **connected socket** can send and receive messages to and from the process on the other end of the connection.
  It is identified by both the local IP address and port number plus the remote address and port, which allows a server to differentiate between concurrent connections from different IPs, or from the same IP on different remote ports.
  
  In Java, clients use a [`Socket`](java:java/net/Socket) constructor to establish a socket connection to a server.
  Servers obtain an connected socket as a `Socket` object returned from `ServerSocket.accept`.

## Buffers

The data that clients and servers exchange over the network is sent in chunks.
These are rarely just byte-sized chunks, although they might be.
The sending side (the client sending a request or the server sending a response) typically writes a large chunk (maybe a whole string like "HELLO, WORLD!" or maybe 20 megabytes of video data).
The network chops that chunk up into packets, and each packet is routed separately over the network.
At the other end, the receiver reassembles the packets together into a stream of bytes.

The result is a bursty kind of data transmission --- the data may already be there when you want to read them, or you may have to wait for them to arrive and be reassembled.

When data arrive, they go into a **buffer**, an array in memory that holds the data until you read it.

Here is an interesting aside written by Gene Ziegler in 1995 (although the issue described in the first stanza is no longer relevant with the obsolescence of floppy disk drives).


```
Here's an easy game to play.
Here's an easy thing to say:

If a packet hits a pocket on a socket on a port,
And the bus is interrupted as a very last resort,
And the address of the memory makes your floppy disk abort,
Then the socket packet pocket has an error to report!

If your cursor finds a menu item followed by a dash,
And the double-clicking icon puts your window in the trash,
And your data is corrupted 'cause the index doesn't hash,
Then your situation's hopeless, and your system's gonna crash!

You can't say this?
What a shame sir!
We'll find you
Another game sir.

If the label on the cable on the table at your house,
Says the network is connected to the button on your mouse,
But your packets want to tunnel on another protocol,
That's repeatedly rejected by the printer down the hall,

And your screen is all distorted by the side effects of gauss
So your icons in the window are as wavy as a souse,
Then you may as well reboot and go out with a bang,
'Cause as sure as I'm a poet, the sucker's gonna hang!

When the copy of your floppy's getting sloppy on the disk,
And the microcode instructions cause unnecessary risc,
Then you have to flash your memory and you'll want to RAM your ROM.
Quickly turn off the computer and be sure to tell your mom!
```

## Streams

The data going into or coming out of a socket is a [**stream**](http://en.wikipedia.org/wiki/Stream_(computing)) of bytes.

In Java, [`InputStream`](java:java/io/InputStream) objects represent sources of data flowing into your program: reading from a file on disk with a [`FileInputStream`](java:java/io/FileInputStream), taking user input from [`System.in`](http://docs.oracle.com/javase/8/docs/api/java/lang/System.html#in), or input from a network socket.

[`OutputStream`](java:java/io/OutputStream) objects represent data sinks, places we can write data to: [`FileOutputStream`s](java:java/io/FileOutputStream) for saving to files, [`System.out`](http://docs.oracle.com/javase/8/docs/api/java/lang/System.html#out) is a [`PrintStream`](java/io/InputStream), or the output to a network socket.

In the Java Tutorials, read:

+ [I/O Streams](http://docs.oracle.com/javase/tutorial/essential/io/streams.html) up to and including *I/O from the Command Line* (8 pages)

With sockets, remember that the *output* of one process is the *input* of another process.  If Alice and Bob have a socket connection, Alice has an output stream that flows to Bob's input stream, and *vice versa*.

## Blocking

**Blocking** means that a thread waits (without doing further work) until an event occurs.
We can use this term to describe methods and method calls: if a method is a **blocking method**, then a call to that method can **block**, waiting until some event occurs before it returns to the caller.

Socket input/output streams exhibit blocking behavior:

+ When an incoming socket's buffer is empty, calling `read` blocks until data are available.
+ When the destination socket's buffer is full, calling `write` blocks until space is available.

Blocking is very convenient from a programmer's point of view, because the programmer can write code as if the `read` (or `write`) call will always work, no matter what the timing of data arrival.
If data (or for `write`, space) is already available in the buffer, the call might return very quickly.
But if the read or write can't succeed, the call **blocks**.
The operating system takes care of the details of delaying that thread until `read` or `write` *can* succeed.

Blocking happens throughout concurrent programming, not just in [I/O](http://en.wikipedia.org/wiki/Input/output) (communication into and out of a process, perhaps over a network, or to/from a file, or with the user on the command line or a GUI, ...).
Concurrent modules don't work in lockstep, like sequential programs do, so they typically have to wait for each other to catch up when coordinated action is required.

We'll see in the next reading that this waiting gives rise to the second major kind of bug (the first was race conditions) in concurrent programming: **deadlock**, where modules are waiting for each other to do something, so none of them can make any progress.
But that's for next time.

## Using network sockets

Make sure you've read about **streams** at the Java Tutorial link above, then read about network sockets:

In the Java Tutorials, read:

+ [All About Sockets](http://docs.oracle.com/javase/tutorial/networking/sockets/index.html) (4 pages)

This reading describes everything you need to know about creating server- and client-side sockets and writing to and reading from their I/O streams.

#### On the second page

The example uses a syntax we haven't seen: the [try-with-resources](http://docs.oracle.com/javase/tutorial/essential/exceptions/tryResourceClose.html) statement.

This statement has the form:

```java
try (
    // create new objects here that require cleanup after being used,
    // and assign them to variables
) {
    // code here runs with those variables
    // cleanup happens automatically after the code completes
} catch(...) {
    // you can include catch clauses if the code might throw exceptions
}
```

#### On the last page

Notice how both `ServerSocket.accept()` and `in.readLine()` are *blocking*.
This means that the server will need a *new thread* to handle I/O with each new client.
While the client-specific thread is working with that client (perhaps blocked in a read or a write), another thread (perhaps the main thread) is blocked waiting to `accept` a new connection.

Unfortunately, their multithreaded `Knock Knock Server` implementation creates that new thread by *subclassing `Thread`*.
That's *not* the recommended strategy.
Instead, create a new class that implements `Runnable`, or use an anonymous `Runnable` that calls a method where that client connection will be handled until it's closed.


## Wire protocols

Now that we have our client and server connected up with sockets, what do they pass back and forth over those sockets?

A **protocol** is a set of messages that can be exchanged by two communicating parties.
A **wire protocol** in particular is a set of messages represented as byte sequences, like `hello world` and `bye` (assuming we've agreed on a way to encode those characters into bytes).

Most Internet applications use simple ASCII-based wire protocols.
You can use a program called Telnet to check them out.
For example:

### HTTP

[Hypertext Transfer Protocol (HTTP)](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) is the language of the World Wide Web.
We already know that port 80 is the well-known port for speaking HTTP to web servers, so let's talk to one on the command line.

You can experiment with `telnet`: For input to the telnet connection, newlines (pressing enter) are shown with <b>&crarr;</b>:

<p>
$ <b>telnet www.ubc.ca 80</b><br />
Trying 137.82.130.49...<br />
Connected to www.ubc.ca.<br />
Escape character is '^]'.<br />
GET / HTTP/1.1&crarr;<br />
Host: www.ubc.ca&crarr;<br />
&crarr;<br />
&lt;!DOCTYPE html&gt;<br />
<i>... lots of output ...</i><br />
</p>

The `GET` command gets a web page.
The `/` is the path of the page you want on the site.
The HTTP/1.1 refers to the data transfer protocol being used.
So this command fetches the page at `http://www.ubc.ca:80/`.
Since 80 is the default port for HTTP, this is equivalent to visiting [http://www.ubc.ca/](http://www.ubc.ca/) in your web browser.
The result is HTML code that your browser renders to display the UBC homepage.

Internet protocols are defined by [RFC specifications](http://en.wikipedia.org/wiki/Request_for_Comments) (RFC stands for "request for comment", and some RFCs are eventually adopted as standards).
[RFC 1945](http://tools.ietf.org/html/rfc1945) defined HTTP version 1.0, and was superseded by HTTP 1.1 in [RFC 2616](http://tools.ietf.org/html/rfc2616).
So for many web sites, you might need to speak HTTP 1.1 if you want to talk to them.

HTTP version 1.1 requires the client to specify some extra information (called headers) with the request, and the blank line signals the end of the headers.

You will also more than likely find that telnet does not exit after making this request --- this time, the server keeps the connection open so you can make another request right away.
To quit Telnet manually, type the escape character (probably `Ctrl`-`]`) to bring up the `telnet>` prompt, and type `quit`.

### SMTP

[Simple Mail Transfer Protocol (SMTP)](http://en.wikipedia.org/wiki/Simple_Mail_Transfer_Protocol) is the protocol for sending email (different protocols are used for client programs that retrieve email from your inbox).
Because the email system was designed in a time before spam, modern email communication is fraught with traps and heuristics designed to prevent abuse.
But we can still try to speak SMTP using `telnet` but the exchange is verbose so we will skip the details here.

## Designing a wire protocol

When designing a wire protocol, apply the same rules of thumb you use for designing the operations of an abstract data type:

+ Keep the number of different messages **small**.
  It's better to have a few commands and responses that can be combined rather than many complex messages.

+ Each message should have a well-defined purpose and **coherent** behavior.

+ The set of messages must be **adequate** for clients to make the requests they need to make and for servers to deliver the results.

Just as we demand representation independence from our types, we should aim for **platform-independence** in our protocols.
HTTP can be spoken by any web server and any web browser on any operating system.
The protocol doesn't say anything about how web pages are stored on disk, how they are prepared or generated by the server, what algorithms the client will use to render them, etc.

We can also apply the three big ideas in this class:

+ **Safe from bugs**

  + The protocol should be easy for clients and servers to generate and parse.
    Simpler code for reading and writing the protocol (whether written with a parser generator like ANTLR, with regular expressions, etc.) will have fewer opportunities for bugs.

  + Consider the ways a broken or malicious client or server could stuff garbage data into the protocol to break the process on the other end.
     
     Email spam is one example: when we spoke SMTP above, the mail server asked *us* to say who was sending the email, and there's nothing in SMTP to prevent us from lying outright.
     We've had to build systems on top of SMTP to try to stop spammers who lie about `From:` addresses.
     
     Security vulnerabilities are a more serious example.
     E.g. protocols that allow a client to send requests with arbitrary amounts of data require careful handling on the server to avoid running out of buffer space, [or worse](http://en.wikipedia.org/wiki/Buffer_overflow).

+ **Easy to understand**: for example, choosing a text-based protocol means that we can debug communication errors by reading the text of the client/server exchange.
  It even allows us to speak the protocol "by hand" as we saw above.

+ **Ready for change**: for example, HTTP includes the ability to specify a version number, so clients and servers can agree with one another which version of the protocol they will use.
  If we need to make changes to the protocol in the future, older clients or servers can continue to work by announcing the version they will use.

[**Serialization**](http://en.wikipedia.org/wiki/Serialization) is the process of transforming data structures in memory into a format that can be easily stored or transted.
Rather than invent a new format for serializing your data between clients and servers, use an existing one.
For example, [JSON (JavaScript Object Notation)](http://en.wikipedia.org/wiki/JSON) is a simple, widely-used format for serializing basic values, arrays, and maps with string keys.

### Specifying a wire protocol

In order to precisely define for clients &amp; servers what messages are allowed by a protocol, use a grammar.

For example, here is a very small part of the HTTP 1.1 request grammar from [RFC 2616 section 5](http://www.w3.org/Protocols/rfc2616/rfc2616-sec5.html):

    request ::= request-line
                ((general-header | request-header | entity-header) CRLF)*
                CRLF
                message-body?
    request-line ::= method SPACE request-uri SPACE http-version CRLF
    method ::= "OPTIONS" | "GET" | "HEAD" | "POST" | ...
    ...

Using the grammar, we can see that in this example request from earlier:

    GET / HTTP/1.1
    Host: web.ubc.ca

+ `GET` is the `method`: we're asking the server to get a page for us.
+ `/` is the `request-uri`: the description of what we want to get.
+ `HTTP/1.1` is the `http-version`.
+ `Host: web.ubc.ca` is some kind of header --- we would have to examine the rules for each of the `...-header` options to discover which one.
+ And we can see why we had to end the request with a blank line: since a single `request` can have multiple headers that end in CRLF (newline), we have another CRLF at the end to finish the `request`.
+ We don't have any `message-body` --- and since the server didn't wait to see if we would send one, presumably that only applies for other kinds of requests.

The grammar is not enough: it fills a similar role to method signatures when defining an ADT.
We still need the specifications:

+ **What are the preconditions of a message?**
  For example, if a particular field in a message is a string of digits, is any number valid?
  Or must it be the ID number of a record known to the server?
  
  Under what circumstances can a message be sent?
  Are certain messages only valid when sent in a certain sequence?

+ **What are the postconditions?**
  What action will the server take based on a message?
  What server-side data will be mutated?
  What reply will the server send back to the client?

## Summary

In the *client/server design pattern*, concurrency is inevitable: multiple clients and multiple servers are connected on the network, sending and receiving messages simultaneously, and expecting timely replies.
A server that *blocks* waiting for one slow client when there are other clients waiting to connect to it or to receive replies will not make those clients happy.
At the same time, a server that performs incorrect computations or returns bogus results because of concurrent modification to shared mutable data by different clients will not make anyone happy.

All the challenges of making our multi-threaded code **safe from bugs**, **easy to understand**, and **ready for change** apply when we design network clients and servers.
These processes run concurrently with one another (if on different machines), and any server that wants to talk to multiple clients concurrently (or a client that wants to talk to multiple servers) must manage that multi-threaded communication.

