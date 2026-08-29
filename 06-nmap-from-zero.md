# Nmap From Zero: Your First Network Scan

*Learning cybersecurity from the ground up. We know what ports represent. Now we can finally use a tool to discover them ourselves.*

So far, we have been looking at examples like:

```text
22/tcp   open   ssh
80/tcp   open   http
443/tcp  open   https
```

But in an actual lab, nobody hands us a neat list of the open ports on a machine.

We may know the IP address of a system we are authorized to test, but we still need to figure out what that system exposes to the network. Checking thousands of ports manually would obviously not make much sense.

This is where **Nmap** comes in.

Nmap stands for **Network Mapper**. It sends network probes to systems and interprets the responses it receives. Depending on how we use it, Nmap can help us discover reachable hosts, identify port states, and gather information about the systems and services available on a network.

Before we start adding options to Nmap commands, I want to understand what the tool is actually asking the network and what we can reasonably conclude from the answers.

## What Nmap Actually Does

Suppose we have an authorized lab machine at:

```text
192.168.1.50
```

We know where the machine is on the network, but we do not yet know what it exposes.

Maybe an SSH server is listening on TCP port 22. Maybe there is a web server on port 80. Maybe neither is accessible from our machine.

A port scan helps us investigate that.

Nmap sends probes toward ports on the target and observes what happens. Based on the responses it receives, or sometimes the responses it does not receive, Nmap can classify those ports into different states.

That gives us an important way to think about scanning:

**Nmap is not looking inside the target and reading a list of its open ports. It is interacting with the target over the network and interpreting its behavior.**

This is why the same system can sometimes appear differently depending on where we scan it from. Firewalls, routing, filtering, and network configuration can all affect what our scanner can reach and observe.

## Before We Scan Anything

Nmap is a legitimate tool used for network discovery, administration, troubleshooting, and security testing. However, scanning still means sending traffic to another system.

Throughout this series, I will only use systems I own, machines inside my own lab, or training environments that explicitly allow security testing.

If you are following along, do the same. **Only scan systems and networks you own or have clear permission to test.**

The goal here is to understand how these tools work in a controlled environment, not to test random systems on the internet.

## Our Lab Setup

For our practice, a simple setup is enough:

```text
Kali VM  ───── Lab Network ───── Target VM
```

Kali will be the machine running Nmap. The target will be another machine that we own and are allowed to test.

Before scanning, we need to know the target’s IP address. If the target is another Linux VM, for example, we can check its interfaces with:

```bash
ip addr
```

On Windows, we could use:

```bash
ipconfig
```

We are not running these commands because they are part of some Nmap ritual. We are answering a specific question:

**What is the IP address of the machine I intend to scan?**

Make sure you identify the correct interface and address rather than copying an IP from an example.

## Checking Nmap Before We Start

Kali commonly includes Nmap, so there is no reason to install another copy before checking.

Run:

```bash
nmap --version
```

This does not scan anything. It simply tells us whether Nmap is available and which version is installed.

Once Nmap is available and we know the address of our authorized target, we can perform the first scan.

## Running the First Scan

Suppose our target has the IP address:

```text
192.168.1.50
```

A basic scan can be started with:

```bash
nmap 192.168.1.50
```

Replace the example address with the actual IP address of your target.

With a normal scan like this, Nmap first performs host discovery unless that behavior is changed. If the target appears to be online, Nmap then scans its default selection of the **1,000 most common TCP ports**.

That last part is useful to understand.

A basic Nmap scan does not automatically check every possible TCP port. TCP and UDP port numbers range from 0 to 65535, with normal service scanning generally concerning ports 1 through 65535. Instead of checking the entire TCP range, Nmap’s default scan concentrates on 1,000 commonly used ports.

That gives us a practical first look at the target without immediately performing a full-port scan.

## Reading the Result Properly

A basic Nmap result may contain columns such as:

```text
PORT      STATE      SERVICE
```

For example:

```text
22/tcp    open       ssh
80/tcp    open       http
```

There is already quite a bit of information here.

`22/tcp` tells us we are looking at TCP port 22.

`open` tells us that Nmap received a response indicating that something is accepting TCP connections there.

`ssh` is the service name associated with that port in Nmap's service database.

That last point is easy to misunderstand.

A basic result showing:

```text
22/tcp   open   ssh
```

does not necessarily mean Nmap has identified the exact SSH software running on the target. Port 22 is conventionally associated with SSH, so Nmap can label the port accordingly.

The difference between **recognizing the service commonly associated with a port** and **actively identifying what service and version are actually responding** will become important when we go further with Nmap.

## Open, Closed, and Filtered

Nmap can report several port states, but the first three worth understanding are **open, closed, and filtered**.

An **open** TCP port means an application is accepting TCP connections on that port.

A **closed** TCP port means the target is reachable, but no application is accepting connections on that port.

A **filtered** port means Nmap cannot determine whether the port is open because packet filtering prevents its probes from reaching the port or prevents the expected response from reaching Nmap.

