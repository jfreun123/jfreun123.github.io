---
layout: post
title:  "The C10k Problem (C1M?)"
# date:   TODO (set when moving to _posts; a non-date value here breaks `jekyll build --drafts`)
categories: cpp
---

*WORK IN PROGRESS*

*The following was written by me, but reviewed by Claude for grammar and general mistakes.*

The C10K problem is a classic problem of making a server that can handle 10,000 connections.  In today's world, this should not be hard.  Let's try to reproduce it locally on my machine with Docker.

// TODO:  start as notes from [this video](https://www.youtube.com/watch?v=DA6AGCadH60)

// TODO:  a section on my hardware and what the theoretical max should be

// TODO:  what would this look like with ZeroMQ.

// TODO:  intro — what the C10k problem is, why 10k was hard in 1999, and why the target here is C1M

// TODO:  goals of the [C1M repo](https://github.com/jfreun123/C1M) — rewrite in my own words:

- Build the same echo server several ways (thread per connection, `select`, `poll`, `epoll`, `io_uring`) so the models can be compared like for like
- Add a UDP echo server next to the TCP ones
- Push one machine from 10k to 100k to 1M concurrent connections, recording which limit broke at each step and what fixed it
- Write a load generator that can open and hold that many connections
- Measure every model:  memory per connection, latency, syscalls per request
- Reproduce each failure (blocked socket, full accept queue, file descriptor exhaustion) and diagnose it with standard Linux tools
- By the end, answer the questions below from code and measurements I can point at

// TODO:  the questions this post should let me answer — check each one off against a section below:

- What is the modern replacement for `netstat`?
- What does it mean for a socket to be blocked, and how would you find what it is blocked on?
- TCP vs UDP, and when would you use one or the other?
- What steps are involved in creating a new connection?
- How does `epoll` work, and how would you use it to build a server?
- Design an async web server handling 100k concurrent connections.  (Follow-ups:  what kind of sockets, how `epoll` works and why it is useful, what `io_uring` does.)

// TODO:  TCP vs UDP — what each one guarantees, and when I would pick one over the other

// TODO:  creating a connection — the syscalls on each side (`socket`, `bind`, `listen`, `accept`, `connect`) and the three-way handshake underneath them.  What are the SYN queue and the accept queue, and what does the `listen` backlog control?

// TODO:  the baseline — blocking sockets, one thread per connection.  Where does it fall over, and why?

// TODO:  blocked sockets — what does it mean for `recv`, `send`, `accept`, and `connect` to block?  Reproduce each one, then find what it is blocked on with `ss`, `lsof`, `strace`, and `/proc/<pid>`

// TODO:  what does one connection actually cost?  (file descriptor, kernel socket buffers, thread stack)

// TODO:  I/O multiplexing — non-blocking sockets, `select` and `poll`, and why they stop scaling

// TODO:  epoll — how does it work, and why is it useful?  Level-triggered vs edge-triggered, and what edge-triggered forces me to do on every read

// TODO:  building the server on epoll — the event loop step by step:  accepting, reading into per-connection buffers, replies that cannot be sent in full, disconnects and errors

// TODO:  scaling across cores — one event loop per core, and how connections get spread between them (`SO_REUSEPORT`)

// TODO:  the limits on the way to 1M — file descriptor limits, ephemeral ports and the 4-tuple, memory per socket

// TODO:  load generation — how do you even open 1M client connections to test this?

// TODO:  io_uring — what changes when the kernel interface is completion-based instead of readiness-based?

// TODO:  measurements — connections vs memory vs latency for each server model

// TODO:  the design question, end to end — "design an async web server handling 100k concurrent connections", answered using everything above

// TODO:  what's next — C10M and kernel bypass

## Resources:

- [C1M on GitHub](https://github.com/jfreun123/C1M)
- [The C10K problem, Dan Kegel](http://www.kegel.com/c10k.html)
- [C10k problem, Wikipedia](https://en.wikipedia.org/wiki/C10k_problem)
- [Beej's Guide to Network Programming](https://beej.us/guide/bgnet/)
- [The Linux Programming Interface, Michael Kerrisk](https://man7.org/tlpi/)
- [tcp(7), Linux manual page](https://man7.org/linux/man-pages/man7/tcp.7.html)
- [listen(2), Linux manual page](https://man7.org/linux/man-pages/man2/listen.2.html)
- [ss(8), Linux manual page](https://man7.org/linux/man-pages/man8/ss.8.html)
- [epoll(7), Linux manual page](https://man7.org/linux/man-pages/man7/epoll.7.html)
- [Lord of the io_uring](https://unixism.net/loti/)
- [1 million is so 2011, WhatsApp Blog](https://blog.whatsapp.com/1-million-is-so-2011)
- [The Road to 2 Million Websocket Connections in Phoenix](https://www.phoenixframework.org/blog/the-road-to-2-million-websocket-connections)
- [The C10M problem, Robert Graham](http://c10m.robertgraham.com/p/manifesto.html)
- [C10M: Defending the Internet at Scale, Robert Graham (Shmoocon 2013)](https://www.youtube.com/watch?v=73XNtI0w7jA)
