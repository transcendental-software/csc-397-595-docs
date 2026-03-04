---
title: Proxy Lab
---

## 1. Lab Assignment 5 Proxy Lab

A web proxy is a program that acts as an intermediary between a Web browser and an end server.  Instead of contacting the end server directly to get a Web page, the browser contacts the proxy, which forwards the request on to the end server. When the end server replies to the proxy, the proxy sends the reply on to the browser.  Proxies are variously used as firewalls or anonymizers, and often have a cache of their own.

In this lab, you will write a simple HTTP proxy in four stages:

- **Part 1.** Deal with GET requests: your proxy reads a GET request with headers from a client and forwards the request to the proper web server.  The proxy then transmits the answer of the server to the client.
- **Part 2.** Deal with POST requests: similarly, but now your proxy will read the `Content-Length` header of the request and read that many bytes from the client.  It will then forward the entire request to the web server and transmit the server’s answer to the client.
- **Part 3.** Deal with multiple clients in parallel: using a pre-threaded architecture, your proxy will be able to deal with a fixed number of clients in parallel.

## 2. Downloading the assignment

Start by accepting the GitHub Assignment by clicking on the invitation link from the assignment description on D2L. Then clone your repository in your home directory on the **matrix.cdm.depaul.edu** machine:

```shell
$ cd ~/
$ git clone git@github.com:transcendental-software/csc-374-lab5-USER.git
```

This will cause a number of files to be unpacked into the directory.  *You should only edit files in `src/` and this is the only folder that is submitted.* The `tests` folder contains the different tests and grading scripts for your program.  Use the `make` command to compile your code and the command `make test` to run the test driver.  This would run all the tests for all parts.  This is more useful during development than in our previous labs, but short tests are still provided: see Section Driver for more finely grained testing options.

## 3. Program behavior

When started, your proxy should listen for incoming connections on a port whose number is specified on the command line.  Throughout its life, your program can print diagnostic messages if you so wish:

```shell
$ src/proxy 3142
Proxy started on port 3142, waiting for connections.
```

Once a client connects to the proxy, your proxy should read and parse the entirety of a request from the client. It should determine whether the client has sent a valid HTTP request; if so, it can then establish its own connection to the appropriate web server and request the object the client specified.  Finally, your proxy should read the server’s response and forward it to the client.  *The proxy should never exit;* if it receives an incorrect request or if the client or web server dies, the proxy should gracefully carry on.

The port you are going to specify on the command line is arbitrary, within the *unprivileged range* 1024-65535.  It is quite convenient to run your proxy always on the same port while working on the lab, but as we all share the ports on our Linux machine, there should be some discipline about it.  To avoid running into port conflicts, use the ports given by `./ports-for-user`:

```shell
$ ./ports-for-user
Please use ports 15384 and 15385
$ src/proxy 15384
Proxy started on port 15384, waiting for connections.
```

Please don’t pick your own random port. If you do, you run the risk of interfering with another user.  Use these ports, as they are unique to you.