A firewall can cause this behavior, but seeing `filtered` does not prove exactly which device or rule caused it. Nmap is reporting what it can determine from its position on the network.

Consider:

```text
22/tcp    open
23/tcp    closed
445/tcp   filtered
```

These results tell us three different things.

Something is accepting TCP connections on port 22. The target responded in a way that tells us port 23 is closed. For port 445, Nmap does not have enough information to confidently classify it as open or closed because the traffic appears to be filtered.

This is one reason scan results should be interpreted as observations from a particular point on the network rather than a complete inventory of everything running on a machine.

## What Is Happening During a TCP Scan?

To understand how Nmap can tell that a TCP port is open, we need a little TCP.

A normal TCP connection begins with the **three-way handshake**:

```text
Client                                Server

SYN          ───────────────▶
             ◀──────────────          SYN-ACK
ACK          ───────────────▶
```

The client sends a SYN to request a connection.

If a service is listening and the traffic is allowed, the server responds with **SYN-ACK**.

The client replies with **ACK**, and the connection is established.

Nmap can use this behavior to learn about the state of a TCP port. If a probe to a port receives a SYN-ACK, that is strong evidence that the port is open.

If instead the target responds with a **RST**, or reset, the port is typically closed.

```text
Scanner                              Target

SYN          ───────────────▶
             ◀──────────────          RST
```

Now the difference between `open` and `closed` is no longer just terminology. It comes from different network behavior.

We will look at TCP in much more detail later. For now, this is enough to understand why Nmap can learn something about a port simply by sending carefully chosen packets and examining the response.

## SYN Scan and TCP Connect Scan

Two Nmap scan types are worth recognizing early because they approach TCP scanning differently.

A **TCP connect scan** can be requested with:

```bash
nmap -sT <target-ip>
```

This uses the operating system’s normal networking functions to establish a TCP connection.

For an open port, the normal handshake is completed:

```text
Scanner                               Target

SYN          ───────────────▶
             ◀──────────────          SYN-ACK
ACK          ───────────────▶
```

A **SYN scan** can be requested with:

```bash
nmap -sS <target-ip>
```

With a SYN scan, Nmap sends the initial SYN and examines the response without completing the normal connection.

For an open port, the simplified exchange looks like:

```text
Scanner                              Target

SYN          ───────────────▶
             ◀──────────────         SYN-ACK
RST          ───────────────▶
```

Once the SYN-ACK tells Nmap that the port is accepting connections, Nmap does not need to complete the connection. This is why SYN scanning is sometimes called a **half-open scan**.

SYN scanning generally requires privileges that allow Nmap to create and work with raw packets. On Linux, this often means running Nmap with appropriate elevated privileges or capabilities.

Without those privileges, Nmap may use a TCP connect scan instead.

The useful distinction is not just remembering `-sS` and `-sT`. One gives Nmap more direct control over the packets used to test the port, while the other relies on the operating system to establish a normal TCP connection.

## Host Discovery Comes First

There is another step happening before the normal port scan that can easily go unnoticed.

Nmap usually performs **host discovery** to determine whether a target appears to be online before spending time scanning its ports.

If Nmap concludes that a host appears down, the normal port scan may not continue.

But a machine can be online while ignoring or blocking the probes Nmap uses for discovery.

This is where you may encounter:

```text
-Pn
```

For example:

```bash
nmap -Pn <target-ip>
```

`-Pn` tells Nmap to skip host discovery and treat the target as online, proceeding with the requested scan.

It does not make an unreachable machine reachable, bypass a firewall, or fix every failed scan. It simply changes one part of Nmap’s behavior.

Instead of asking:

> *Is this host up before I scan it?*

we are telling Nmap:

> *Assume it is up and proceed with the scan.*

That is a much better way to remember what `-Pn` actually does.

## Choosing Which Ports to Scan

Sometimes we do not need Nmap’s default selection.

If we only want to check TCP port 22:

```bash
nmap -p 22 <target-ip>
```

For several specific ports:

```bash
nmap -p 22,80,443 <target-ip>
```

For a range:

```bash
nmap -p 1-1000 <target-ip>
```

And to scan all TCP ports from 1 through 65535:

```bash
nmap -p- <target-ip>
```

A full-port scan can reveal services running outside the ports included in Nmap’s default selection. This matters because applications do not have to use their conventional ports.

But that does not mean `-p-` should automatically appear in every Nmap command we run. A full scan takes longer and generates more traffic than checking a smaller set.

The useful habit is to know **why we are choosing a particular port range**.

## Discovering Hosts on a Network

Nmap can also work with more than one IP address.

Suppose our authorized lab network is:

```text
192.168.56.0/24
```

Instead of scanning one known host, we might first want to know which systems appear to be active on that network.

We can perform host discovery without the normal port scan using:

```bash
nmap -sn 192.168.56.0/24
```

The `-sn` option tells Nmap to perform host discovery without performing the normal port scan afterward.

You may hear this called a **ping scan**, although Nmap’s discovery process is not limited to traditional ICMP echo requests.

