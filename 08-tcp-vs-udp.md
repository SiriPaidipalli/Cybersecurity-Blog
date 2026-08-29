# TCP vs UDP: Understanding What Your Scans Are Actually Testing

*TCP and UDP are often introduced as “reliable versus fast.” That is useful as a starting point, but it is nowhere near enough to understand what we actually see in network scans, packet captures, firewall rules, logs, and security investigations.*

A network connection can look simple from the application side. We open a website, connect to a server over SSH, query DNS, stream media, or send a message, and data moves between two systems. Underneath that communication, however, the operating systems involved need to make several decisions. How should the data be delivered? Should the receiver confirm that it arrived? What happens if data is lost? Does communication need to establish a connection first? How does the receiving system know which application should receive the traffic?

Two protocols appear constantly when answering those questions: **TCP, the Transmission Control Protocol, and UDP, the User Datagram Protocol**. Both operate at the transport layer of the TCP/IP model, both use port numbers, and both carry data between applications. They behave very differently, though, and those differences directly affect how we scan networks, interpret packet captures, configure firewalls, investigate alerts, and reason about network activity.

Understanding TCP and UDP is therefore not just a networking requirement. It is part of understanding what network evidence actually tells us.

## Where TCP and UDP Fit in Network Communication

Network communication is handled by multiple layers working together. A protocol such as HTTPS does not independently move packets from one computer to another. It relies on protocols beneath it to provide transport and network delivery.

A simplified HTTPS connection might look like this:

```text
 Application Layer
   HTTP / HTTPS
        ↓
  Transport Layer
       TCP
        ↓
  Internet Layer
        IP
        ↓
 Network Interface
 Ethernet / Wi-Fi
```

Each layer has a different responsibility. IP is primarily concerned with addressing and moving packets between networks. TCP or UDP provides transport between applications. Application protocols such as HTTP, SSH, DNS, SMTP, and many others define how particular applications communicate.

This distinction matters because SSH is not an alternative to TCP. SSH normally **uses TCP**. Traditional HTTP/1.1 and HTTP/2 connections also normally use TCP, while HTTP/3 uses QUIC, which operates over UDP.

When we see something like:

```text
22/tcp
```

we are looking at two separate pieces of information. `22` is the port number, while `tcp` tells us which transport protocol is being used. TCP port 22 and UDP port 22 are separate endpoints. Sharing the same numerical port does not make them the same service or the same network path.

## Addresses, Ports, and Network Conversations

A network conversation is not identified only by a destination port. Suppose a client connects to a web server:

```text
192.168.1.25:53142 → 203.0.113.10:443
```

The client is using:

```text
Source IP:    192.168.1.25
Source Port:  53142
```

and the server is receiving traffic at:

```text
Destination IP:    203.0.113.10
Destination Port:  443
```

The transport protocol also matters. A TCP conversation is commonly identified using the combination of the source IP, source port, destination IP, and destination port. This is often called a **4-tuple**. When the transport protocol is included as well, the complete set is commonly described as a **5-tuple**.

```text
Source IP
Source Port
Destination IP
Destination Port
Transport Protocol
```

This idea appears constantly in security tools. Firewalls use these values when evaluating and tracking traffic. Packet-analysis tools use them to separate conversations. SIEM events frequently record them. Network detection systems use similar flow information when correlating activity.

It is also how a single server can handle thousands of connections to the same service simultaneously. Even when many clients connect to TCP port 443, each conversation has a different combination of addresses and ports.

## TCP: Connection-Oriented Communication

TCP is a **connection-oriented transport protocol**. Before normal application data is exchanged, TCP establishes communication between the endpoints and creates state that both systems can track.

This process begins with the TCP **three-way handshake**:

```text
Client                                       Server

SYN        ───────────────────────▶

           ◀───────────────────────         SYN-ACK

ACK        ───────────────────────▶
```

The client begins by sending a TCP segment with the **SYN** flag set. If the server is accepting connections on that destination port, it normally responds with **SYN-ACK**. The client then responds with **ACK**, completing the handshake.

The handshake is not simply a greeting between systems. It establishes information that TCP needs to manage the connection, including the initial sequence numbers used to track the data stream. Once the handshake completes, application data can begin moving through the connection.

This predictable handshake is also one reason TCP provides useful information during network scanning. Open, closed, and filtered TCP ports often produce observably different behavior.

## TCP Flags

