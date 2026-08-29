# Ports & Protocols: What You’re Actually Looking for When You Scan a Machine

*Learning cybersecurity from the ground up. Before running our first network scan, we need to understand what a scan is actually trying to find.*

At some point in a cybersecurity lab, you will probably see output that looks like this:

```text
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
```

It is easy to recognize these as “open ports” without really knowing what that means. Why does one machine have all these numbers attached to it? What is actually open? What does `tcp` tell us? And if port 22 usually means SSH, does seeing port 22 prove that SSH is running?

Before we use a tool to find open ports, it helps to understand what those ports represent in the first place.

## One Machine, Many Network Services

An IP address gives us a way to identify and communicate with a machine on a network. But one machine can be doing several things at the same time.

A server at `192.168.1.20` might host a website, allow administrators to connect remotely, and provide other network services. All of that communication can involve the same IP address, so the operating system needs another way to determine which application should receive incoming traffic.

That is where **ports** come in.

## Ports: Where Network Traffic Goes

A port is a numbered logical endpoint used with transport protocols such as TCP and UDP. Port numbers range from 0 to 65535, and unlike USB or HDMI ports, they are not physical openings on the computer.

For example, a machine might have:

```text
192.168.1.20

22    SSH
80    HTTP
443   HTTPS
```

The IP address identifies the machine or, more precisely, a network interface on it. The port helps identify the network endpoint associated with a particular application or service.

This is why an IP address alone is not always enough when looking at network communication. We often care about the **IP address and the port together**.

## Why Certain Port Numbers Keep Appearing

Some services are commonly associated with particular port numbers.

| Port | Common Service | Typical Use                       |
|------|----------------|-----------------------------------|
| 22   | SSH            | Remote command-line access        |
| 23   | Telnet         | Older unencrypted remote access   |
| 25   | SMTP           | Sending email                     |
| 53   | DNS            | Name resolution                   |
| 80   | HTTP           | Web traffic                       | 
| 443  | HTTPS          | Encrypted web traffic             |
| 445  | SMB            | Windows file and resource sharing |
| 3389 | RDP            | Windows remote desktop            |

You will naturally start recognizing some of these as you work with them. There is no reason to memorize hundreds of port numbers before doing anything practical.

There is also an important catch: **a port number does not guarantee which application is running there.**

SSH commonly uses TCP port 22, but it can be configured to use another port. A web application might listen on `8080` instead of `80`, and software can be configured in many different ways.

So if we discover that TCP port 22 is open, the careful conclusion is not automatically:

> *“This machine definitely runs SSH.”*

What we actually know is that **something appears to be listening on TCP port 22**. Since SSH commonly uses that port, SSH is a reasonable possibility that we can investigate further.

That distinction becomes important when we move from simply discovering ports to identifying the services and versions behind them.

## Open, Closed, and Filtered Ports

For TCP, when we describe a port as **open**, we generally mean that an application or service is listening there and accepting connection attempts.

Suppose a Linux server is running an SSH server that listens on TCP port 22. When appropriate traffic arrives for that address and port, the operating system can deliver it to the SSH server process.

```text
 Traffic arrives for TCP port 22
                ↓
         Operating system
                ↓
        SSH server process
```

This is why open ports matter in security. A network-accessible service creates something that other systems may be able to interact with.

But an open port is **not automatically a vulnerability**.

An SSH service might be fully patched, properly configured, and intentionally accessible. The security question is what is running there, how it is configured, who can reach it, and whether that exposure is expected.

You may also encounter **closed** and **filtered** ports.

A closed TCP port generally means the target is reachable, but there is no service accepting connections on that port. A filtered result means a scanner cannot reliably determine whether the port is open, often because something such as a firewall is affecting the traffic.

So later, if we see:

```text
22/tcp   open
23/tcp   closed
445/tcp  filtered
```

those three results are telling us different things about how the target responded.

We will see how a scanner determines these states when we actually use one.

## What “Listening” Actually Means

The word **listening** appears constantly in networking and security.

A program that expects incoming network connections can ask the operating system to listen on a particular address and port. The operating system then knows which process should receive matching traffic.

This gives us a useful connection between networking and what is actually running on a computer.

Suppose we discover something listening on TCP port `8080`. Knowing that 8080 is often used by web applications gives us a clue, but it does not tell us what is actually running.

A useful next question would be:

> ***Which process owns that port?***

From there, we might want to know which application started the process, which user it is running as, whether it is expected, and whether that service should be reachable from the network.

That is how a port number becomes useful security information rather than something we simply memorize.

## Source Ports and Destination Ports

Both sides of network communication use ports.

Suppose your laptop connects to a web server over HTTPS. The connection might look something like this:

```text
192.168.1.20:53142  →  203.0.113.10:443
```

The server is using destination port `443`, which is commonly associated with HTTPS. Your laptop also needs a port for its side of the communication, so the operating system may choose a temporary high-numbered port such as `53142`.