This gives us a useful distinction:

```bash
nmap -sn 192.168.56.0/24
```

asks which hosts appear to be active.

While:

```bash
nmap 192.168.56.20
```

goes further and scans the default TCP ports on a particular target after discovery.

This is the beginning of separating **host discovery** from **port discovery** instead of treating everything Nmap does as simply “scanning.”

## Why You Sometimes See `sudo nmap`

A lot of Nmap tutorials begin commands with:

```bash
sudo
```

On Linux, `sudo` allows an authorized user to execute a command with elevated privileges.

Some Nmap techniques require privileges that ordinary user accounts do not have, particularly when Nmap needs to construct and send raw packets itself.

That is relevant to SYN scanning. When Nmap has sufficient privileges for raw packet operations, SYN scanning is commonly used for TCP port scanning. Without those privileges, Nmap can use a TCP connect scan instead.

So:

```bash
sudo nmap ...
```

should not become something we type automatically just because we are using Kali.

The privilege level can change what Nmap is capable of doing, which is why it matters.

## A Note on Scan Speed

You may eventually encounter Nmap timing options from `-T0` through `-T5`.

For example:

```bash
nmap -T4 <target-ip>
```

These options influence how aggressively Nmap performs the scan.

Higher timing settings can make scanning faster under suitable network conditions, but speed is not free. More aggressive timing can increase traffic, affect reliability on unstable networks, and make the scan more noticeable.

For a small local lab, we do not need to optimize every second of scanning time.

There is a broader lesson here too. A command with six flags is not automatically better than a command with one. Start with the simplest scan that answers the question, understand the result, and add options when you actually need them.

## Reading a Scan as Evidence

Suppose our scan produces:

```text
Nmap scan report for 10.10.10.25
```

```text
PORT      STATE      SERVICE
22/tcp    open       ssh
80/tcp    open       http
445/tcp   filtered   microsoft-ds
3389/tcp  closed     ms-wbt-server
```

We can already make several observations.

The target appears reachable from our scanning position. TCP ports 22 and 80 are accepting connections. Those ports are commonly associated with SSH and HTTP.

Port 3389 is closed, so the target responded but there is no TCP service accepting connections there.

Port 445 is filtered from our perspective, so Nmap cannot confidently determine whether it is open or closed.

There are also several things we should **not** conclude from this scan.

We have not proved that port 22 is running a particular SSH implementation or version. We do not know which web server is behind port 80. We do not know why port 445 is filtered, and we have not discovered a vulnerability simply because a port is open.

The scan has given us information about the target’s **network exposure from our current position**.

It has also given us new questions.

That is what a useful scan should do.

## Try It on Your Own Lab

For this exercise, use a machine you own and are authorized to scan.

First, identify its IP address and make sure you know which machine you are targeting.

Then begin with:

```bash
nmap <target-ip>
```

Read the result before adding more options.

Look at which ports were reported, what state Nmap assigned to them, and which service names appear beside them.

Then choose one of those ports and scan it specifically:

```bash
nmap -p <port> <target-ip>
```

If you want to compare the default scan with the full TCP port range inside your lab, try:

```bash
nmap -p- <target-ip>
```

You can also compare a TCP connect scan and SYN scan where your environment and privileges allow it:

```bash
nmap -sT <target-ip>
```

```bash
sudo nmap -sS <target-ip>
```

Do not worry about getting an impressive result. A machine exposing only one service is enough to understand what the scan is doing.

## The Nmap Options We Have Used So Far

At this point, these options should have meaning rather than looking like random flags:

**1. `nmap <target>`**  
Perform a basic scan of a target

**2. `nmap -p 22 <target>`**  
Check TCP port 22

**3. `nmap -p 22,80,443 <target>`**  
Check selected TCP ports

**4. `nmap -p- <target>`**  
Scan TCP ports 1 through 65535

**5. `nmap -sn <network>`**  
Discover hosts without the normal port scan

**6. `nmap -Pn <target>`**  
Skip host discovery and treat the target as online

**7. `nmap -sT <target>`**  
Perform a TCP connect scan

**8. `nmap -sS <target>`**  
Perform a TCP SYN scan when sufficient privileges are available

The goal is not to memorize eight commands separately.

If you understand **the target, the question you are trying to answer, and what each option changes**, the syntax becomes much easier to remember.

## An Open Port Gives Us the Next Question

At this point, we can take an IP address, scan an authorized target, understand the basic port states Nmap reports, choose which ports to scan, distinguish host discovery from port scanning, and understand the difference between a TCP connect scan and a SYN scan.

But imagine our scan finds:

```text
22/tcp   open   ssh
```

We still do not know very much about what is actually behind that port.

Which SSH software is running?

Which version?

What if something is running on an unusual port and the port number gives us no useful clue at all?

Finding an open port tells us **where to look**. The next step is figuring out **what is actually there**.

That is where we go next:

**Nmap Part 2: Finding Services, Versions, and Understanding the Results**