The TCP header contains several control flags that help describe what is happening within a connection. The most important flags to recognize during early network and security analysis are **SYN, ACK, FIN, and RST**, although others also appear.

**SYN** is primarily associated with establishing TCP connections and synchronizing sequence numbers. **ACK** indicates that acknowledgment information in the TCP header is valid and is present throughout most established TCP communication.

**FIN** is used when one side wants to gracefully close its direction of a TCP connection. **RST**, or reset, immediately rejects or terminates a connection rather than allowing the normal graceful shutdown process to complete.

TCP also includes **PSH**, which is used to request timely delivery of buffered data to the receiving application, and **URG**, which indicates that the urgent pointer is significant. URG is much less common in ordinary modern traffic than SYN, ACK, FIN, and RST. Additional flags such as **ECE** and **CWR** are related to Explicit Congestion Notification.

You do not need to memorize every possible TCP flag immediately. Being able to recognize SYN, SYN-ACK, ACK, FIN, and RST already provides a large amount of useful context when looking at Nmap traffic, firewall logs, or Wireshark captures.

## Sequence Numbers and Acknowledgments

TCP is designed to provide applications with a **reliable, ordered byte stream**. To accomplish that, it needs a way to track which parts of the stream have been transmitted and received.

TCP uses **sequence numbers** to identify positions within the byte stream and **acknowledgment numbers** to indicate what data has been received and what byte the receiver expects next.

A simplified exchange could look like this:

```text
Sender                                             Receiver

Data Seq=1000       ───────────────────────▶

                    ◀───────────────────────       ACK=1500
```

At a simplified level, an acknowledgment of `1500` means the receiver has successfully received the preceding portion of the stream and is expecting the next byte beginning at sequence number 1500.

The real behavior is more sophisticated, but this concept explains why TCP can detect missing data, restore the correct order of received information, and retransmit data when necessary. TCP is not treating each packet as an isolated message. It is managing an ongoing stream of bytes.

## What Happens When TCP Data Is Lost

IP networks do not guarantee that every packet reaches its destination. Packets may be dropped because of network congestion, overloaded devices, wireless interference, routing problems, filtering, unstable links, or many other conditions.

TCP detects when expected data has not been acknowledged and retransmits when appropriate.

```text
Sender                                                 Receiver

Segment 1          ────────────────────────▶
Segment 2          ─────────── X
Segment 3          ────────────────────────▶

                    Missing data detected

Segment 2          ────────────────────────▶
```

The actual mechanisms include retransmission timers, acknowledgments, duplicate acknowledgments, and behaviors such as fast retransmit.

This is useful in security analysis because TCP retransmissions are visible in packet captures. A retransmission does **not** automatically indicate malicious activity. It may simply represent packet loss or network instability. An unusually high number of retransmissions, however, can provide evidence of connectivity problems, congestion, filtering, poor network conditions, or an unstable path.

The packet itself gives us an observation. Context tells us what that observation means.

## TCP Preserves Data Order

Packets do not necessarily arrive at exactly the same time or in exactly the same order in which they were transmitted. TCP uses sequence information to reconstruct the byte stream correctly before delivering it to the application.

If part of the stream arrives out of order, TCP can temporarily hold received data while waiting for the missing portion. This is one reason applications that require complete and correctly ordered information often use TCP.

Consider downloading a file. Receiving most of the bytes is not enough if a missing section corrupts the executable, archive, document, or webpage. TCP provides the transport mechanisms required to reconstruct the stream correctly before the application uses it.

## Flow Control

Reliability is not only about handling lost data. A sender may also be capable of transmitting information faster than the receiving system can process it.

TCP includes **flow control** to help prevent the sender from overwhelming the receiver. The receiving system advertises how much data it is currently prepared to accept using its TCP receive window. The sender uses this information when deciding how much unacknowledged data it can have in flight.

Flow control is therefore concerned with the capacity of the **receiving endpoint**.

## Congestion Control

TCP also includes mechanisms for reacting to congestion within the network itself. If systems inject data into a network faster than the path can carry it, packets may be dropped and network performance may deteriorate.

TCP implementations use congestion-control algorithms to adjust how aggressively they transmit based on observed network conditions. Concepts such as slow start and congestion avoidance can become quite detailed, but the distinction between flow control and congestion control is worth remembering.

**Flow control is concerned with preventing the sender from overwhelming the receiver. Congestion control is concerned with avoiding excessive load on the network path.**

They address related problems, but they are not the same mechanism.

