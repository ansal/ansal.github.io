---
layout: post
title:  "Writing a simple web server in Python"
date:   2019-04-07 15:16:00
---

And learning about Python's Sockets and HTTP too!

Last year I read a couple of networking text books - [Computer networking by
UCLouvain](http://cnp3book.info.ucl.ac.be/) and [Computer networking by
Tanenbaum](https://www.goodreads.com/book/show/166190.Computer_Networks). Since
then, I was looking for some network projects to build in order to put my
theoretical knowledge into action. So, I decided to build a simple web server
in Python.

The idea was that I will build a web server that can serve files over HTTP. So
it should do what `python -m SimpleHTTPServer` does when ran in a command line.

[https://github.com/ansal/http-server](https://github.com/ansal/http-server)
is my humble attempt to build a web Server using raw sockets offered by Python.
And the following sections describe how I built it. The source code also
contains a lot of comments that explain how everything works.

## Getting started with Socket programming in Python
Like I have said above, I started from scratch and built a TCP server using Sockets first. Though the above mentioned text books contain some examples for using sockets, I still had to use another excellent guide for understanding how low level socket works - [Beej's Guide to Network Programming](https://www.beej.us/guide/bgnet/).

Working with sockets is easy in Python. Most of the stuff you need to know are
described in this [HOWTO](https://docs.python.org/3/howto/sockets.html). 

I have also written a simple TCP client that sends text data to a server to
understand how sockets work on the client side as well. Here is the TCP client
source -
[https://github.com/ansal/http-server/blob/master/examples/client.py](https://github.com/ansal/http-server/blob/master/examples/client.py)

## Building the TCP Server
On the server side, first I built a TCP server that binds to an IP address and port and listens for client connections. Here is the source for the TCP server -
[https://github.com/ansal/http-server/blob/master/httpserver/tcpserver.py](https://github.com/ansal/http-server/blob/master/httpserver/tcpserver.py).
As you can see, I binded to the IP address and port set by the user and started
listening for connections inside an infinite loop. Inside the infinite loop, I
used `select.select` to fetch those socket streams that had data to read/write.
If you don't know about `select`, it basically gives access to your operating
system's system call under the same name `select`. It lets you monitor file
descriptors, waiting until they are are ready for some input/output operation.
You can read more about select in the docs here -
[https://docs.python.org/3/library/select.html](https://docs.python.org/3/library/select.html).

## Using the TCP Server to build the HTTP Server
[HTTP](https://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) is just text
followed by the server and client(usually browsers) to specify and get the
resources you want. I sub-classed the `TCPServer` and wrote methods for parsing
requests, looking up the file requested by the client etc. However this became
really complicated with all the operations needed for my server - serving files
from a folder.

Taking hints from the Python standard library
[http](https://github.com/python/cpython/blob/master/Lib/http/server.py)
itself, I have divided the web server into two parts. First one is the
`HTTPServer` that sub classes `TCPServer` and listen for requests. It is also
responsible for parsing the incoming request and finding the HTTP method and
path. It also handles HTTP errors like `METHOD_NOT_ALLOWED`, `NOT_FOUND` etc.
You can view the source for HTTPServer here -
[https://github.com/ansal/http-server/blob/master/httpserver/httpserver.py](https://github.com/ansal/http-server/blob/master/httpserver/httpserver.py)

Second, I created a `FileSystemHandler` that take cares of things like reading
files, handling `HEAD` and `GET` methods, sending headers, data etc. The
`HTTPServer` creates a new `FileSystemHandler` for each request and gets data
for the resource requested by the client. Source for `FileSystemHandler` is here
[https://github.com/ansal/http-server/blob/master/httpserver/handler.py](https://github.com/ansal/http-server/blob/master/httpserver/handler.py)

Putting all these things together, I was able to setup a simple web server that
serves files from a directory. I was able to view html, css, js, images etc in
my browser from my server!

I am thinking of continue writing programs that implement low level network
protocols as it is much more fun and informative than just reading books. A
simple Python bit torrent client on top of UDP is something that I want to work
on. Hopefully I can do something like that soon.
