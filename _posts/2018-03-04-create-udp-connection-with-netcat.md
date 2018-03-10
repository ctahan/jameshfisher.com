---
title: "Creating a UDP connection with netcat"
tags: ["programming", "networking", "unix"]
---

The netcat command `nc` is most often used to create TCP connections,
but `nc` can also create UDP connections.
From my remote server, I start listening for UDP connections to UDP port `12345`:

```console
jim@remote:~$ nc -u -l 0.0.0.0 12345
```

I connect to this UDP server from my laptop using:

```console
jim@local:~$ nc -u -p 54321 personal.jameshfisher.com 12345
```

Above,
I use the `-u` flag to toggle UDP mode for listening and connecting,
and I use the `-p` flag to set the source UDP port on my laptop.
Before starting `nc` on either machine,
I started `tcpdump` on both machines, like this:

```console
jim@remote:~$ sudo tcpdump -n 'udp port 12345 or icmp'
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on ens4, link-type EN10MB (Ethernet), capture size 262144 bytes
```

(You'll see in a minute why I also included `icmp` in the filter.)
If we were using TCP,
we would have seen packets in this list as soon as we started the connection,
due to the TCP "handshake" to initiate a connection.
But with UDP,
no packets have been exchanged yet,
even though both `nc` processes are running.

So after starting both `nc` processes,
do we have a "connection"?
UDP is sometimes said to be "connectionless",
but I don't quite buy this.
My laptop at least is aware of a connection,
and we can see the connection with `lsof`:

```console
jim@local:~$ sudo lsof -nP -i UDP | awk 'NR == 1 || /12345/'
COMMAND     PID           USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
nc        41383            jim    3u  IPv4 0x86646b3906fdc4d      0t0  UDP 192.168.1.4:54321->35.190.176.201:12345
```

On the other hand,
the server is not yet aware of the connection;
`lsof` still only shows the process listening for a new connection:

```console
jim@remote:~$ sudo lsof -nP -i UDP | awk 'NR == 1 || /12345/'
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc       19075  jim    3u  IPv4  80196      0t0  UDP *:12345
```

Next, from my laptop,
I type "hi" and hit return.
This sends one UDP packet containing the string `"hi\n"`.
Finally, we see a UDP packet with `tcpdump`:

```
12:49:04.462186 IP 51.6.191.203.54321 > 10.142.0.2.12345: UDP, length 3
```

At this point, the server is aware of the connection,
and we can see it with `lsof`:

```console
jim@remote:~$ sudo lsof -nP -i UDP | awk 'NR == 1 || /12345/'
COMMAND    PID USER   FD   TYPE DEVICE SIZE/OFF NODE NAME
nc       19075  jim    3u  IPv4  80196      0t0  UDP 10.142.0.2:12345->51.6.191.203:54321
```

So UDP _does_ have connections,
but in contrast to TCP,
the UDP connection is only fully created once the first data packet is sent.

Next, I type "ack" on the server connection, and hit return.
A UDP packet goes in the other direction:

```
13:12:57.824337 IP 35.190.176.201.12345 > 192.168.1.4.54321: UDP, length 4
```

Next, I kill the `nc` process on the server:

```console
jim@remote:~$ nc -u -l 0.0.0.0 12345
hi
ack
^C
```

At this point,
if this were TCP, would expect more packets negotiating a connection shutdown.
But UDP doesn't have this handshake either, so nothing is exchanged.

After killing the server process,
do we have a connection?
According to the server, no.
But my laptop is completely unaware that the process was killed on the server,
and still shows an active connection:

```console
jim@local:~$ sudo lsof -nP -i UDP | awk 'NR == 1 || /12345/'
COMMAND     PID           USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
nc        41383            jim    3u  IPv4 0x86646b3906fdc4d      0t0  UDP 192.168.1.4:54321->35.190.176.201:12345
```

Because the `nc` process on the laptop still thinks there's a connection,
I can continue to send stuff with it.
I type "hello..?", and hit return.
On `tcpdump`, I see:

```
13:18:27.846128 IP 192.168.1.4.54321 > 35.190.176.201.12345: UDP, length 9
13:18:27.943967 IP 35.190.176.201 > 192.168.1.4: ICMP 35.190.176.201 udp port 12345 unreachable, length 45
```

My laptop duly sends the UDP packet,
but quickly receives a reply: "udp port 12345 unreachable"!
At this point,
my laptop knows the connection is gone,
and it is no longer listed:

```console
jim@local:~$ sudo lsof -nP -i UDP | awk 'NR == 1 || /12345/'
COMMAND     PID           USER   FD   TYPE            DEVICE SIZE/OFF NODE NAME
```

The "udp port 12345 unreachable" information is not in a UDP packet,
but an ICMP packet.
This is why I included `icmp` in the `tcpdump` filter.
ICMP is "Internet Control Message Protocol",
and one use is to notify remote hosts that hosts or ports are unreachable.
The ICMP packet was generated by my server
after it received the "hello..?" message:

```
13:18:27.898045 IP 51.6.191.203.54321 > 10.142.0.2.12345: UDP, length 9
13:18:27.898099 IP 10.142.0.2 > 51.6.191.203: ICMP 10.142.0.2 udp port 12345 unreachable, length 45
```

As with the initial connection setup,
the connection teardown is delayed in UDP until a packet is received.