## Closing a TCP Connection

Because TCP creates a connection, it also has mechanisms for closing that connection cleanly. TCP is full-duplex, which means each direction of communication can be closed independently.

A simplified graceful shutdown can look like this:

```text
Client                                        Server

FIN           ───────────────────────▶

              ◀───────────────────────         ACK

              ◀───────────────────────         FIN

ACK           ───────────────────────▶
```

This is why TCP termination is commonly described as a four-step process. In actual packet captures, the FIN flag is commonly seen together with ACK as `FIN, ACK` because established TCP communication normally already has the ACK flag set.

Not every connection ends gracefully. A system can send a **RST** to immediately reset or reject a TCP connection. Seeing FIN generally suggests an orderly attempt to close communication, while RST indicates that a connection is being reset, rejected, or abruptly terminated.

Neither one is automatically suspicious. The surrounding traffic and system activity determine whether it matters.

## TCP Connection States

Because TCP maintains state, operating systems track where each connection currently sits in its lifecycle.

On Linux, a command such as:

```bash
ss -tan
```

may display TCP states including:

```text
LISTEN
ESTAB
SYN-SENT
SYN-RECV
FIN-WAIT-1
FIN-WAIT-2
TIME-WAIT
CLOSE-WAIT
```

`LISTEN` means a socket is waiting for incoming TCP connections. `ESTAB`, short for established, means the TCP handshake has completed and a connection currently exists.

`SYN-SENT` means the local system has sent a SYN and is waiting for a response, while `SYN-RECV` means a SYN has been received and the system has responded but the handshake has not yet fully completed.

States involving FIN and TIME-WAIT appear during connection termination. You do not need to memorize the complete TCP state machine to interpret basic security traffic, but understanding that TCP connections move through defined states becomes extremely useful when examining endpoints, firewalls, network captures, and connection failures.

## Why TCP Produces Clearer Scan Results

The structured behavior of TCP provides scanners with useful responses.

Suppose a scanner sends a SYN to a TCP port. If the port is open and accepting connections, the target normally responds with SYN-ACK:

```text
Scanner                                     Target

SYN           ───────────────────────▶

              ◀───────────────────────       SYN-ACK
```

If the target is reachable but the TCP port is closed, it commonly responds with RST:

```text
Scanner                                     Target

SYN           ───────────────────────▶

              ◀───────────────────────        RST
```

If filtering prevents useful communication, the scanner may receive no response or another indication that leads it to classify the port as `filtered`.

This is why Nmap can produce results such as:

```text
22/tcp   open
23/tcp   closed
445/tcp  filtered
```

These states are based on **observed network behavior**. Nmap is not remotely reading a configuration file that tells it which ports are open. It sends probes, observes responses or the absence of responses, and makes classifications based on that evidence.

## TCP Connect Scanning

A TCP connect scan asks the operating system to perform an ordinary TCP connection attempt.

With Nmap:

```bash
nmap -sT <target-ip>
```

If a port is open, the normal TCP handshake is completed:

```text
Scanner                                         Target

SYN            ───────────────────────▶

               ◀───────────────────────         SYN-ACK

ACK            ───────────────────────▶
```

Because a complete TCP connection is established, the connection may be recorded by the target's operating system, application, firewall, or monitoring tools depending on their configuration.

The important idea behind `-sT` is that Nmap is relying on the operating system's normal networking behavior to establish the connection.

## SYN Scanning

A SYN scan behaves differently. With sufficient privileges, Nmap can construct the probes directly:

```bash
sudo nmap -sS <target-ip>
```

When the destination port is open:

```text
Scanner                                        Target

SYN          ───────────────────────▶

             ◀───────────────────────          SYN-ACK

RST          ───────────────────────▶
```

Once Nmap receives SYN-ACK, it already has enough evidence to classify the TCP port as open. It therefore does not need to complete the normal connection by sending the final handshake ACK. Instead, the connection is reset.

This behavior is why SYN scanning is commonly called **half-open scanning**.

You may also hear it called a “stealth scan,” but that description should be treated carefully. Modern firewalls, intrusion detection systems, packet captures, endpoint monitoring tools, and network sensors can still observe SYN scan activity.

Failing to complete the three-way handshake does not make the traffic invisible.

## UDP: Connectionless Communication

UDP works differently because it does not establish a TCP-style connection before sending data.

A system can simply send a UDP datagram to another IP address and UDP port:

