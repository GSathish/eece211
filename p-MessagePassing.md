---
layout: page
title: Message Passing
---

## Goals of concurrent program design

Now is a good time to step back and look at what we're doing with regards to concurrent programming.

Recall that our primary goals are to create software that is **safe from bugs**, **easy to understand**, and **ready for change**.

Building concurrent software is clearly a challenge for all three of these goals.
We can break the issues into two general classes.
When we ask whether a concurrent program is *safe from bugs*, we care about two properties:

+ **Safety.**
  Does the concurrent program satisfy its invariants and its specifications?
  Races in accessing mutable data threaten safety.
  Safety asks the question: can you prove that **some bad thing never happens**?

+ **Liveness.**
  Does the program keep running and eventually do what you want, or does it get stuck somewhere waiting forever for events that will never happen?
  Can you prove that **some good thing eventually happens**?

Deadlocks threaten liveness.
Liveness may also require *fairness*, which means that concurrent modules are given processing capacity to make progress on their computations.
Fairness is mostly a matter for the operating system's thread scheduler, but you can influence it (for good or for ill) by setting thread priorities.

## Message passing with threads

We can discuss, separately, *message passing* between processes: [clients and servers communicating over network sockets](../q-NetworkProgramming)

We can also use message passing between threads within the same process, and this design is often preferable to a shared memory design with locks.

Use a synchronized queue for message passing between threads.
The queue serves the same function as the buffered network communication channel in client/server message passing.
Java provides the [`BlockingQueue`](java:java/util/concurrent/BlockingQueue) interface for queues with blocking operations:

In an ordinary [`Queue`](java:java/util/Queue):

+ **`add(e)`** adds element `e` to the end of the queue.
+ **`remove()`** removes and returns the element at the head of the queue, or throws an exception if the queue is empty.

A synchronized queue, according to the Java API documentation:

> additionally supports operations that wait for the queue to become non-empty when retrieving an element, and wait for space to become available in the queue when storing an element.

+ **`put(e)`** *blocks* until it can add element `e` to the end of the queue (if the queue does not have a size bound, `put` will not block).
+ **`take()`** *blocks* until it can remove and return the element at the head of the queue, waiting until the queue is non-empty.

![Producer-consumer with message passing]({{ site.url }}/public/images/producer-consumer.png)

Analogous to the client/server pattern for message passing over a network is the **producer-consumer design pattern** for message passing between threads.
Producer threads and consumer threads share a synchronized queue.
Producers put data or requests onto the queue, and consumers remove and process them.
One or more producers and one or more consumers might all be adding and removing items from the same queue.
This queue must be safe for concurrency.

Java provides two implementations of `BlockingQueue`:

+ [`ArrayBlockingQueue`](java:java/util/concurrent/ArrayBlockingQueue) is a fixed-size queue that uses an array representation.
  `put`ting a new item on the queue will block if the queue is full.
+ [`LinkedBlockingQueue`](java:java/util/concurrent/LinkedBlockingQueue) is a growable queue using a linked-list representation.
  If no maximum capacity is specified, the queue will never fill up, so `put` will never block.

Unlike the streams of bytes sent and received by sockets, these synchronized queues (like normal collections classes in Java) can hold objects of an arbitrary type.
Instead of designing a wire protocol, we must choose or design a type for messages in the queue.
And just as we did with operations on a threadsafe ADT or messages in a wire protocol, we must design our messages here to prevent race conditions and enable clients to perform the atomic operations they need.

### Bank account example

![Message passing model for bank accounts]({{ site.url }}/public/images/concurrency/message-passing-bank-account.jpg)

Our first example of message passing was the [bank account example](../m-Concurrency/).

Each cash machine and each account is its own module, and modules interact by sending messages to one another.
Incoming messages arrive on a queue.

We designed messages for `get-balance` and `withdraw`, and said that each cash machine checks the account balance before withdrawing to prevent overdrafts:

```
get-balance
if balance >= 1 then withdraw 1
```

But it is still possible to interleave messages from two cash machines so they are both fooled into thinking they can safely withdraw the last dollar from an account with only $1 in it.

We need to choose a better atomic operation: `withdraw-if-sufficient-funds` would be a better operation than just `withdraw`.

## Implementing message passing with queues