These temporary client-side ports are often called **ephemeral ports**.

This explains why security logs and packet captures can contain both a **source port** and a **destination port**. A high-numbered source port does not necessarily mean an unusual service is running on the client. It may simply be the temporary port selected for that connection.

Being able to read `source IP:port → destination IP:port` will become especially useful once we start looking at network traffic and logs.

## Where TCP and UDP Fit In

A port number does not exist in isolation. It is used with a transport protocol, most commonly **TCP** or **UDP**.

That is why you will see things written as:

```text
22/tcp
53/udp
443/tcp
```

TCP and UDP use the same range of port numbers, but they are separate transport protocols. TCP port 53 and UDP port 53 are therefore different endpoints even though the number is the same.

At a high level, **TCP is connection-oriented** and provides mechanisms for reliable, ordered delivery of data. **UDP is connectionless** and does not establish a connection in the same way or provide TCP’s built-in delivery guarantees.

We will look at TCP and UDP more closely when we can observe what they are actually doing on the network. For now, the important thing is that when you see `22/tcp`, you should read it as **TCP port 22**, not simply "port 22."

## Ports, Protocols, and Services

These three terms are closely related, but they are not identical.

A **protocol** defines rules for communication. HTTP, SSH, and DNS are examples.

A **service** is functionality provided by software running on a system. An SSH server, for example, provides remote access using the SSH protocol.

A **port** is part of how TCP or UDP traffic reaches the appropriate network endpoint on that system.

So if we eventually see:

```text
22/tcp   open   ssh
```

we can read it as: TCP port 22 appears open, and SSH is commonly associated with that port.

What we still do not know for certain is **what software is actually running there and which version it is**.

That becomes the next layer of information we will eventually learn to collect.

## Looking at Ports on Your Own Computer

We do not need a scanner yet to see that ports are real things our operating system is already managing.

We can ask our own computer which network connections exist and which ports are listening.

### Windows

Open Command Prompt and run:

```bash
netstat -ano
```

We are trying to inspect active network connections and listening ports on our own machine.

The output contains information such as the local address, foreign address, connection state, and PID. The **PID**, or process ID, can help us identify which process owns a connection or listening endpoint.

You may see something like `192.168.1.20:53142`. The part before the colon is the IP address, and the number after it is the port.

For TCP entries, `LISTENING` means a process is waiting for incoming connections. `ESTABLISHED` means a TCP connection currently exists.

### macOS

You can inspect processes using network connections with:

```bash
lsof -i
```

If we specifically want to see TCP processes listening for incoming connections, we can use:

```bash
lsof -iTCP -sTCP:LISTEN
```

Look at the process names and port information rather than trying to understand every column immediately.

### Linux

A common command is:

```bash
ss -tuln
```

Here, `-t` includes TCP sockets, `-u` includes UDP, `-l` shows listening sockets, and `-n` keeps the addresses and ports numeric.

The goal is not to memorize these commands or flags yet. We are simply using our operating system to answer a question:

> ***What network endpoints are currently listening on my computer?***

## Where Ports Show Up in Security Work

Ports are not something we learn only so scan output makes sense.

They appear in firewall rules, vulnerability scans, packet captures, security alerts, endpoint investigations, and network logs. Understanding them helps us interpret what systems are communicating and what services may be involved.

Imagine an analyst sees this in a network event:

```text
Source:       10.0.0.25:51544
Destination:  10.0.0.50:22
Protocol:     TCP
```

We can already read quite a bit from it.

The source machine is `10.0.0.25`, using source port `51544`. It is communicating with `10.0.0.50` on TCP port `22`.

Since TCP port 22 is commonly associated with SSH, that gives us context about what the connection may be.

But it does **not** tell us whether the connection is malicious.

It could be an administrator legitimately connecting to a server, an automated process, or activity that needs further investigation. We would need more evidence before reaching a conclusion.

This is an important habit in security work. A port number gives us context, not a verdict.

## What You Actually Need to Remember

You do not need to memorize all 65,536 port numbers. The common ones will become familiar naturally because you will keep seeing them.

What matters is being able to look at something like:

```text
443/tcp   open
```

and understand what it tells you.

We are looking at **TCP port 443**. The port appears to be **open**, meaning something is accepting connections there. HTTPS commonly uses TCP port 443, but the port number alone does not prove exactly what application or version is running.

That is enough to give us our next question:

**What is actually listening there?**

## Now We Have Something Worth Scanning For

An IP address can tell us which machine we are interested in, but it does not tell us which network services that machine exposes.

Ports help distinguish the network endpoints on that machine. TCP and UDP tell us which transport protocol is involved, and the services behind those ports tell us what the machine may actually be offering to the network.

Now this output has context:

```text
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
```

The next problem is practical. We do not want to manually try thousands of ports one by one just to find out which ones respond.

We need a tool that can systematically check them and report what it finds.

That is where we finally get to Nmap:

**Nmap From Zero: Your First Network Scan**