```text
  Client                                           Server

UDP Datagram       ───────────────────────▶
```

There is no SYN/SYN-ACK exchange, no transport-layer acknowledgment confirming that the receiver obtained the datagram, no TCP-style sequence numbers for reconstructing a reliable byte stream, and no built-in TCP-style retransmission mechanism.

This makes UDP comparatively simple, but it also means that any additional reliability or session behavior required by the communication must be provided elsewhere.

## What UDP Provides

Connectionless does not mean useless or uncontrolled. UDP still provides the information required to deliver datagrams between applications.

Its header includes:

```text
Source Port
Destination Port
Length
Checksum
```

The source and destination port fields identify application endpoints. The length field records the size of the UDP datagram, and the checksum helps detect corruption during transmission.

In IPv4, a UDP checksum can technically be disabled, although in IPv6 UDP checksums are normally required.

What UDP does not provide by itself is TCP's connection establishment, ordered byte stream, transport-layer acknowledgment system, retransmission logic, flow control, or TCP congestion-control mechanisms.

If communication over UDP needs some of those properties, another protocol above UDP or the application itself must provide them.

## UDP Does Not Mean Unreliable Applications

One of the most common shortcuts used when teaching networking is:

```text
TCP = reliable
UDP = unreliable
```

This is useful only if we understand what it really means.

TCP includes reliability mechanisms as part of the transport protocol itself. UDP does not. That does **not** mean every application using UDP is inherently unreliable.

Protocols operating over UDP can implement their own acknowledgments, retransmissions, sequencing, encryption, congestion control, connection management, and other behavior.

A major modern example is **QUIC**. QUIC operates over UDP but provides features needed for reliable modern network communication at a higher layer. HTTP/3 uses QUIC instead of traditional HTTP over TCP.

A more accurate distinction is therefore:

**TCP provides reliability and ordering mechanisms directly as part of TCP. UDP provides datagram transport and leaves additional behavior to protocols or applications above it.**

That explanation is much more useful than simply remembering “TCP reliable, UDP unreliable.”

## Why Applications Use UDP

If TCP provides so much functionality, it is reasonable to wonder why applications use UDP at all.

Not every application needs TCP's exact behavior. Some applications care more about low-latency communication, simple request-response exchanges, preserving message boundaries, broadcast or multicast capabilities, or controlling reliability at a different layer.

Protocols and applications commonly associated with UDP include:

```text
DNS
DHCP
NTP
SNMP
VoIP
Real-time media
Online gaming
QUIC / HTTP/3
```

These associations should not be treated as universal rules because many application protocols can use more than one transport depending on the situation.

DNS is a good example. Traditional DNS queries often use **UDP port 53** because a relatively small request and response can be exchanged efficiently without establishing a TCP connection first.

DNS can also use **TCP port 53**. Large responses, zone transfers, fallback situations, and various modern DNS behaviors may involve TCP.

Memorizing:

```text
DNS = UDP
```

would therefore be incomplete.

A better understanding is that DNS can use both UDP and TCP depending on what the communication requires.

## TCP Is Stream-Oriented, UDP Is Datagram-Oriented

Another important difference appears in how the two protocols present data to applications.

TCP provides a **byte stream**. It does not preserve application message boundaries in the same way UDP preserves individual datagrams. Applications using TCP therefore need their own mechanism for determining where application messages begin and end.

UDP is **datagram-oriented**. Each UDP datagram is delivered as an individual message at the transport layer.

This difference matters when designing software and when interpreting network captures. TCP gives the application an ordered stream of bytes, while UDP gives the application separate datagrams.

## Comparing TCP and UDP Headers

The difference in complexity is visible in the protocol headers themselves.

A UDP header contains only four major fields:

```text
Source Port
Destination Port
Length
Checksum
```

A TCP header contains considerably more information, including:

```text
Source Port
Destination Port
Sequence Number
Acknowledgment Number
Header Length
Flags
Window Size
Checksum
Urgent Pointer
Options
```

The additional TCP fields support capabilities such as connection state, acknowledgments, ordered delivery, retransmission, and flow control.

You do not need to memorize every bit in a TCP header, but being able to recognize fields such as ports, sequence numbers, acknowledgment numbers, flags, window size, and checksum becomes very useful when working with Wireshark or other packet-analysis tools.

## Why UDP Scanning Is More Ambiguous

The absence of a handshake makes UDP scanning fundamentally different from TCP scanning.

