---
layout: post
title: Non-Blocking Request Handling
tags: python, web service
blogfeed: true
---


Web applications normally perform various operations behind the scenes which take some time to process such as writing to remote database, logging over network file system or sending emails. Synchronous processing of the slow operations will reduce the responsiveness of the web application and make the user experience not very pleasant. Here I compare two non-blocking approaches using `epoll` and `threading`.

{{ more }}

> Code: [surfsnippets/asynchandler][source-asynchandler]

Suppose there is some operation in our web application which slows it down. The question is how to handle this somewhat independent operation asynchronously without blocking the response. I wrote two simple scripts which demonstrate `epoll` and `threading` and compared the benchmarks for these two approaches. In our case the slow operation is just sleeping for 1 second.

{% highlight python linenos %}
import time

def make_log(recv):
    time.sleep(1)
{% endhighlight %}

## Blocking Request Handling

Naive approach to request handling is to perform the operations synchronously without noticing that some operations take much more time to complete than the others. Here is the code:

{% highlight python linenos %}
# sync.py

import os
import socket
import threading
import time

PORT        = 8080
HOST        = "127.0.0.1"
SOCK_FLAGS  = socket.AI_PASSIVE | socket.AI_ADDRCONFIG
counter     = 0     # global variable

def get_inet_socket(backlog=128):
    "Blocking socket"
    res     = socket.getaddrinfo(HOST, PORT, socket.AF_UNSPEC, socket.SOCK_STREAM, 0, SOCK_FLAGS)
    af, socktype, proto, canonname, sockaddr = res[0]
    sock    = socket.socket(af, socktype, proto)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(sockaddr)
    sock.listen(backlog)
    return sock


def make_log(recv):
    "Perform logging"
    global counter
    counter  += 1
    print "num = %s" % counter
    print recv
    time.sleep(1)


def main():
    # Create server socket
    isock   = get_inet_socket()

    while True:
        # Get data from the inet client
        conn, addr  = isock.accept()
        recv    = conn.recv(1024)

        # Blocking request handling
        make_log(recv)

        # Respond to the inet client
        conn.send("Doobie Doo")
        conn.close()

    isock.close()

if __name__ == "__main__":
    main()
{% endhighlight %}

## Non-Blocking Request Handling with Epoll

A better approach is to use `epoll` system call to process the event. The epoll mechanism is used in a popular [Tornado](http://www.tornadoweb.org/) and [Nginx](http://nginx.org/) web servers. In Tornado inet socket is registered with the epoll and list of callback handlers. When the request comes in, the epoll event is received and processed by the corresponding handler.

In my approach, I already have web server listening to this socket and some of the web servers already use the epoll mechanism. So instead of registering inet socket I use internal unix socket to talk to the epoll thread. The system looks like the following. Initially, two threads are running: main and epoll threads. The main thread listens to incoming requests and sends data to the unix socket registered with the epoll. The epoll thread (or `LoggingThread`) receives data from the unix socket and asynchronously handles the request (in our case it just takes a nap :) ). Here is the code:

{% highlight python linenos %}
# async_epoll.py

import os
import errno
import select
import socket
import functools
import threading
import time

_EPOLLIN    = 0x001
_EPOLLERR   = 0x008
_EPOLLHUP   = 0x010

PORT        = 8080
HOST        = "127.0.0.1"
TIMEOUT     = 3600
SOCK_FLAGS  = socket.AI_PASSIVE | socket.AI_ADDRCONFIG
EPOLL_FLAGS = _EPOLLIN | _EPOLLERR | _EPOLLHUP
SOCK_NAME   = "/tmp/logger.sock"
counter     = 0     # global variable

class LoggerThread(threading.Thread):

    def __init__(self):
        threading.Thread.__init__(self)


    def run(self):
        sock    = get_server_socket()
        ep      = select.epoll()
        ep.register(sock.fileno(), EPOLL_FLAGS)         # register socket
        handler = functools.partial(conn_ready, sock)   # add handler for the socket

        events      = {}
        while True:
            event_pairs = ep.poll(TIMEOUT)
            events.update(event_pairs)
            while events:
                fd, ev = events.popitem()
                try:
                    handler(fd, ev)
                except (OSError, IOError), e:
                    if e.args[0] == errno.EPIPE:
                        pass


def handle_connection(conn, address):
    "Handles connection"
    make_log(conn.recv(1024))


def conn_ready(sock, fd, ev):
    while True:
        try:
            conn, address = sock.accept()
        except socket.error, e:
            if e.args[0] not in (errno.EWOULDBLOCK, errno.EAGAIN):
                raise
            return
        conn.setblocking(0)
        handle_connection(conn, address)


# Unix socket
def get_server_socket(backlog=128):
    "Server for unix socket which listens for connections"
    sock    = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.setblocking(0)
    try:
        os.unlink(SOCK_NAME)    # Clean up socket
    except:
        pass
    sock.bind(SOCK_NAME)
    sock.listen(backlog)
    return sock


def get_client_socket():
    "Client for unix socket"
    sock    = socket.socket(socket.AF_UNIX, socket.SOCK_STREAM)
    return sock


# Inet socket
def get_inet_socket(backlog=128):
    "Blocking inet socket"
    res     = socket.getaddrinfo(HOST, PORT, socket.AF_UNSPEC, socket.SOCK_STREAM, 0, SOCK_FLAGS)
    af, socktype, proto, canonname, sockaddr = res[0]
    sock    = socket.socket(af, socktype, proto)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(sockaddr)
    sock.listen(backlog)
    return sock


def make_log(recv):
    "Perform logging"
    global counter
    counter  += 1
    print "counter = %s" % counter
    print recv
    time.sleep(1)