> **Note:** In practice, ports are separated into [three lists](https://en.wikipedia.org/wiki/Registered_port):
>
> - Ports 0-1023 are *well-known ports* (sometimes “system ports”),
> - Ports 1024-49151 are *registered ports* (sometimes “user ports”),
> - Ports 49152–65535 are *ephemeral ports*.
>
> The kernel requires admin rights to bind to ports 0-1023.  There’s a separation between ephemeral and user ports to avoid using an ephemeral port on a port that would traditionally be used for a specific service (e.g., 25565 is Minecraft’s default port number).

## 4. Part 1: GET requests

### 4.1. Client to proxy

#### 4.1.1. Request Line

When a URL such as [http://csc-sys.cdm.depaul.edu/cgi-bin/echo?X&Y](http://csc-sys.cdm.depaul.edu/cgi-bin/echo?X&Y) is entered into the address bar of a web browser that uses a proxy, the browser will send an HTTP request to the proxy that follows this syntax:

```
GET http://csc-sys.cdm.depaul.edu:80/cgi-bin/echo?X&Y HTTP/1.1
```

Your proxy should parse this request into the following fields, none of which being optional:

- Request type: `GET` (in Part 2, this can also be `POST`)
- Protocol: `http`
- Hostname: `csc-sys.cdm.depaul.edu:80` (where `80` is the connection port *and is optional;* without the port information, this defaults to port `80`)
- Path: `/cgi-bin/echo?X&Y`
- Protocol version: `HTTP/1.1` (this can also be `HTTP/1.0` or `HTTP/2.0`)

Your proxy should reject the connection if the request is not properly formatted.

**To parse the request** you are *not* allowed to use [`sscanf(3)`](https://man7.org/linux/man-pages/man3/sscanf.3.html).  You can use `for` loops, [`strchr(3)`](https://man7.org/linux/man-pages/man3/strchr.3.html), or [`strstr(3)`](https://man7.org/linux/man-pages/man3/strstr.3.html).  **Any use of `sscanf(3)`, at any point you are working on this lab, will be considered cheating.**

#### 4.1.2. Headers

This is then followed by a CRLF (the sequence of characters `\r\n`) and a sequence of headers with each line ending with CRLF.  A header is a pair `Key: Value` where `Key` is alphanumerical and can contain dashes and `Value` is any printable data, up to a CRLF (this means in particular that it won’t contain a `\0` character, i.e., the end-of-string character).  For instance, these are three headers:

```
User-Agent: ELinks/0.13.GIT (textmode; Linux 5.8.10-arch1-1 x86_64; 238x61-2)
Accept: */*
Accept-Language: en
```

> **Note:** The [HTTP/1.1 RFC](https://tools.ietf.org/html/rfc2616#page-15), that codifies the protocol, is much more specific on what can and cannot be part of a header.  For all purposes, this is pretty much equivalent to what is written here, but if you are curious, the `Key` part can be a “token” (any printable character except for separators, such as `/`), and the value have an [even more complex description](https://tools.ietf.org/html/rfc2616#page-31).

#### 4.1.3. Full request

The GET request is concluded with empty line (CRLF). The full request reads:

```
GET http://csc-sys.cdm.depaul.edu:80/cgi-bin/echo?X&Y HTTP/1.1
User-Agent: ELinks/0.13.GIT (textmode; Linux 5.8.10-arch1-1 x86_64; 238x61-2)
Accept: */*
Accept-Language: en

```

If the client’s request is not syntactically correct, the connection should be closed and the proxy should return to its listening state.

Each line of the request should be shorter than `MAXLINE` bytes, a constant defined to be `8192` in the handout.  If a line is longer than this, you should reject the request.  (This implies that you do not need to use [`malloc(3)`](https://man7.org/linux/man-pages/man3/malloc.3.html) to store any part of the query, since you can use a `char buffer[MAXLINE]`.)

### 4.2. Proxy to remote server

The proxy should forward the request it just received to the appropriate hostname and port with the following modifications:

- The path in the first line of the request *should not* have the hostname and *should* start with `/`.
- The HTTP version should be set to 1.0, that is, the first line of your request should end with `HTTP/1.0`.
- The proxy should set the header `Host` to the hostname value *if the header `Host` was not given by the client*.
- The proxy should set (possibly overwriting) the header `User-Agent` to the value of the constant `USER_AGENT` (defined in the code given).
- The proxy should set (possibly overwriting) the headers `Connection` and `Proxy-Connection` to the value `close`.

Resuming the previous example, your proxy sends the following request to the server `csc-sys.cdm.depaul.edu` (on port `80`):

```
GET /cgi-bin/echo?X&Y HTTP/1.0
Connection: close
Proxy-Connection: close
Accept-Language: en
Accept: */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3
Host: csc-sys.cdm.depaul.edu:80

```

Again, each line should end with CRLF and the request is concluded by an empty line.  The order of the headers is not important.  In case of error (if the server is unreachable or closes during the transmission), the proxy should neatly close the client and remote server connections and return to its listening state.

### 4.3. Remote server back to client

After the request is sent from the proxy to the remote server, the proxy simply reproduces the reply of the remote server to the client, until the remote server closes.  The server’s reply may be *huge*, the proxy should not save this information, but read it in chunks (say, of size `MAXLINE`, hence avoiding mallocing by using a `char buffer[MAXLINE]`) and forward these to the client. At this point, the proxy closes the client and remote server connections and returns to its listening state.

## 5. Part 2: POST requests

### 5.1. Client to proxy

POST requests are similar to GET requests but for the fact that the client provides raw data to the server after the headers and newline.  The client specifies how many bytes are in the payload using the `Content-Length` header; if that header does not exist, it is an invalid request.  Here is an example of such a connection, where the client sends a 12-byte payload:

```
POST http://csc-sys.cdm.depaul.edu/cgi-bin/echo?X&Y HTTP/1.1
User-Agent: curl/7.72.0
Accept: */*
Proxy-Connection: Keep-Alive
Content-Length: 12
Content-Type: application/octet-stream

Hello world!
```

Note that the data does *not* end with CRLF and that it can contain *any* character, including the end-of-string character `\0`.  **You *should not* use string processing functions to read or write the POST payload.**  In particular, `rio_readlineb` should not be used here (there are other `rio` functions that you can use, see the section on `rio` for more).

The payload may be huge: the proxy *should not* save it in memory but read it in chunks and send it to the remote server.

### 5.2. Proxy to remote server

As soon as the *headers* are over (after the empty line) and before reading the POST payload, the proxy should connect to the remote server as with GET.  Once the connection is established, the proxy should forward `Content-Length` bytes from the client to the server.  The proxy *should not try* to read from the server until all of the client’s payload is sent to the server.

### 5.3. Remote server back to client

This is the same as GET: The proxy simply forwards whatever it receives from the server to the client.

## 6. Part 3: Concurrency

In this part, you should modify your proxy so that it is prethreaded with 64 worker threads.  It should thus be able to manage 64 connections in parallel, while the main thread is still listening for and queueing new connections.  If you elect to have a function like `serve_client(int fd)` in previous parts, this should be a minor modification of your code.

> **Note:** To count the number of threads of your proxy, the test driver reads the file `/proc/PID/status`, where `PID` is the process ID of your proxy.  The `/proc` folder is a special folder ([actually, a filesystem](https://en.wikipedia.org/wiki/Procfs)) that allows accessing information stored by the kernel.  Each process has a folder therein:
>
> ```shell
> $ cat /proc/$$/status       # $$ is the PID of the current shell
> Name:    bash
> ...
> Threads: 1
> ...
> nonvoluntary_ctxt_switches:     1
> $ cat /proc/$$/maps         # This is the virtual memory map of $$
> 5579a4871000-5579a4979000 r-xp  /usr/bin/bash
> ...
> 5579a52c3000-5579a5407000 rw-p  [heap]
> ...
> 7fdb40754000-7fdb4090d000 r-xp /usr/lib64/libc-2.28.so
> ...
> 7ffdcfa75000-7ffdcfa96000 rw-p  [stack]
> ...
> ffffffffff600000-ffffffffff601000 r-xp [vsyscall]
> ```

## 7. Evaluation

### 7.1. Scoring

The scoring for this assignment is available from the driver:

```shell
$ tests/driver.sh -P list
* PART 1: GET -- 42 pts
    PART 1a: GET (correctness) -- 15 pts
    PART 1b: GET (checking headers are correctly forwarded). -- 5 pts
    PART 1c: GET (robustness, syntax) -- 7 pts
    PART 1d: GET (robustness, connection) -- 7 pts
    PART 1e: GET (check that the server's output is forwarded in chunks) -- 8 pts
* PART 2: POST -- 36 pts
    PART 2a: POST (correctness) -- 20 pts
    PART 2b: POST (robustness, syntax) -- 6 pts
    PART 2c: POST (robustness, connection) -- 10 pts
* PART 3: CONCURRENCY -- 22 pts
    PART 3a: CONCURRENCY (two files in parallel) -- 10 pts
    PART 3b: CONCURRENCY (many files in parallel) -- 12 pts
```

The main score sums up to 100 points.  You should not use any of the prohibited functions in order to score points.

### 7.2. Driver

The driver for this lab has the following usage:

```shell
$ tests/driver.sh -h
Usage: tests/driver.sh [-h] [-p PROXY_PORT] [-s SERVER] [-P PART]"
Runs the driver for the proxylab.

  -h             Print this.
  -p PROXY_PORT  Use the proxy at localhost:PROXY_PORT throughout.
  -P PART        Only check specific part.  PART is for instance 1a, 2, or Bd.
                 Use "tests/driver.sh -P list" to list all parts.
  -s SERVER      Use the server at SERVER instead of csc-sys.cdm.depaul.edu.
```

For testing purposes, it will be useful to only start one (sub)part of the driver on a proxy that you manually started (for instance in `gdb(1)`). This can be done with

```shell
... proxy started in another session on port 3142
$ tests/driver.sh -P 2c -p 3142
```

Running the driver without argument starts all the tests.

The driver logs the output of your proxy and shows the differences between your fetched pages and the expected output in the file `driver.log`.

## 8. Tools

### 8.1. `telnet`

`telnet` can connect to a remote server, take input from the user, and send it to the server.  It changes linefeeds to CRLF, hence it is very much adapted to HTTP queries.  In the following example, the user is typing the GET line, then hits return twice, indicating no headers.

```shell
... proxy started in another session on port 3142 (src/proxy 3142)
$ telnet localhost 3142
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
GET http://csc-sys.cdm.depaul.edu:80/home.html HTTP/1.1

HTTP/1.1 200 OK
Date: Mon, 22 Feb 2021 18:31:56 GMT
Server: Apache/2.4.37 (Oracle Linux) mod_fcgid/2.3.9
Last-Modified: Sun, 21 Feb 2021 20:04:06 GMT
ETag: "f0-5bbde3002e333"
Accept-Ranges: bytes
Content-Length: 240
Connection: close
Content-Type: text/html; charset=UTF-8

<html>
  <head>
    <title>Vous Etes Perdu ?</title>
  </head>
  <body>
    <h1>Perdu sur l'Internet ?</h1>
    <h2>Pas de panique, on va vous aider</h2>
    <strong><pre>    * &lt;----- vous &ecirc;tes ici</pre></strong>
  </body>
</html>
Connection closed by foreign host.
```

### 8.2. `curl`

You can use `curl` to generate HTTP requests; the above `telnet` query is equivalent to:

```shell
... proxy started in another session on port 3142 (src/proxy 3142)
$ curl -v --proxy http://localhost:3142 http://csc-sys.cdm.depaul.edu/home.html
*   Trying 127.0.0.1:3142...
* Connected to localhost (127.0.0.1) port 3142 (#0)
> GET http://csc-sys.cdm.depaul.edu/home.html HTTP/1.1
> Host: csc-sys.cdm.depaul.edu
> User-Agent: curl/7.74.0
> Accept: */*
> Proxy-Connection: Keep-Alive
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 22 Feb 2021 18:33:36 GMT
< Server: Apache/2.4.37 (Oracle Linux) mod_fcgid/2.3.9
< Last-Modified: Sun, 21 Feb 2021 20:04:06 GMT
< ETag: "f0-5bbde3002e333"
< Accept-Ranges: bytes
< Content-Length: 240
< Connection: close
< Content-Type: text/html; charset=UTF-8
<
<html>
  <head>
    <title>Vous Etes Perdu ?</title>
  </head>
  <body>
    <h1>Perdu sur l'Internet ?</h1>
    <h2>Pas de panique, on va vous aider</h2>
    <strong><pre>    * &lt;----- vous &ecirc;tes ici</pre></strong>
  </body>
</html>
* Closing connection 0
```

You can add additional headers using the `-H` option and some POST data using `--data-binary`.  For instance:

```shell
... proxy started in another session on port 3142 (src/proxy 3142)
$ curl -v --proxy http://localhost:3142 'http://csc-sys.cdm.depaul.edu/cgi-bin/echo?argument' \
        -H 'X-ExtraHeader: Value' --data-binary 'Raw POST data'
*   Trying 127.0.0.1:3142...
* Connected to localhost (127.0.0.1) port 3142 (#0)
> POST http://csc-sys.cdm.depaul.edu/cgi-bin/echo?argument HTTP/1.1
> Host: csc-sys.cdm.depaul.edu
> User-Agent: curl/7.74.0
> Accept: */*
> Proxy-Connection: Keep-Alive
> X-ExtraHeader: Value
> Content-Length: 13
> Content-Type: application/x-www-form-urlencoded
>
* upload completely sent off: 13 out of 13 bytes
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 22 Feb 2021 18:38:15 GMT
< Server: Apache/2.4.37 (Oracle Linux) mod_fcgid/2.3.9
< Connection: close
< Content-Type: text/html; charset=UTF-8
<
<html>Echo POST:<br/><pre>
Raw POST data
</pre><br/>Echo headers, including QUERY_STRING:<br/><pre>
X_EXTRAHEADER=Value
PROXY_CONNECTION=close
QUERY_STRING=argument
</html>
* Closing connection 0
```

### 8.3. `netcat`

`netcat(1)`, also known as `nc`, is a versatile network utility. It can be used both as a client (like `telnet`) and a server.  *It is recommended to use `telnet` as a client, since it uses CRLF.*

Running as a server, you can see request your proxy to connect to it, so that you can see what your proxy is sending out.  Use `nc -klp PORT` to start `nc` as a server.  In the following example, we start the proxy and `netcat`, have the proxy connect to `netcat`, then have a look at what the proxy sent:

```shell
... proxy started in another session on port 3142 (src/proxy 3142)
... netcat started in another session on port 3143 (nc -klp 3143)
$ curl -v --proxy http://localhost:3142 http://localhost:3143/TEST -H 'TestHeader: t'
*   Trying ::1:3142...
* connect to ::1 port 3142 failed: Connection refused
*   Trying 127.0.0.1:3142...
* Connected to localhost (127.0.0.1) port 3142 (#0)
> GET http://localhost:3143/TEST HTTP/1.1
> Host: localhost:3143
> User-Agent: curl/7.72.0
> Accept: */*
> Proxy-Connection: Keep-Alive
>
```

`curl` stalls there, as `netcat` is not replying anything (this is not the goal).  We can now go to the session in which `nc` is started, and see:

```shell
$ nc -klp 3143
GET /TEST HTTP/1.0
Connection: close
TestHeader: t
Proxy-Connection: close
Accept: */*
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:10.0.3) Gecko/20120305 Firefox/10.0.3
Host: localhost:3143
```

We can see here that the extra header `TestHeader` was properly forwarded.

## 9. Provided libraries (use them!)

The handout comes packaged with three libraries, automatically linked to your program.  The headers are located at `~/prof/include/` and automatically included.

### 9.1. Some CS:APP functions, including `rio`

This is part of the code provided in the textbook.  The functions available are the following, make sure you are familiar with all of them:

```c
/**
 * FUNCTIONS FOR ALL PARTS.
 */

/* Rio (Robust I/O) package, for all parts */
ssize_t rio_readn(int fd, void* usrbuf, size_t n);
ssize_t rio_writen(int fd, const void* usrbuf, size_t n);
void    rio_readinitb(rio_t* rp, int fd);
ssize_t rio_readnb(rio_t* rp, void* usrbuf, size_t n);
ssize_t rio_readlineb(rio_t* rp, void* usrbuf, size_t maxlen);

/* Reentrant protocol-independent client/server helpers */
int open_clientfd(char* hostname, char* port);
int open_listenfd(char* port);

/* Wrapper */
int Open_listenfd(char* port);

/**
 * FUNCTIONS FOR PART 3.
 */

/* Pthreads thread control wrappers */
void Pthread_create(pthread_t* tidp, pthread_attr_t* attrp,
                    void*  (*routine)(void*), void* argp);
void Pthread_join(pthread_t tid, void** thread_return);
void Pthread_cancel(pthread_t tid);
void Pthread_detach(pthread_t tid);

/* POSIX semaphore wrappers. */
void Sem_init(sem_t* sem, int pshared, unsigned int value);
void P(sem_t* sem);
void V(sem_t* sem);
```

You are heavily encouraged to use the `rio` functions and the client/server helpers.  You should *not* however use `rio_readlineb` when you don’t know whether the input is ASCII text.  It is safe (and good!) to use when you are reading a request and its headers, but it is not safe when reading the client’s payload in a POST request, *or reading the web server’s answer*.

Other wrapper functions (e.g., `Rio_readn`) were removed as you should not use them: in case of an error, you should cleanly continue executing, while these functions would exit the proxy.

### 9.2. `sbuf`: Producer-consumer buffer

This is the producer-consumer thread-safe structure we introduced in class. This should be used in your prethreaded implementation (Part 3).   The interface is:

```c
void sbuf_init(sbuf_t* sp, int n);
void sbuf_deinit(sbuf_t* sp);
void sbuf_insert(sbuf_t* sp, int item);
int sbuf_remove(sbuf_t* sp);
```

### 9.3. `dict`: A dictionary structure

This is a dictionary structure with the following interface (more details in `~/prof/include/dict.h`):

```c
dict_t* dict_create ();
void    dict_destroy (dict_t* dic);

void    dict_put (dict_t* dic, const char* key, const char* val);
char*   dict_get (const dict_t* dic, const char* key);
void    dict_del (dict_t* dic, const char* key);
size_t  dict_size (const dict_t* dic);

typedef void (*dict_apply_fun_t) (const char* key, const char* val, size_t val_len, void* arg);
void    dict_apply (const dict_t* dic, const dict_apply_fun_t fun, void* arg);
```

This is very similar to the structure you had to develop for the DictLab.  In particular, you need not worry about memory allocation if you use this structure; these functions make local copies of keys and values.  In fact, using these dictionaries, the instructor’s implementation of the proxylab does not have a single call to `malloc` or `free`.  Here’s an example usage:

```c
dict_t* headers = dict_create ();
dict_put (headers, "Host", "Lorem");
dict_put (headers, "X-Test", "Ipsum");
printf ("Host is: %s\n", dict_get (headers, "Host")); // Prints "Host is: Lipsum"
```

The whole dictionary can be iterated using `dict_apply`, refer to the DictLab writeup if you need a refresher.  The only difference is that your function is also given the length of the value (see the type `dict_apply_fun_t` above).

After using a dictionary, you should make sure to free it:

```c
dict_destroy (headers);
```

The instructor used three `dict_t` structures in the whole project: the headers read from the client, a dictionary describing the request (with `"Port"`, `"URL"`, …), and a dictionary for the cache (mapping a URL to its cached contents). Arguably, fewer structures could have been used.

> **Note:** *If you are feeling adventurous,* you can use a different syntax to iterate through a dictionary.  This macro defines a local function called `_iter` and pass it to `dict_apply`:
>
> ```c
> #define dict_foreach(dict, fun)                                         \
>   do {                                                                  \
>     void _iter (const char* key, const char* val, size_t len, void* arg) { \
>       fun;                                                              \
>     }                                                                   \
>     dict_apply (dict, &_iter, NULL);                                    \
>   } while (0)
> ```
>
> As it is locally defined, the function can use all the variables that are already defined, so we don’t need to pass extra information as `arg`.  For instance:
>
> ```c
> int i = 0;
> dict_foreach (headers,
>               {
>                 printf ("[Header %d] %s: %s\n", ++i, key, val);
>               });
> ```
>
> This prints:
>
> ```
> [Header 1] X-Test: Ipsum
> [Header 2] Host: Lorem
> ```
>
> **Do not use that macro if you don’t fully understand how it works.**

#### 9.3.1. Special case: using the dictionary to cache pages

The cache stores a map from URL to data, it seems that dictionaries can be helpful for that.  However, the data returned by the server can be non-ASCII. The above interface is not friendly with this, since it finds the length of `val` by calling `strlen(3)`.  To deal with these, the interface is extended with:

```c
void    dict_putn (dict_t* dic, const char* key, const char* val, size_t val_len);
char*   dict_getn (const dict_t* dic, const char* key, size_t* pval_len);
```

The first one inserts a pair `(key, val)` in the dictionary, where `val` is of length `val_len`.  The second one retrieves a value associated with a key and sets the integer pointed by `pval_len` to the length of the value, if `pval_len` is non-`NULL`.  The function `dict_apply`  also has access to the value length.

## 10. Handing in Your Work

When you have completed the lab, you will submit it as follows:

```shell
$ git add src/proxy.c
$ git commit -m "update code"
$ git push
```

**You may commit and push your code as many times as you want. Just keep in mind that you will also need to submit the screenshot proof of your submission on D2L.**

## 11. Suggestions

- Use the libraries provided.  Using dictionaries throughout can save you a lot of work and make your code more readable.
- Don’t forget to close your file descriptors.  Working with file descriptors and threads is *much* simpler than with `fork(2)`, but leaving a file descriptor open is a recipe for disaster.
- Be sure to free what you allocate (dictionaries are destroyed using `dict_destroy`).  To track memory leaks, you can use:

  ```shell
  $ valgrind --tool=memcheck --leak-check=yes src/proxy 3142
  ```

  Then when you kill the proxy using Ctrl-C, Valgrind will report on the memory usage of your program.  Since you do not kill the threads, there should be a “memory leak” there.  It may also be that `getnameinfo` leaks a bit, this is a known problem.

- Your submission is comprised of everything in your `src/` folder.  If you wish to modify `src/Makefile` and add some headers and sources to produce a cleaner code, you can do it.  You should have to only modify the variables `HEADERS` and `SOURCES` therein.