Suppose a scanner sends a UDP probe to a destination port and the application responds:

```text
Scanner                                              Target

UDP Probe          ───────────────────────▶

                   ◀───────────────────────         UDP Response
```

That response provides useful evidence that something is listening and responding on the UDP port.

Now consider a different result:

```text
Scanner                                              Target

UDP Probe          ───────────────────────▶

                   No response
```

That silence is difficult to interpret.

The port might be open, but the application may only respond to correctly formatted protocol-specific requests. A firewall may have dropped the incoming probe. A firewall may have blocked the response. The response could have been lost. The host or network device may also be rate-limiting traffic.

Unlike TCP, an open UDP port does not have a SYN-ACK response that it is required to send.

This uncertainty is one of the most important things to understand about UDP scanning.

## ICMP and Closed UDP Ports

Sometimes a UDP probe sent to a closed port causes the target to return an ICMP error, commonly an **ICMP Destination Unreachable: Port Unreachable** message.

```text
Scanner                                                  Target

UDP Probe          ───────────────────────▶

                   ◀───────────────────────        ICMP Port Unreachable
```

That response gives the scanner strong evidence that the destination UDP port is closed.

When no useful response arrives, however, the scanner may be unable to determine whether the port is open or whether filtering prevented communication.

This is why UDP scan results may include states such as:

```text
open
closed
filtered
open|filtered
```

`open|filtered` is particularly important. It means the scanner cannot confidently distinguish between an open port that did not respond and a port for which filtering prevented the expected response.

That classification is not Nmap being vague. It represents genuine uncertainty in the network evidence available to the scanner.

## Scanning UDP With Nmap

Nmap performs UDP scanning with:

```bash
sudo nmap -sU <target-ip>
```

UDP scans can take significantly longer than basic TCP scans because silence requires more careful interpretation. Nmap may need to wait for timeouts, retry probes, or account for ICMP rate limiting before deciding how to classify a port.

Rather than immediately scanning all 65,535 UDP ports, it is often more practical to begin with ports relevant to the services being investigated.

For example:

```bash
sudo nmap -sU -p 53,123,161 <target-ip>
```

This checks UDP ports commonly associated with DNS, NTP, and SNMP.

The port numbers provide useful context, but they still do not prove which applications are actually running. Additional service detection or protocol-specific investigation may be needed.

As with any network scanning, these commands should only be used against systems and networks you own or have explicit authorization to test.

## TCP and UDP Can Use the Same Port Number

TCP and UDP maintain separate port spaces, which means the same numerical port can exist for both protocols at the same time.

A system might show:

```text
53/tcp   open
53/udp   open
```

These are separate transport endpoints.

A firewall rule allowing UDP destination port 53 does not automatically allow TCP destination port 53. Similarly, finding UDP port 161 open tells us nothing by itself about TCP port 161.

Whenever a port number appears in a firewall rule, packet capture, Nmap scan, or security event, the **transport protocol is part of the meaning**.

Saying:

> “Port 53 is open.”

can therefore be incomplete.

A more precise statement would be:

> “UDP port 53 appears open.”

or:

> “TCP and UDP port 53 are both reachable from this network position.”

Precision matters because security analysis depends on describing what the evidence actually supports.

## TCP and UDP in Firewall Rules

Firewalls commonly evaluate information such as:

```text
Source IP
Destination IP
Transport Protocol
Source Port
Destination Port
Connection State
```

A firewall might permit:

```text
TCP destination port 443
```

while blocking other inbound TCP connections. Another rule might permit:

```text
UDP destination port 53
```

to a designated DNS server.

Because TCP has an explicit connection lifecycle, **stateful firewalls** can track established TCP sessions and recognize traffic that belongs to those sessions.

If an internal client establishes an outbound TCP connection, the firewall can remember that connection and permit the corresponding return traffic according to its policy.

UDP has no TCP-style connection lifecycle. Stateful firewalls can still maintain temporary state for UDP flows by tracking addresses, ports, direction, and timeouts, but they are not following a SYN → SYN-ACK → ACK connection state machine because UDP has no such handshake.

This becomes useful when troubleshooting or investigating situations where traffic appears to work in one direction but not another.

## TCP and UDP in Packet Captures

Packet analysis makes the difference between TCP and UDP immediately visible.

A TCP conversation in Wireshark may contain a pattern such as:

```text
SYN
SYN, ACK
ACK
Application Data
ACK
Application Data
ACK
FIN, ACK
ACK
```