def main():
    # Create Logger thread
    t   = LoggerThread()
    t.setDaemon(True)
    t.start()

    # Create server socket
    isock   = get_inet_socket()

    while True:
        # Get data from the inet client
        conn, addr  = isock.accept()
        recv    = conn.recv(1024)

        # Send received data to socket
        sock    = get_client_socket()
        sock.connect(SOCK_NAME)
        sock.send(recv)
        sock.close()

        # Respond to the inet client
        conn.send("Doobie Doo")
        conn.close()

    isock.close()

    # Wait for the thread
    t.join()


if __name__ == "__main__":
    main()
{% endhighlight %}

## Non-Blocking Request Handling with Threads

Another approach for non-blocking request handling is to use threads. The idea is somewhat described in the post ["Threading in Django"](http://www.artfulcode.net/articles/threading-django/). When the request comes in, a new thread is created which handles the request. This method is used by Apache web server and requires more memory than for event driven web servers. Here is the code:

{% highlight python linenos %}
# async_thread.py

import os
import socket
import threading
import time

PORT        = 8080
HOST        = "127.0.0.1"
SOCK_FLAGS  = socket.AI_PASSIVE | socket.AI_ADDRCONFIG
counter     = 0     # global variable

class LoggerThread(threading.Thread):

    def __init__(self):
        threading.Thread.__init__(self)
        self._recv  = None


    def set_recv(self, recv):
        self._recv  = recv

    def run(self):
        make_log(self._recv)


# Inet socket
def get_inet_socket(backlog=128):
    "Blocking socket"
    res     = socket.getaddrinfo(HOST, PORT, socket.AF_UNSPEC, socket.SOCK_STREAM, 0, SOCK_FLAGS)
    af, socktype, proto, canonname, sockaddr = res[0]
    sock    = socket.socket(af, socktype, proto)
    sock.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    sock.bind(sockaddr)
    sock.listen(backlog)
    return sock


def make_log(recv):
    "Perform logging"
    global counter
    counter  += 1
    print "counter = %s" % counter
    print recv
    time.sleep(1)


def main():
    # Create server socket
    isock   = get_inet_socket()

    while True:
        # Get data from the inet client
        conn, addr  = isock.accept()
        recv    = conn.recv(1024)

        # Create Logger thread
        t   = LoggerThread()
        t.set_recv(recv)
        t.setDaemon(True)
        t.start()

        # Respond to the inet client
        conn.send("Doobie Doo")
        conn.close()

    isock.close()


if __name__ == '__main__':
    main()
{% endhighlight %}

## Benchmarks

Now we can do some benchmarks for synchronous, asynchronous with epoll and asynchronous with threads approaches. For benchmarking I used popular Apache Benchmark `ab` tool. I start one of the scripts in one terminal, like:

{% highlight bash %}
[terminal 1]$ python asynch_epoll.py
{% endhighlight %}

{% highlight bash %}
[terminal 2]$ ab -n 100 -c 10 http://localhost:8080/
{% endhighlight %}

The benchmark result for request handling with epoll will look like the following. Here we get `12345 req/sec`.

    This is ApacheBench, Version 2.3 <$Revision: 655654 $>
    Copyright 1996 Adam Twiss, Zeus Technology Ltd, http://www.zeustech.net/
    Licensed to The Apache Software Foundation, http://www.apache.org/

    Benchmarking localhost (be patient).....done


    Server Software:
    Server Hostname:        localhost
    Server Port:            8080

    Document Path:          /
    Document Length:        0 bytes

    Concurrency Level:      10
    Time taken for tests:   0.008 seconds
    Complete requests:      100
    Failed requests:        0
    Write errors:           0
    Total transferred:      1000 bytes
    HTML transferred:       0 bytes
    Requests per second:    12345.68 [#/sec] (mean)
    Time per request:       0.810 [ms] (mean)
    Time per request:       0.081 [ms] (mean, across all concurrent requests)
    Transfer rate:          120.56 [Kbytes/sec] received

    Connection Times (ms)
                  min  mean[+/-sd] median   max
    Connect:        0    0   0.1      0       1
    Processing:     0    0   0.2      0       1
    Waiting:        0    0   0.1      0       1
    Total:          0    1   0.2      1       1

    Percentage of the requests served within a certain time (ms)
      50%      1
      66%      1
      75%      1
      80%      1
      90%      1
      95%      1
      98%      1
      99%      1
     100%      1 (longest request)

Performing similar benchmarks for other scripts we can create the table:


| | No Handling | Synchronous	| Asynchronous Epoll | Asynchronous Threads
|-
| Script | `nohandling.py` | `sync.py` | `async_epoll.py` | `async_thread.py`
| Req/sec | 10000 | 1 | 12000 | 3500
{: class="table table-striped"}


Looking at the table we see that synchronous request handling gives the worst req/sec. The best performance is achieved by asynchronous request handling with epoll. Though in this method the requests are not blocked, there are a few disadvantages: a) the concurrent number of requests is limited by about 130 and b) it normally takes longer to process all the requests. The point a) can be fixed by writing request handler more carefully and reach about 1000 concurrent requests as it is implemented in Tornado. Asynchronous requests with threads gives about 4 times less responsiveness than the method with epoll the all requests are processed much faster and number of concurrent requests can be higher. Performance in asynchronous with epoll method is better than for method without request handling because the later printed the received data in the terminal.

If you need the best performance and don’t care much when the requests get handled then you better go with `Asynchronous Epoll` method. I you want a reasonable performance and do care when the requests get handled then `Asynchronous Threads` will be a better approach. In any case, blocking request handling is not a solution.

[source-asynchandler]: https://bitbucket.org/dexity/surfsnippets/src/8d3e76f12706/asynchandler