You can see all the code for this example on GitHub: [**Fibonacci Server**](https://github.com/EECE-210/example16).
All the relevant parts are excerpted below.

Here's a message passing module for finding Fibonacci numbers:

```java
/**
 * Fibonacci numbers.
 */
class FibonacciFinder {

	private final BlockingQueue<Integer> in;
	private final BlockingQueue<FibonacciResult> out;

	// Rep invariant: in, out != null

	/**
	 * Make a FibonacciFinder that will listen for requests and generate replies.
	 * 
	 * @param requests
	 *            queue to receive requests from
	 * @param replies
	 *            queue to send replies to
	 */
	public FibonacciFinder(BlockingQueue<Integer> requests,
			BlockingQueue<FibonacciResult> replies) {
		this.in = requests;
		this.out = replies;
	}

	/**
	 * Compute the n^th Fibonacci number
	 * 
	 * @param n
	 *            indicates the Fibonacci number to compute. Requires n > 0.
	 * @return the n^th Fibonacci number
	 */
	public BigInteger fibonacci(int n) {
		if (n <= 0)
			return BigInteger.valueOf(0);
		if (n == 1)
			return BigInteger.valueOf(1);

		// if n >= 2, do the following

		BigInteger a = BigInteger.valueOf(0);
		BigInteger b = BigInteger.valueOf(1);
		BigInteger c;

		while (n > 1) {
			c = b;
			b = a.add(b);
			a = c;
		}
		return b;
	}

	/**
	 * Start handling squaring requests.
	 */
	public void start() {
		new Thread(new Runnable() {
			public void run() {
				while (true) {
					// TODO: we may want a way to stop the thread
					try {
						// block until a request arrives
						int x = in.take();
						// compute the answer and send it back
						BigInteger y = fibonacci(x);
						out.put(new FibonacciResult(x, y));
					} catch (InterruptedException ie) {
						ie.printStackTrace();
					}
				}
			}
		}).start();
	}
}

/**
 * A squaring result message.
 */
class FibonacciResult {
	private final int input;
	private final BigInteger output;

	/**
	 * Make a new result message.
	 * 
	 * @param input
*            input number
	 * @param output
	 *            Fibonacci(input)
	 */
	public FibonacciResult(int input, BigInteger output) {
		this.input = input;
		this.output = output;
	}

	// TODO: we will want more observers, but for now...

	@Override
	public String toString() {
		return "fibonacci(" + input + ") = " + output;
	}
}
```

Incoming messages to the `FibonacciFinder` are integers; the Fibonacci Finder knows that its job, given an integer *n*, is to find the n<sup>th</sup> Fibonacci number, so no further details are required.

Outgoing messages are instances of `FibonacciResult`:

```java
/**
 * A Fibonacci result message.
 */
class FibonacciResult {
	private final int input;
	private final BigInteger output;

	/**
	 * Make a new result message.
	 * 
	 * @param input
	 *            input number
	 * @param output
	 *            Fibonacci(input)
	 */
	public FibonacciResult(int input, BigInteger output) {
		this.input = input;
		this.output = output;
	}

	// TODO: we will want more observers, but for now...

	@Override
	public String toString() {
		return "fibonacci(" + input + ") = " + output;
	}
}
```

We would probably add additional observers to `FibonacciResult` so clients can retrieve the input number and output result.

Finally, here's a main method that uses `FibonacciFinder`:

```java
	/**
	 * Create and use a FibonacciFinder.
	 */
	public static void main(String[] args) {

		BlockingQueue<Integer> requests = new LinkedBlockingQueue<>();
		BlockingQueue<FibonacciResult> replies = new LinkedBlockingQueue<>();

		FibonacciFinder fibber = new FibonacciFinder(requests, replies);
		fibber.start();

		try {
			// make a request
			requests.put(42);

			// maybe do something concurrently...

			// read the reply
			System.out.println(replies.take());
		} catch (InterruptedException ie) {
			ie.printStackTrace();
		}
	}
```

It should not surprise us that this code has a very similar flavour to the code for implementing message passing with sockets.

### Stopping

What if we want to shut down `Fibonacci Finder` so it is no longer waiting for new inputs?
In the client/server model, if we want the client or server to stop listening for our messages, we close the socket.
And if we want the client or server to stop altogether, we can quit that process.
But here, the Fibonacci Finder is just another thread in the *same* process, and we can't "close" a queue.

One strategy is a *poison pill*: a special message on the queue that signals the consumer of that message to end its work.
To shut down the Fibonacci Finder, since its input messages are merely integers, we would have to choose a magic poison integer (everyone knows the 0th Fibonacci number is 0 so no one will need to ask for the *Fibonacci(0)*…) or use null (don't use null).
Instead, we might change the type of elements on the requests queue to an ADT:

    FibonacciRequest = IntegerRequest + StopRequest

with operations:

    input : FibonacciRequest &rarr; int
    shouldStop : FibonacciRequest &rarr; boolean

and when we want to stop the Fibonacci Finder, we enqueue a `StopRequest` where `shouldStop` returns `true`.

For example, in `FibonacciFinder.start()`:

```java
    public void run() {
        while (true) {
            try {
                // block until a request arrives
                FibonacciRequest req = in.take();
                // see if we should stop
                if (req.shouldStop()) { break; }
                // compute the answer and send it back
                int x = req.input();
                BigInteger y = fibonacci(x);
                out.put(new FibonacciResult(x, y));
            } catch (InterruptedException ie) {
                ie.printStackTrace();
            }
        }
    }
```

It is also possible to *interrupt* a thread by calling its `interrupt()` method.
If the thread is blocked waiting, the method it's blocked in will throw an `InterruptedException` (that's why we have to try-catch that exception almost any time we call a blocking method).
If the thread was not blocked, an *interrupted* flag will be set.
The thread must check for this flag to see whether it should stop working.
For example:

```java
    public void run() {
        // handle requests until we are interrupted
        while ( ! Thread.interrupted()) {
            try {
                // block until a request arrives
                int x = in.take();
                // compute the answer and send it back
                BigInteger y = fibonacci(x);
                out.put(new FibonacciResult(x, y));
            } catch (InterruptedException ie) {
                // stop
                break;
            }
        }
    }
```

## Thread safety arguments with message passing

A thread safety argument with message passing might rely on:

+ **Existing threadsafe data types** for the synchronized queue.
  This queue is definitely shared and definitely mutable, so we must ensure it is safe for concurrency.

+ **Immutability** of messages or data that might be accessible to multiple threads at the same time.

+ **Confinement** of data to individual producer/consumer threads.
  Local variables used by one producer or consumer are not visible to other threads, which only communicate with one another using messages in the queue. 

+ **Confinement** of mutable messages or data that are sent over the queue but will only be accessible to one thread at a time.
  This argument must be carefully articulated and implemented.
  But if one module drops all references to some mutable data like a hot potato as soon as it puts them onto a queue to be delivered to another thread, only one thread will have access to those data at a time, precluding concurrent access.

## Message passing deadlock

When buffers fill up, message passing systems can experience deadlock.

In a deadlock, two (or more) concurrent modules are both blocked waiting for each other to do something.
Since they're blocked, no module will be able to make anything happen, and none of them will break the deadlock.

In general, in a system of multiple concurrent modules communicating with each other, we can imagine drawing a graph in which the nodes of the graph are modules, and there's an edge from A to B if module A is blocked waiting for module B to do something.
The system is deadlocked if at some point in time, there is a *cycle* in this graph.
The simplest case is the two-node deadlock, A &rarr; B and B &rarr; A, but more complex systems can achieve (not that they'll be happy about it) larger deadlocks.

A message passing system in deadlock appears to simply hang, just the way a deadlock with locks does.

### Example: network Fibonacci server

Let's see an example of message passing deadlock.
We'll use a network version of the Fibonacci server, so you can compare their implementations.
All you need to pay attention to is the spec of the server: the messages it receives and sends.

```java
/**
 * FibonacciServer is a server that finds *Fibonacci(n)* given *n*.
 * It accepts requests of the form:
 *      Request ::= Number "\n"
 *      Number ::= [0-9]+
 * and for each request, returns a reply of the form:
 *      Reply ::= (Number | "err") "\n"
 * where a Number is the requested Fibonacci number,
 * or "err" is used to indicate a misformatted request.
 * FibonacciServer can handle only one client at a time.
 */
public class FibonacciServer {
    /** Default port number where the server listens for connections. */
    public static final int FIBONACCI_PORT = 4949;
    
    private ServerSocket serverSocket;
    // Rep invariant: serverSocket != null
    
    /** Make a FibonacciServer that listens for connections on port.
     *  @param port port number, requires 0 <= port <= 65535 */
    public FibonacciServer(int port) throws IOException {
        serverSocket = new ServerSocket(port);
    }
    
    /** Run the server, listening for connections and handling them.
     *  @throws IOException if the main server socket is broken */
    public void serve() throws IOException {
        while (true) {
            // block until a client connects
            Socket socket = serverSocket.accept();
            try {
                handle(socket);
            } catch (IOException ioe) {
                ioe.printStackTrace(); // but don't terminate serve()
            } finally {
                socket.close();
            }
        }
    }
    
    /** Handle one client connection. Returns when client disconnects.
     *  @param socket socket where client is connected
     *  @throws IOException if connection encounters an error */
    private void handle(Socket socket) throws IOException {
        System.err.println("client connected");
        
        // get the socket's input stream, and wrap converters around it
        // that convert it from a byte stream to a character stream,
        // and that buffer it so that we can read a line at a time
        BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()));
        
        // similarly, wrap character=>bytestream converter around the
        // socket output stream, and wrap a PrintWriter around that so
        // that we have more convenient ways to write Java primitive
        // types to it.
        PrintWriter out = new PrintWriter(new OutputStreamWriter(
                                  socket.getOutputStream()));

        try {
            // each request is a single line containing a number
            for (String line = in.readLine(); line != null; line = in.readLine()) {
                System.err.println("request: " + line);
                try {
                    int x = Integer.valueOf(line);
                    // compute answer and send back to client
                    int y = x * x;
                    System.err.println("reply: " + y);
                    out.print(y + "\n");
                } catch (NumberFormatException e) {
                    // complain about ill-formatted request
                    System.err.println("reply: err");
                    out.println("err");
                }
                // important! flush our buffer so the reply is sent
                out.flush();
            }
        } finally {
            out.close();
            in.close();
        }
    }
    
    /** Start a FibonacciServer running on the default port. */
    public static void main(String[] args) {
        try {
            FibonacciServer server = new FibonacciServer(FIBONACCI_PORT);
            server.serve();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

Here's a client for our number-squaring protocol; of course we could also talk to the server in a program like Telnet:

```java
/**
 * FibonacciClient is a client that sends requests to the FibonacciServer
 * and interprets its replies.
 * A new FibonacciClient is "open" until the close() method is called,
 * at which point it is "closed" and may not be used further.
 */
public class FibonacciClient {
    private Socket socket;
    private BufferedReader in;
    private PrintWriter out;
    // Rep invariant: socket, in, out != null
    
    /** Make a FibonacciClient and connect it to a server running on
     *  hostname at the specified port.
     *  @throws IOException if can't connect */
    public FibonacciClient(String hostname, int port) throws IOException {
        socket = new Socket(hostname, port);
        in = new BufferedReader(new InputStreamReader(
                     socket.getInputStream()));
        out = new PrintWriter(new OutputStreamWriter(
                      socket.getOutputStream()));
    }
    
    /** Send a request to the server. Requires this is "open".
     *  @param x number to find *Fibonacci(x)*
     *  @throws IOException if network or server failure */
    public void sendRequest(int x) throws IOException {
        out.print(x + "\n");
        out.flush(); // important! make sure x actually gets sent
    }
    
    /** Get a reply from the next request that was submitted.
     *  Requires this is "open". 
     *  @return the requested Fibonacci number
     *  @throws IOException if network or server failure */
    public int getReply() throws IOException {
        String reply = in.readLine();
        if (reply == null) {
            throw new IOException("connection terminated unexpectedly");
        }
        
        try {
            return Integer.valueOf(reply);
        } catch (NumberFormatException nfe) {
            throw new IOException("misformatted reply: " + reply);
        }
    }

    /** Closes the client's connection to the server.
     *  This client is now "closed". Requires this is "open".
     *  @throws IOException if close fails */
    public void close() throws IOException {
        in.close();
        out.close();
        socket.close();
    }
}
```

Finally, here's the broken part: a program that uses `FibonacciClient` to talk to the `FibonacciServer`:

```java
    private static final int N = 100;
    
    /**  Use a FibonacciServer to find the first *N* Fibonacci numbers. */
    public static void main(String[] args) throws IOException {
        FibonacciClient client = new FibonacciClient("localhost",
                                               FibonacciServer.FIBONACCI_PORT);
        // send the requests to find *Fibonacci(1)…Fibonacci(N)*
        for (int x = 1; x <= N; ++x) {
            client.sendRequest(x);
            System.out.println(“Fibonacci(“ + x + ”)= ?");
        }
        // collect the replies
        for (int x = 1; x <= N; ++x) {
            int y = client.getReply();
            System.out.println(“Fibonacci(“x + “) = " + y);
        }
        client.close();
    }
```

Can you smell danger in the air?

As `N` grows larger and larger, our client is making many requests to the `FibonacciServer` *without reading any replies*.
Eventually the client's receive buffer will fill up with unread replies.
Then the server's sending buffer will fill up, since data can't be sent successfully.
Finally, one of its calls to `out.print` will block, waiting for more buffer space --- and we have our deadly embrace.
The server is waiting for the client to read some replies, but the client is waiting for the server to accept its requests.
Deadlock.

### Final suggestions for preventing deadlock

One solution to deadlock is to design the system so that there is no possibility of a cycle --- so that if A is waiting for B, it cannot be that B was already (or will start) waiting for A.

Another approach to deadlock is *timeouts*.
If a module has been blocked for too long (maybe 100 milliseconds? or 10 seconds? how to decide?), then you stop blocking and throw an exception.
Now the problem becomes: what do you do when that exception is thrown?