A simple UDP request-response exchange might contain only:

```text
UDP Request
UDP Response
```

Other UDP traffic may appear as a sequence of independent datagrams:

```text
UDP Datagram
UDP Datagram
UDP Datagram
```

There are no TCP acknowledgment packets between them because UDP itself does not implement that mechanism.

Useful Wireshark display filters include:

```text
tcp
```

for TCP traffic and:

```text
udp
```

for UDP traffic.

Filters can also be made more specific:

```text
tcp.port == 443
```

or:

```text
udp.port == 53
```

Once we know what normal TCP and UDP behavior looks like, packet captures stop looking like random rows of addresses and numbers. We can begin asking what each packet is doing and whether the sequence of packets makes sense.

## What a Successful TCP Handshake Actually Proves

Suppose a packet capture shows:

```text
10.0.0.15:53422 → 10.0.0.20:22    SYN
10.0.0.20:22    → 10.0.0.15:53422 SYN-ACK
10.0.0.15:53422 → 10.0.0.20:22    ACK
```

From this evidence, we can reasonably conclude that a TCP connection was successfully established from `10.0.0.15` to TCP port 22 on `10.0.0.20`.

Because TCP port 22 is conventionally associated with SSH, SSH is a reasonable hypothesis. The handshake alone, however, does not prove that the application was actually SSH. It does not tell us whether a user successfully authenticated, what account was involved, what commands may have been executed, or whether the connection was malicious.

Those conclusions require additional evidence such as application logs, authentication logs, service identification, endpoint telemetry, or packet contents where available.

This distinction is central to security analysis. Network evidence tells us what happened on the network. Additional context helps us determine what that activity means.

## What Repeated TCP Connection Attempts Can Tell Us

Now imagine that a source sends SYN packets to many destination ports on the same target:

```text
10.0.0.15 → 10.0.0.20:21   SYN
10.0.0.15 → 10.0.0.20:22   SYN
10.0.0.15 → 10.0.0.20:23   SYN
10.0.0.15 → 10.0.0.20:25   SYN
10.0.0.15 → 10.0.0.20:80   SYN
10.0.0.15 → 10.0.0.20:443  SYN
```

A single SYN is completely normal. A rapid sequence of connection attempts across many destination ports, however, may be consistent with port scanning or automated service discovery.

Even then, we should not immediately call the activity malicious. The source could be an authorized vulnerability scanner, asset-discovery system, administrator, monitoring platform, penetration test, or security lab.

The packet pattern gives us a **lead**. Context determines whether the activity is expected or suspicious.

## SYN Floods and TCP Abuse

The TCP handshake can also be abused.

When a server receives a SYN, it may allocate resources while waiting for the connection to complete. An attacker can attempt to create large numbers of incomplete handshakes by sending SYN packets without completing the final step.

```text
Attacker                                              Server

SYN                 ───────────────────────▶
SYN                 ───────────────────────▶
SYN                 ───────────────────────▶
SYN                 ───────────────────────▶
SYN                 ───────────────────────▶
```

This is the basic idea behind a **SYN flood**, a type of denial-of-service attack.

Modern operating systems and network devices include defenses such as SYN cookies, rate controls, connection limits, and filtering, so real-world behavior is more complex than this simplified diagram.

For security analysis, the important concept is that unusually high numbers of SYN packets combined with large numbers of incomplete TCP handshakes may be meaningful evidence.

Understanding the handshake explains why the attack is possible and why defenders look for that particular pattern.

## UDP Reflection and Amplification

UDP's connectionless behavior creates a different type of abuse.

Because UDP does not establish a connection through a handshake, some UDP-based services can be abused in **reflection and amplification attacks** when attackers are able to spoof source IP addresses.

A simplified attack might look like this:

```text
Attacker
   |
   | Small UDP request
   | Spoofed Source IP = Victim
   ↓
UDP Service
   |
   | Larger response
   ↓
 Victim
```

The attacker sends a request to a UDP service but places the victim's IP address in the source field. The server therefore sends its response to the victim rather than the attacker.

When the response is significantly larger than the original request, the traffic is **amplified**. When many servers participate at the same time, substantial traffic can be directed toward the victim.

Services using protocols such as DNS, NTP, and SSDP have historically been abused for this purpose when they were exposed or misconfigured.

The lesson is not that UDP itself is insecure. The lesson is that protocol behavior influences what types of attacks are possible.

