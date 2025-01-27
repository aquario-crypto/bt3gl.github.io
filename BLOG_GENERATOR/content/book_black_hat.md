Title: Black Hat Python Networking: The Socket Module
Date: 2014-12-10
Category: Networking
Tags: Python, netcat, socket, threading, subprocess, UDP, TCP, Book, getopt, proxy

Last week I got my copy of [Black Hat Python](http://www.nostarch.com/blackhatpython), the new [Justin Seitz](https://twitter.com/jms_dot_py)'s book. The compilation talks about network programming, web hacking, and Windows exploitation. All in Python!

In this first post, I discuss Python's [socket](https://docs.python.org/2/library/socket.html) module, which contains all the tools to write [TCP](http://en.wikipedia.org/wiki/Transmission_Control_Protocol)/[UDP](http://en.wikipedia.org/wiki/User_Datagram_Protocol) clients and servers, including [raw sockets](http://en.wikipedia.org/wiki/Raw_socket). It's nice! **T-1000** is loving it!

Ah, by the way, all the source codes in this post are available at [my repo](https://github.com/go-outside-labs/My-Gray-Hacker-Resources).

![cyber](http://i.imgur.com/l3iyyOO.jpg)

---
## A TCP Client

Let's start from the beginning. Whenever you want to create a TCP connection with the **socket** module, you do two things: create a socket object and then connect to a host in some port:

```python
client = socket.socket( socket.AF_INET, socket.SOCK_STREAM )
client.connect(( HOST, PORT ))
```

The **AF_INET** parameter is used to define the standard IPv4 address (other options are *AF_UNIX* and *AF_INET6*). The **SOCK_STREAM** parameters indicate it is a **TCP** connection (other options are *SOCK_DGRAM*, *SOCK_RAW*, *SOCK_RDM*, *SOCK_SEQPACKET*).

All right, so the next thing you want to do is to send and receive data using socket's **send** and **recv** methods. And this should be good enough for a first script! Let's put everything together to create our TCP client:


```python
import socket

HOST = 'www.google.com'
PORT = 80
DATA = 'GET / HTTP/1.1\r\nHost: google.com\r\n\r\n'

def tcp_client():
 client = socket.socket( socket.AF_INET, socket.SOCK_STREAM)
 client.connect(( HOST, PORT ))
 client.send(DATA)
 response = client.recv(4096)
 print response

if __name__ == '__main__':
 tcp_client()
```

The simplicity of this script relies on making the following assumptions about the sockets:

* our *connection will always succeed*,
* the *server is always waiting for us to send data first* (as opposed to servers that expect to send data and then wait for response), and
* the server will always send us data back in a *short time*.

Let's run this script (notice that we get *Moved Permanently* because Google issues HTTPS connections):

```bash
$ python tcp_client.py
HTTP/1.1 301 Moved Permanently
Location: http://www.google.com/
Content-Type: text/html; charset=UTF-8
Date: Mon, 15 Dec 2014 16:52:46 GMT
Expires: Wed, 14 Jan 2015 16:52:46 GMT
Cache-Control: public, max-age=2592000
Server: gws
Content-Length: 219
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Alternate-Protocol: 80:quic,p=0.02

<HTML><HEAD><meta http-equiv="content-type" content="text/html;charset=utf-8">
<TITLE>301 Moved</TITLE></HEAD><BODY>
<H1>301 Moved</H1>
The document has moved
<A HREF="http://www.google.com/">here</A>.
</BODY></HTML>
```

Simple like that.

----------
## A TCP Server

Let's move on and write a *multi-threaded* TCP server. For this, we will use Python's [threading](https://docs.python.org/2/library/threading.html) module.

First, we define the IP address and port that we want the server to listen on. We then define a **handle_client** function that starts a thread to handle client connections. The function takes the client socket and gets data from the client, sending an **ACK** message.

The main function for our server, **tcp_server**, creates a server socket and starts listening on the port and IP (we set the maximum backlog of connections to 5). Then it starts a loop waiting for when a client connects. When this happens, it receives the client socket (the client variables go to the **addr** variable).

At this point, the program creates a thread object for the function **handle_client** which we mentioned above:

```python
import socket
import threading

BIND_IP = '0.0.0.0'
BIND_PORT = 9090

def handle_client(client_socket):
 request = client_socket.recv(1024)
 print "[*] Received: " + request
 client_socket.send('ACK')
 client_socket.close()

def tcp_server():
 server = socket.socket( socket.AF_INET, socket.SOCK_STREAM)
 server.bind(( BIND_IP, BIND_PORT))
 server.listen(5)
 print"[*] Listening on %s:%d" % (BIND_IP, BIND_PORT)

 while 1:
 client, addr = server.accept()
 print "[*] Accepted connection from: %s:%d" %(addr[0], addr[1])
 client_handler = threading.Thread(target=handle_client, args=(client,))
 client_handler.start()

if __name__ == '__main__':
 tcp_server()
```

We can run this script in one terminal and the client script (like the one we saw before) in a second terminal. Running the server:
```bash
$ python tcp_server.py
[*] Listening on 0.0.0.0:9090
```

Running the client script (we changed it to connect at 127.0.0.1:9090):
```bash
$ python tcp_client.py
ACK
```

Now, back to the server terminal, we successfully see the established connection:

```bash
$ python tcp_server.py
[*] Listening on 0.0.0.0:9090
[*] Accepted connection from: 127.0.0.1:44864
[*] Received: GET / HTTP/1.1
```

Awesome!



----------
## A UDP Client

UDP is an alternative protocol to TCP. Like TCP, it is used for packet transfer from one host to another. Unlike TCP, it is a *connectionless* and *non-stream oriented protocol*. This means that a UDP server receives incoming packets from any host without establishing a reliable pipe type of connection.

We can make a few changes in the previous script to create a UDP client connection:

* we use **SOCK_DGRAM** instead of **SOCK_STREAM**,
* because UDP is a connectionless protocol, we don't need to establish a connection beforehand, and
* we use **sendto** and **recvfrom** instead of **send** and **recv**.

```python
import socket

HOST = '127.0.0.1'
PORT = 9000
DATA = 'AAAAAAAAAA'

def udp_client():
 client = socket.socket( socket.AF_INET, socket.SOCK_DGRAM)
 client.sendto(DATA, ( HOST, PORT ))
 data, addr = client.recvfrom(4096)
 print data, adr

if __name__ == '__main__':
 udp_client()
```

-----

## A UDP Server

Below is an example of a very simple UDP server. Notice that there are no **listen** or **accept**:


```python
import socket

BIND_IP = '0.0.0.0'
BIND_PORT = 9000

def udp_server():
 server = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
 server.bind(( BIND_IP, BIND_PORT))
 print "Waiting on port: " + str(BIND_PORT)

 while 1:
 data, addr = server.recvfrom(1024)
 print data

if __name__ == '__main__':
 udp_server()
```

You can test it by running the server in one terminal and the client in another. It works and it's fun!

---------
## A Very Simple Netcat Client

Sometimes when you are penetrating a system, you wish you have [netcat](http://netcat.sourceforge.net/), which might be not installed. However, if you have Python, you can create a netcat network client and server.

The following script is the simplest netcat client setup one can have, extended from our TCP client script to support a loop.

In addition, now we use the **sendall** method. Unlike **send**, it will continue to send data until either all data has been sent or an error occurs (None is returned on success).

We also use **close** to release the resource. This does not necessarily close the connection immediately, so we use **shutdown** to close the connection in a timely fashion:


```python
import socket

PORT = 12345
HOSTNAME = '54.209.5.48'

def netcat(text_to_send):
 s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
 s.connect(( HOSTNAME, PORT ))
 s.sendall(text_to_send)
 s.shutdown(socket.SHUT_WR)

 rec_data = []
 while 1:
 data = s.recv(1024)
 if not data:
 break
 rec_data.append(data)

 s.close()
 return rec_data

if __name__ == '__main__':
 text_to_send = ''
 text_recved = netcat( text_to_send)
 print text_recved[1]
```




---------
## A Complete Netcat Client and Server


Let's extend our previous example to write a full program for a netcat server and client.

For this task we are going to use two special Python modules: [getopt](https://docs.python.org/2/library/getopt.html), which is a parser for command-line options (familiar to users of the C getopt()), and [subprocess](https://docs.python.org/2/library/subprocess.html), which allows you to spawn new processes.


### The Usage Menu
The first function we write is **usage**, with the options we want for our tool:

```python
def usage():
 print "Usage: netcat_awesome.py -t <HOST> -p <PORT>"
 print " -l --listen listen on HOST:PORT"
 print " -e --execute=file execute the given file"
 print " -c --command initialize a command shell"
 print " -u --upload=destination upload file and write to destination"
 print
 print "Examples:"
 print "netcat_awesome.py -t localhost -p 5000 -l -c"
 print "netcat_awesome.py -t localhost -p 5000 -l -u=example.exe"
 print "netcat_awesome.py -t localhost -p 5000 -l -e='ls'"
 print "echo 'AAAAAA' | ./netcat_awesome.py -t localhost -p 5000"
 sys.exit(0)
```

## Parsing Arguments in the Main Function
Now, before we dive in each specific functions, let's see what the **main** function does. First, it reads the arguments and parses them using **getopt**. Then, it processes them. Finally, the program decides if it is a client or a server, with the constant **LISTEN**:

```python
import socket
import sys
import getopt
import threading
import subprocess

LISTEN = False
COMMAND = False
UPLOAD = False
EXECUTE = ''
TARGET = ''
UP_DEST = ''
PORT = 0

def main():
 global LISTEN
 global PORT
 global EXECUTE
 global COMMAND
 global UP_DEST
 global TARGET

 if not len(sys.argv[1:]):
 usage()

 try:
 opts, args = getopt.getopt(sys.argv[1:],"hle:t:p:cu", \
 ["help", "LISTEN", "EXECUTE", "TARGET", "PORT", "COMMAND", "UPLOAD"])
 except getopt.GetoptError as err:
 print str(err)
 usage()

 for o, a in opts:
 if o in ('-h', '--help'):
 usage()
 elif o in ('-l', '--listen'):
 LISTEN = True
 elif o in ('-e', '--execute'):
 EXECUTE = a
 elif o in ('-c', '--commandshell'):
 COMMAND = True
 elif o in ('-u', '--upload'):
 UP_DEST = a
 elif o in ('-t', '--target'):
 TARGET = a
 elif o in ('-p', '--port'):
 PORT = int(a)
 else:
 assert False, "Unhandled option"

 # NETCAT client
 if not LISTEN and len(TARGET) and PORT > 0:
 buffer = sys.stdin.read()
 client_sender(buffer)

 # NETCAT server
 if LISTEN:
 if not len(TARGET):
 TARGET = '0.0.0.0'
 server_loop()

if __name__ == '__main__':
 main()
```


### The Client Function
The **client_sender** function is very similar to the netcat client snippet we have seen above. It creates a socket object and then it goes to a loop to send/receive data:

```python
def client_sender(buffer):
 client = socket.socket( socket.AF_INET, socket.SOCK_STREAM )

 try:
 client.connect(( TARGET, PORT ))

 # test to see if received any data
 if len(buffer):
 client.send(buffer)

 while True:
 # wait for data
 recv_len = 1
 response = ''

 while recv_len:
 data = client.recv(4096)
 recv_len = len(data)
 response += data
 if recv_len < 4096:
 break
 print response

 # wait for more input until there is no more data
 buffer = raw_input('')
 buffer += '\n'

 client.send(buffer)

 except:
 print '[*] Exception. Exiting.'
 client.close()
```

### The Server Functions
Now, let's take a look into the **server_loop** function, which is very similar to the TCP server script we saw before:

```python
def server_loop():
 server = socket.socket( socket.AF_INET, socket.SOCK_STREAM )
 server.bind(( TARGET, PORT ))
 server.listen(5)

 while True:
 client_socket, addr = server.accept()
 client_thread = threading.Thread( target =client_handler, \
 args=(client_socket,))
 client_thread.start()
```

The **threading** function calls **client_handler** which will either upload a file or execute a command (in a particular shell named *NETCAT*):
```python

def client_handler(client_socket):
 global UPLOAD
 global EXECUTE
 global COMMAND

 # check for upload
 if len(UP_DEST):
 file_buf = ''

 # keep reading data until no more data is available
 while 1:
 data = client_socket.recv(1024)
 if data:
 file_buffer += data
 else:
 break

 # try to write the bytes (wb for binary mode)
 try:
 with open(UP_DEST, 'wb') as f:
 f.write(file_buffer)
 client_socket.send('File saved to %s\r\n' % UP_DEST)
 except:
 client_socket.send('Failed to save file to %s\r\n' % UP_DEST)

 # Check for command execution:
 if len(EXECUTE):
 output = run_command(EXECUTE)
 client_socket.send(output)

 # Go into a loop if a command shell was requested
 if COMMAND:
 while True:
 # show a prompt:
 client_socket.send('NETCAT: ')
 cmd_buffer = ''

 # scans for a newline character to determine when to process a command
 while '\n' not in cmd_buffer:
 cmd_buffer += client_socket.recv(1024)

 # send back the command output
 response = run_command(cmd_buffer)
 client_socket.send(response)
```

Observe the two last lines above. The program calls the function **run_command** which use the **subprocess** library to allow a process-creation interface. This gives a number of ways to start and interact with client programs:

```python
def run_command(command):
 command = command.rstrip()
 print command
 try:
 output = subprocess.check_output(command, stderr=subprocess.STDOUT, \
 shell=True)
 except:
 output = "Failed to execute command.\r\n"
 return output
```

### Firing Up a Server and a Client
Now we can put everything together and run the script as a server in a terminal and as a client in another. Running as a server:

```bash
$ netcat_awesome.py -l -p 9000 -c
```

And as a client (to get the shell, press CTRL+D for EOF):

```bash
$ python socket/netcat_awesome.py -t localhost -p 9000
NETCAT:
ls
crack_linksys.py
netcat_awesome.py
netcat_simple.py
reading_socket.py
tcp_client.py
tcp_server.py
udp_client.py
NETCAT:
```

### The Good 'n' Old Request

Additionally, we can use our client to send out requests:

```bash
$ echo -ne "GET / HTTP/1.1\nHost: www.google.com\r\n\r\n" | python socket/netcat_awesome.py -t www.google.com -p 80
HTTP/1.1 200 OK
Date: Tue, 16 Dec 2014 21:04:27 GMT
Expires: -1
Cache-Control: private, max-age=0
Content-Type: text/html; charset=ISO-8859-1
Set-Cookie: PREF=ID=56f21d7bf67d66e0:FF=0:TM=1418763867:LM=1418763867:S=cI2xRwXGjb6bGx1u; expires=Thu, 15-Dec-2016 21:04:27 GMT; path=/; domain=.google.com
Set-Cookie: NID=67=ZGlY0-8CjkGDtTz4WwR7fEHOXGw-VvdI9f92oJKdelRgCxllAXoWfCC5vuQ5lJRFZIwghNRSxYbxKC0Z7ve132WTeBHOCHFB47Ic14ke1wdYGzevz8qFDR80fpiqHwMf; expires=Wed, 17-Jun-2015 21:04:27 GMT; path=/; domain=.google.com; HttpOnly
P3P: CP="This is not a P3P policy! See http://www.google.com/support/accounts/bin/answer.py?hl=en&answer=151657 for more info."
Server: gws
X-XSS-Protection: 1; mode=block
X-Frame-Options: SAMEORIGIN
Alternate-Protocol: 80:quic,p=0.02
Transfer-Encoding: chunked
(...)
```

Cool, huh?

------

## A TCP Proxy

A TCP proxy can be very useful for forwarding traffic and when assessing network-based software (for example, when you cannot run [Wireshark](http://singularity-sh.vercel.app/wiresharking-for-fun-or-profit.html), or you cannot load drivers or tools in the machine you are exploiting).

To create a proxy we need to verify if we need to *first initiate a connection* to the remote side. This will request data before going into our main loop, and some server daemons expect you to do this first (for instance, FTP servers send a banner first). We call this information **receive_first**.


### The Main Function
So let us start with our **main** function. First, we define the usage, which should have four more arguments together with **receive_first**. Then we check these arguments to variables and start a listening socket:

```python
import socket
import threading
import sys

def main():
 if len(sys.argv[1:]) != 5:
 print "Usage: ./proxy.py <localhost> <localport> <remotehost> <remoteport> <receive_first>"
 print "Example: ./proxy.py 127.0.0.1 9000 10.12.122.1 9999 True"
 sys.exit()

 local_host = sys.argv[1]
 local_port = int(sys.argv[2])
 remote_host = sys.argv[3]
 remote_port = int(sys.argv[4])

 if sys.argv[5] == 'True':
 receive_first = True
 else:
 receive_first = False

 server_loop(local_host, local_port, remote_host, remote_port, receive_first)
```


### The Server Loop Function

Like before we start creating a socket and binding this to a port and a host. Then we start a loop that accepts incoming connections and spawns a thread to the new connection:

```python
def server_loop(local_host, local_port, remote_host, remote_port, receive_first):
 server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

 try:
 server.bind(( local_host, local_port))
 except:
 print "[!!] Failed to listen on %s:%d" % (local_host, local_port)
 sys.exit()

 print "[*] Listening on %s:%d" % (local_host, local_port)
 server.listen(5)

 while 1:
 client_socket, addr = server.accept()
 print "[==>] Received incoming connection from %s:%d" %(addr[0], addr[1])

 # start a thread to talk to the remote host
 proxy = threading.Thread(target=proxy_handler, \
 args=(client_socket, remote_host, remote_port, receive_first))
 proxy.start()
```

### The Proxy Handler Functions

In the last two lines of the above snippet, the program spawns a thread for the function **proxy_handler** which we show below. This function creates a TCP socket and connects to the remote host and port. It then checks for the **receive_first** parameter. Finally, it goes to a loop where it:

1. reads from localhost (with the function **receive_from**),
2. processes (with the function **hexdump**),
3. sends to a remote host (with the function **response_handler** and **send**),
4. reads from a remote host (with the function **receive_from**),
5. processes (with the function **hexdump**), and
6. sends to localhost (with the function **response_handler** and **send**).

This keeps going until the loop is stopped, which happens when both local and remote buffers are empty. Let's take a look:

```python
def proxy_handler(client_socket, remote_host, remote_port, receive_first):
 remote_socket = socket.socket( socket.AF_INET, socket.SOCK_STREAM)
 remote_socket.connect(( remote_host, remote_port ))

 if receive_first:
 remote_buffer = receive_from(remote_socket)
 hexdump(remote_buffer)
 remote_buffer = response_handler(remote_buffer)

 # if we have data to send to client, send it:
 if len(remote_buffer):
 print "[<==] Sending %d bytes to localhost." %len(remote_buffer)
 client_socket.send(remote_buffer)

 while 1:
 local_buffer = receive_from(client_socket)
 if len(local_buffer):
 print "[==>] Received %d bytes from localhost." % len(local_buffer)
 hexdump(local_buffer)
 local_buffer = request_handler(local_buffer)
 remote_socket.send(local_buffer)
 print "[==>] Sent to remote."

 remote_buffer = receive_from(remote_socket)
 if len(remote_buffer):
 print "[==>] Received %d bytes from remote." % len(remote_buffer)
 hexdump(remote_buffer)
 remote_buffer = response_handler(remote_buffer)
 client_socket.send(remote_buffer)
 print "[==>] Sent to localhost."

 if not len(local_buffer) or not len(remote_buffer):
 client_socket.close()
 remote_socket.close()
 print "[*] No more data. Closing connections"
 break
```


The **receive_from** function takes a socket object and performs the receive, dumping the contents of the packet:

```python
def receive_from(connection):
 buffer = ''
 connection.settimeout(2)
 try:
 while True:
 data = connection.recv(4096)
 if not data:
 break
 buffer += data
 except:
 pass
 return buffer
```

The **response_handler** function is used to modify the packet contents from the inbound traffic (for example, to perform fuzzing, test for authentication, etc). The function **request_handler** does the same for outbound traffic:

```python
def request_handler(buffer):
 # perform packet modifications
 buffer += ' Yaeah!'
 return buffer

def response_handler(buffer):
 # perform packet modifications
 return buffer
```


Finally, the function **hexdump** outputs the packet details with hexadecimal and ASCII characters:

```python
def hexdump(src, length=16):
 result = []
 digists = 4 if isinstance(src, unicode) else 2
 for i in range(len(src), lenght):
 s = src[i:i+length]
 hexa = b' '.join(['%0*X' % (digits, ord(x)) for x in s])
 text = b''.join([x if 0x20 <= ord(x) < 0x7F else b'.' for x in s])
 result.append(b"%04X %-*s %s" % (i, length*(digits + 1), hexa, text))
```



### Firing Up our Proxy

Now we need to run our script with some server. For example, for an FTP server at the standard port 21:
```sh
$ sudo ./tcp_proxy.py localhost 21 ftp.target 21 True
[*] Listening on localhost:21
(...)
```




---
## Extra Stuff: The socket Object Methods

Additionally, let's take a quick look to all the methods available with the **socket** object from the **socket** module. I think it's useful to have an idea of this list:

* **socket.accept()**: Accept a connection.

* **socket.bind(address)**: Bind the socket to address.

* **socket.close()**: Close the socket.

* **socket.fileno()**: Return the socket's file descriptor.

* **socket.getpeername()**: Return the remote address to which the socket is connected.

* **socket.getsockname()**: Return the socket's own address.

* **socket.getsockopt(level, optname[, buflen])**: Return the value of the given socket option.

* **socket.listen(backlog)**: Listen for connections made to the socket. The backlog argument specifies the maximum number of queued connections.

* **socket.makefile([mode[, bufsize]])**: Return a file object associated with the socket.

* **socket.recv(bufsize[, flags])**: Receive data from the socket.

* **socket.recvfrom(bufsize[, flags])**: Receive data from the socket.

* **socket.recv_into(buffer[, nbytes[, flags]])**: Receive up to nbytes bytes from the socket, storing the data into a buffer rather than creating a new string.

* **socket.send(string[, flags])**: Send data to the socket.

* **socket.sendall(string[, flags])**: Send data to the socket.

* **socket.sendto(string, address)**: Send data to the socket.

* **socket.setblocking(flag)**: Set blocking or non-blocking mode of the socket.

* **socket.settimeout(value)**: Set a timeout on blocking socket operations.

* **socket.gettimeout()**: Return the timeout in seconds associated with socket operations, or None if no timeout is set.

* **socket.setsockopt(level, optname, value)**: Set the value of the given socket option.

* **socket.shutdown(how)**: Shut down one or both halves of the connection.

* **socket.family**: The socket family.

* **socket.type**: The socket type.

* **socket.proto**: The socket protocol.







-----

** Fun, isn't it? Now check the next post about Python's [scapy module](http://singularity-sh.vercel.app/black-hat-python-infinite-possibilities-with-the-scapy-module.html) and [paramiko module](http://singularity-sh.vercel.app/black-hat-python-the-paramiko-module.html)!**

## Further References:

- [Python's Socket Documentation](https://docs.python.org/2/library/socket.html)
- [Black Hat Python](http://www.nostarch.com/blackhatpython).
- [My Gray hat repo](https://github.com/go-outside-labs/My-Gray-Hacker-Resources).
- [A TCP Packet Injection tool](https://github.com/OffensivePython/Pinject/blob/master/pinject.py).
- [An asynchronous HTTP Proxy](https://github.com/OffensivePython/PyProxy/blob/master/PyProxy.py).
- [A network sniffer at the Network Layer](https://github.com/OffensivePython/Sniffy/blob/master/Sniffy.py).
- [A Guide to Network Programming in C++](http://beej.us/guide/bgnet/output/html/multipage/index.html).