## Source Ports and Ephemeral Ports

Security logs frequently contain both source and destination ports.

Consider:

```text
Source:       10.0.0.25:54821
Destination:  10.0.0.50:443
Protocol:     TCP
```

The destination port `443` gives us useful context because it is commonly associated with HTTPS.

The source port `54821` is likely an **ephemeral port**, a temporary port selected by the client operating system for that connection.

A high-numbered source port therefore does not automatically mean a server or service is listening on that port. Source and destination ports play different roles depending on the direction and context of the communication.

This distinction becomes particularly important when reading firewall logs, NetFlow-style records, packet captures, and SIEM events.

## Stateful and Stateless Depend on the Layer

TCP is often described as **stateful**, while UDP is described as **stateless** or connectionless. That description is useful when discussing the transport protocols themselves.

TCP maintains explicit connection state. UDP does not establish a TCP-style connection or maintain a TCP connection state machine.

The word *stateful*, however, also appears in other parts of cybersecurity. A firewall can maintain temporary state about UDP traffic even though UDP itself is connectionless. An application using UDP can maintain its own session information. A monitoring system can group multiple UDP packets into the same observed network flow.

When someone describes something as stateful or stateless, the useful follow-up question is:

**At which layer?**

The transport protocol, firewall, application, and monitoring system can each maintain different kinds of state.

## TCP and UDP Are Not Security Labels

TCP should not be treated as secure simply because it is reliable, and UDP should not be treated as insecure simply because it is connectionless.

Neither protocol automatically provides application-level confidentiality, authentication, or authorization.

TCP can transport completely unencrypted application protocols. UDP can transport encrypted modern protocols.

Traditional HTTP over TCP can be plaintext, while QUIC over UDP uses TLS-based cryptographic protection as part of modern HTTP/3 communication.

Transport behavior and security properties are related, but they are not the same question.

When evaluating network communication, we need to ask both:

```text
How is the data being transported?
```

and:

```text
How is the communication being protected?
```

The answer to one does not automatically answer the other.

## Choosing Between TCP and UDP

Neither protocol is universally better. Applications choose transport behavior based on what their communication requires.

TCP is useful when applications benefit from:

```text
Reliable delivery
Ordered data
Connection state
Built-in retransmission
Flow control
Congestion control
```

UDP is useful when applications benefit from characteristics such as:

```text
No TCP connection-establishment handshake
Message-oriented datagrams
Lower transport overhead
Application-controlled reliability
Low-latency communication patterns
Broadcast or multicast where applicable
```

The choice depends on the needs of the protocol or application, not on one transport protocol being inherently superior to the other.

## TCP and UDP at a Glance

The differences can now be summarized without reducing the entire topic to “reliable versus fast.”

|      Characteristic     |                 TCP                |                   UDP                  |
|-------------------------|------------------------------------|----------------------------------------|
| Connection model        | Connection-oriented                | Connectionless                         |
| Handshake               | Three-way handshake                | No TCP-style handshake                 |
| Reliability mechanisms  | Built into TCP                     | Not provided by UDP itself             |
| Ordered delivery        | Yes                                | No                                     |
| Built-in retransmission | Yes                                | No                                     |
| Sequence tracking       | Yes                                | No TCP-style sequence tracking         |
| Flow control            | Yes                                | No                                     |
| Congestion control      | Built into TCP                     | Not provided by UDP itself             |
| Data model              | Byte stream                        | Datagrams                              |
| Header complexity       | Higher                             | Lower                                  |
| Open-port scanning      | Usually provides clearer responses | Often more ambiguous                   |
| Common scan states      | Open, closed, filtered             | Open, closed, filtered, open\|filtered |

The table is useful for revision, but the behavior behind each row is what makes the distinction useful in practice.

## Reading Network Evidence With TCP and UDP in Mind

Suppose a firewall event contains:

```text
Source IP:         10.0.0.25
Source Port:       51544
Destination IP:    10.0.0.50
Destination Port:  22
Protocol:          TCP
Action:            ALLOW
```

From this event, we can say that TCP traffic from `10.0.0.25:51544` was allowed toward TCP port 22 on `10.0.0.50`.

Because TCP port 22 is commonly associated with SSH, SSH is a reasonable possibility. We cannot conclude from this event alone that someone successfully logged in, that the activity was malicious, or even that the application listening on the port was definitely SSH.

Now consider:

```text
Source IP:         10.0.0.25
Destination IP:    8.8.8.8
Destination Port:  53
Protocol:          UDP
```

A DNS query would be a reasonable hypothesis because UDP port 53 is commonly used for DNS. To confirm that interpretation, we could inspect the packet contents, DNS telemetry, endpoint activity, resolver logs, or other available evidence.

This is a better approach to security analysis than memorizing ports and assigning conclusions immediately.

## Questions Worth Asking During an Investigation

When TCP or UDP traffic appears in a scan, packet capture, firewall log, flow record, or security alert, the transport information gives us a starting point for investigation.

Useful questions include:

```text
What are the source and destination IP addresses?

Which transport protocol is being used?

What are the source and destination ports?

Which service is normally associated with that destination port?

Does the observed traffic actually behave like that protocol?

For TCP, was a connection successfully established?

Are there repeated SYN packets or incomplete handshakes?

Are there unusual resets or retransmissions?

For UDP, did the destination respond?

Was an ICMP error returned?

Is this communication expected between these systems?

Is the frequency or volume normal?

Does endpoint, firewall, DNS, application, or authentication telemetry
support the same interpretation?
```

No single question proves that something is malicious. The value comes from combining multiple observations until we have enough evidence to explain what happened.

## Try It in an Isolated Lab

Only scan systems and networks that you own or have explicit authorization to test.

On a Linux system, begin by examining the TCP and UDP sockets currently listening for traffic:

```bash
ss -tuln
```

Here, `-t` displays TCP sockets, `-u` displays UDP sockets, `-l` limits the output to listening sockets, and `-n` displays numerical addresses and ports instead of resolving names.

Rather than simply looking for port numbers, pay attention to whether each endpoint is TCP or UDP.

Next, perform a normal Nmap scan against an authorized target:

```bash
nmap <target-ip>
```

If you want to examine a particular TCP port:

```bash
nmap -p <port> <target-ip>
```

Where appropriate and with sufficient privileges, compare this with a SYN scan:

```bash
sudo nmap -sS -p <port> <target-ip>
```

Then choose a UDP service that actually exists in the lab and scan its relevant port:

```bash
sudo nmap -sU -p <port> <target-ip>
```

The useful part of this exercise is not simply writing down whether Nmap reports `open`.

For the TCP scan, ask what response allowed Nmap to classify the port. Did the target return SYN-ACK, RST, nothing, or some other response?

For the UDP scan, ask whether the service responded and whether Nmap has enough evidence to distinguish between `open`, `closed`, `filtered`, and `open|filtered`.

If Wireshark is available, capturing the traffic makes the distinction much easier to understand. The TCP handshake can be observed directly, while UDP communication can be compared against it without an equivalent transport-layer handshake.

## What This Gives Us in Practice

Understanding TCP and UDP means being able to reason about their behavior rather than only repeating definitions.

For TCP, that includes understanding why the three-way handshake exists, what SYN and ACK represent, why sequence and acknowledgment numbers are needed, how missing data leads to retransmission, why data can be delivered in order, and how flow control differs from congestion control.

For UDP, it means understanding what connectionless transport actually means, what guarantees UDP does and does not provide, why applications still choose it, and how protocols operating above UDP can implement capabilities that UDP itself does not provide.

From a security perspective, it means understanding why TCP ports generally provide clearer responses during scanning, why UDP scanning can produce states such as `open|filtered`, how ICMP errors help identify closed UDP ports, why TCP and UDP versions of the same port number are separate endpoints, and how transport behavior influences firewall and monitoring decisions.

Most importantly, it gives us a way to separate **what the packets actually prove** from **what we are inferring based on conventions such as common port numbers**.

That habit becomes increasingly important as network analysis moves from simple scans into actual investigations.

## From Transport Behavior to Actual Packets

TCP and UDP become much easier to understand once we can watch them moving across a network.

A TCP handshake is not just a diagram. SYN, SYN-ACK, and ACK appear as actual packets. Sequence and acknowledgment numbers can be inspected. Retransmissions are visible. Flags can be filtered and compared. Connection establishment and termination can be followed packet by packet.

UDP traffic can also be opened and examined. DNS queries and responses can be inspected field by field. ICMP errors can be traced back to UDP probes. We can observe which system initiated communication, which ports were involved, whether a response came back, and exactly what information moved across the network.

The next step is learning how to see that evidence directly:

**Wireshark From Zero: Capturing and Reading Your First Packets**
