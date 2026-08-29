# Wireshark From Zero: Capturing and Reading Your First Packets

*Network traffic is constantly moving between our systems and everything they communicate with. Wireshark gives us a way to stop looking at that communication as an invisible process and inspect the packets themselves.*

Opening a browser, resolving a domain name, connecting to a remote server, downloading a file, or signing in to an application can generate many network exchanges in only a few seconds. Most of the time, the operating system and applications handle those exchanges quietly in the background.

From a security perspective, that traffic can contain valuable evidence. It can show which systems communicated, which protocols were involved, which ports were used, whether a TCP connection succeeded, whether DNS queries were made, whether traffic was encrypted, whether packets were retransmitted, and how a conversation developed over time.

**Wireshark** is a network protocol analyzer that captures and examines this traffic packet by packet.

Learning Wireshark is not about memorizing every field that appears on the screen. The useful skill is learning how to move from thousands of packets to a smaller set of relevant evidence and then explain what that evidence actually tells us.

## What Wireshark Actually Captures

When applications communicate across a network, data moves through several protocol layers before being transmitted.

A simplified communication stack might look like this:

```text
Application Data
      ↓
  TCP or UDP
      ↓
     IP
      ↓
Ethernet / Wi-Fi
      ↓
   Network
```

Each layer adds information needed for a different part of communication.

Suppose a browser connects to a web server using HTTPS. At different layers, we may encounter information related to:

```text
Ethernet
IP
TCP
TLS
HTTP
```

Wireshark captures network traffic visible to a network interface and decodes the protocol structures it recognizes.

Instead of showing only a long sequence of raw bytes, Wireshark interprets those bytes and presents fields such as:

```text
Source MAC address
Destination MAC address
Source IP address
Destination IP address
Source port
Destination port
TCP flags
Sequence numbers
DNS queries
Protocol fields
Packet lengths
Timestamps
```

The important distinction is that Wireshark does not magically know what happened inside every application or endpoint. It analyzes the network traffic available at the location where the capture was performed.

That location matters.

## The Capture Point Determines What You Can See

A packet capture represents traffic visible from a particular **observation point**.

If Wireshark runs on a laptop, it can normally capture traffic entering or leaving the selected network interface on that laptop. It does not automatically receive every packet moving through the entire local network.

Consider three systems:

```text
System A ─────── Switch ─────── System B
                   │
                   │
                System C
```

Running Wireshark on System C does not necessarily allow System C to see all unicast communication between System A and System B.

Modern switched Ethernet networks normally forward unicast frames only toward the switch port associated with the destination MAC address. This is different from older shared-network environments where traffic could be visible more broadly.

Security teams that need visibility into traffic elsewhere in a network may use mechanisms such as switch port mirroring, network TAPs, packet capture appliances, sensors, or captures performed directly on relevant endpoints.

This gives us one of the most important rules of packet analysis:

**A packet capture can only show traffic that was visible at its capture point.**

Not seeing something in a packet capture does not automatically prove that it never happened anywhere on the network.

## Packets, Frames, Segments, and Datagrams

The word **packet** is often used casually for almost any unit of network traffic, including in Wireshark discussions. Technically, different layers use different terminology.

At the data-link layer, Ethernet carries **frames**.

At the IP layer, we commonly refer to **IP packets**.

TCP carries **segments**.

UDP carries **datagrams**.

A single captured Ethernet frame may therefore contain an IP packet, which contains a TCP segment, which carries application data.

Conceptually:

```text
Ethernet Frame
└── IP Packet
    └── TCP Segment
        └── Application Data
```

For UDP:

```text
Ethernet Frame
└── IP Packet
    └── UDP Datagram
        └── Application Data
```

Wireshark lets us expand these layers individually, which is one reason it is so useful for understanding how protocols fit together.

## Choosing the Correct Network Interface

When Wireshark starts, it displays available network interfaces.

Depending on the system, these may include interfaces associated with:

```text
Wi-Fi
Ethernet
Loopback
VPN software
Virtual machines
Docker or other virtual networking
USB network adapters
```

The correct interface depends on where the traffic you want to observe is moving.

If a laptop is connected through Wi-Fi, the active Wi-Fi interface will usually be the relevant one for ordinary Internet traffic. If the system is using Ethernet, the Ethernet interface may be appropriate.

Virtual cybersecurity labs can introduce additional interfaces because hypervisors create virtual networks for guest machines.

Before capturing traffic, it is useful to ask:

> Which interface is actually carrying the communication I want to observe?

Choosing the wrong interface can produce an empty capture or a capture full of unrelated traffic.

A screenshot is useful here because the Wireshark interface selection screen establishes where the capture is being performed.

## Starting a Capture

Once the correct interface is selected, Wireshark begins recording packets visible to that interface.

Within seconds, the packet list may fill with traffic even if you are not actively doing anything.

That is normal.

Modern systems constantly generate background network activity. Applications may check for updates, browsers maintain connections, operating systems contact network services, devices perform discovery, DNS queries occur, and cloud applications synchronize data.

A packet capture can therefore become large very quickly.

The goal is not to read every packet from top to bottom.

The goal is to narrow the capture to the communication relevant to the question being investigated.

This simple exercise demonstrates something important: network activity exists continuously, and security analysis requires filtering relevant traffic from background noise.

## Understanding the Wireshark Packet List

The upper portion of Wireshark displays a packet list. Common columns include:

```text
No.
Time
Source
Destination
Protocol
Length
Info
```

**No.** identifies the packet within the capture.

**Time** indicates when the packet was observed relative to the capture or according to the selected time-display format.

**Source** identifies where the packet came from.

**Destination** identifies where it was sent.

**Protocol** shows Wireshark's interpretation of the protocol.

**Length** shows the captured frame length.

**Info** provides a short protocol-specific summary.

A row might conceptually look like:

```text
No.   Time      Source       Destination   Protocol   Info
42    2.481     10.0.0.15    10.0.0.20     TCP        53422 → 22 [SYN]
```

Even without opening the packet, this already tells us that `10.0.0.15` sent a TCP SYN from source port `53422` toward destination port `22` on `10.0.0.20`.

The packet list is useful for spotting patterns, but the real detail appears when we inspect an individual packet.

## The Three Main Wireshark Panes

Wireshark commonly presents captured traffic using three panes.

The **Packet List** shows one row per captured packet.

The **Packet Details** pane breaks the selected packet into protocol layers and fields.

The **Packet Bytes** pane displays the actual captured bytes, usually in hexadecimal alongside an ASCII representation where applicable.

A selected Ethernet packet may contain expandable sections such as:

```text
Frame
Ethernet II
Internet Protocol Version 4
Transmission Control Protocol
Transport Layer Security
```

or:

```text
Frame
Ethernet II
Internet Protocol Version 4
User Datagram Protocol
Domain Name System
```

These sections represent encapsulation in practice. Instead of learning protocol layers only as a diagram, we can inspect the actual headers belonging to each layer.

This is one of the most useful first screenshots for the article because it demonstrates how a single captured frame contains information from several protocols.

## Reading the Ethernet Layer

At the Ethernet layer, Wireshark may show fields such as:

```text
Destination MAC Address
Source MAC Address
EtherType
```

MAC addresses identify network interfaces at the local data-link layer.

The EtherType field indicates what protocol is carried inside the Ethernet frame. For example, a value may indicate that the frame contains IPv4 or IPv6.

An important security distinction appears here: MAC addresses and IP addresses solve different problems.

MAC addresses are primarily relevant to communication on the local network segment, while IP addresses provide logical addressing used for communication across IP networks.

When a packet travels through routers, the Layer 2 frame information can change from one network segment to another while the IP communication continues toward its destination.

## Reading the IP Layer

Expanding the IP section reveals information such as:

```text
Source IP Address
Destination IP Address
Time To Live
Protocol
Total Length
Flags
```

The source and destination IP addresses identify the communicating IP endpoints for that packet.

The **Time To Live**, or TTL, limits how many routing hops an IPv4 packet can traverse before being discarded. Each router forwarding the packet normally decreases the TTL.

The **Protocol** field identifies what is carried inside the IP packet. For example, IP protocol number `6` represents TCP and `17` represents UDP.

This is a useful reminder that TCP and UDP are encapsulated inside IP rather than replacing it.

IP tells us which network endpoints are communicating. TCP or UDP then helps identify communication between applications on those endpoints.

## Reading the TCP Layer

If the packet contains TCP, expanding the TCP section can reveal:

```text
Source Port
Destination Port
Sequence Number
Acknowledgment Number
Header Length
Flags
Window Size
Checksum
TCP Options
```

Suppose Wireshark shows:

```text
Source Port:       53422
Destination Port:  443
Flags:             SYN
```

This tells us that the source system is attempting to begin a TCP connection toward destination TCP port 443.

If the next packet contains:

```text
Source Port:       443
Destination Port:  53422
Flags:             SYN, ACK
```

and the following packet contains:

```text
Source Port:       53422
Destination Port:  443
Flags:             ACK
```

we have observed the TCP three-way handshake.

```text
Client                                               Server

SYN             ───────────────────────▶

                ◀───────────────────────             SYN-ACK

ACK             ───────────────────────▶
```

Instead of simply knowing that TCP uses a handshake, Wireshark lets us verify that the handshake actually occurred between particular endpoints.

## Filtering for a TCP Handshake

Large captures make it difficult to find individual connections manually. Wireshark **display filters** allow us to reduce what is shown without changing the packets stored in the capture.

To display TCP traffic:

```text
tcp
```

To display TCP SYN packets:

```text
tcp.flags.syn == 1
```

A SYN-ACK also has the SYN flag set, so if we specifically want initial SYN packets that do not also contain ACK:

```text
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

This can be useful when examining connection attempts or looking for systems initiating many TCP connections.

To focus on a particular TCP port:

```text
tcp.port == 443
```

To combine conditions:

```text
tcp.port == 443 && ip.addr == 10.0.0.20
```

The ability to combine filters is one of the most important Wireshark skills because real captures can contain thousands or millions of packets.

## Capture Filters and Display Filters Are Different

Wireshark has two different filtering concepts, and confusing them causes a lot of frustration.

A **capture filter** controls which packets Wireshark records in the first place.

A **display filter** controls which already-captured packets are currently shown on the screen.

Suppose a capture contains 10,000 packets.

Applying the display filter:

```text
dns
```

does not delete the other packets. It simply hides packets that do not match the filter.

If a capture filter had been configured to record only DNS traffic before the capture started, unrelated traffic would never have been recorded.

This distinction has practical consequences.

Display filters are flexible because we can change them repeatedly while investigating the same capture. Capture filters reduce the amount of data collected but can permanently exclude traffic that later turns out to be important.

For initial learning and many troubleshooting situations, capturing broadly and using display filters afterward is often easier. In high-volume environments, however, capture filters may be necessary to control capture size and performance.

Capture filters use **Berkeley Packet Filter**, or BPF-style, syntax, while Wireshark display filters use Wireshark's own display-filter language.

For example, a capture filter might look like:

```text
host 10.0.0.20
```

while a display filter for the same IP might look like:

```text
ip.addr == 10.0.0.20
```

They serve related purposes but use different syntax.

## Filtering by IP Address

One of the most common investigation tasks is isolating traffic involving a particular host.

To show any IPv4 traffic where an address appears as either source or destination:

```text
ip.addr == 10.0.0.20
```

To show packets specifically originating from that address:

```text
ip.src == 10.0.0.20
```

To show packets specifically sent to it:

```text
ip.dst == 10.0.0.20
```

Filters can also be combined:

```text
ip.src == 10.0.0.15 && ip.dst == 10.0.0.20
```

This shows traffic moving specifically from one system toward another.

Being precise about direction becomes important during investigations. Communication from A to B may have a very different meaning from communication from B to A.

## Filtering by Port

TCP and UDP port filters can help isolate particular services or conversations.

Examples include:

```text
tcp.port == 22
```

```text
tcp.port == 443
```

```text
udp.port == 53
```

If direction matters, we can specify source or destination ports:

```text
tcp.dstport == 22
```

or:

```text
udp.srcport == 53
```

A port number is still only evidence, not proof of the application using it. TCP port 22 is commonly associated with SSH, but another application can technically listen there.

Wireshark may identify higher-layer protocols using protocol structure and dissectors, but the same evidence-first rule still applies: do not infer more than the traffic supports.

## UDP in Wireshark

UDP looks different from TCP because there is no three-way handshake and no TCP connection state.

A UDP packet may contain fields such as:

```text
Source Port
Destination Port
Length
Checksum
```

A request-response exchange might appear as:

```text
Client                                                  Server

UDP Request        ───────────────────────▶

                   ◀───────────────────────           UDP Response
```

There are no TCP sequence numbers, SYN flags, or transport-layer acknowledgments.

If a UDP packet is sent to a closed destination port, an ICMP error may be returned:

```text
Client                                                   Server

UDP Datagram       ───────────────────────▶

                   ◀───────────────────────        ICMP Port Unreachable
```

Seeing this in Wireshark makes it easier to understand why UDP scanning can be more ambiguous than TCP scanning. If there is no UDP response and no ICMP error, silence alone may not tell us whether the port is open, filtered, or simply ignoring the probe.

## DNS Traffic

DNS is one of the easiest and most useful protocols to inspect when learning Wireshark because its request-response behavior is clear and many traditional DNS queries are not encrypted.

A DNS query might ask:

```text
What is the IP address for example.com?
```

and the resolver may return one or more records containing the answer.

To display DNS traffic:

```text
dns
```

A DNS packet can contain information such as:

```text
Transaction ID
Flags
Questions
Answer Resource Records
Authority Records
Additional Records
Query Name
Query Type
Response Code
```

Query types may include:

```text
A       IPv4 address
AAAA    IPv6 address
CNAME   Canonical name
MX      Mail exchange
NS      Name server
TXT     Text record
```

If a system repeatedly queries unusual domains, generates a very large number of DNS requests, or contacts domains that are unexpected for its normal behavior, DNS traffic can become valuable investigation evidence.

However, a DNS query alone does not prove that a connection to the returned IP address was made. It tells us that name resolution activity occurred. We need additional packets or telemetry to determine what happened afterward.

This is a good place for an actual Wireshark screenshot because the DNS fields are visually meaningful and easy to explain.

## DNS Is Not Always Visible in Plaintext

Traditional DNS over UDP or TCP port 53 is often visible to packet-analysis tools, but modern systems may use encrypted DNS mechanisms.

**DNS over HTTPS (DoH)** sends DNS queries through HTTPS.

**DNS over TLS (DoT)** encrypts DNS communication using TLS.

When encrypted DNS is used, a network capture may still reveal communication with a DNS provider, but the individual domain queries may not be visible in plaintext.

This is an important limitation. The absence of readable DNS packets does not necessarily mean the system performed no DNS resolution.

## HTTP Traffic

Traditional HTTP is especially useful for understanding packet analysis because the application data may be readable directly from the capture.

A client might send a request such as:

```http
GET /index.html HTTP/1.1
Host: example.com
User-Agent: ...
```

The server may respond with:

```http
HTTP/1.1 200 OK
Content-Type: text/html
...
```

To filter HTTP traffic:

```text
http
```

Wireshark can decode fields including:

```text
Request Method
Request URI
Host
User-Agent
Status Code
Content Type
Headers
```

From a security perspective, plaintext HTTP can expose sensitive information because anyone with appropriate visibility into the traffic may be able to inspect its contents.

This is one of the reasons HTTPS became the normal way of delivering web traffic.

## HTTPS and TLS

When HTTPS is used, HTTP communication is protected using TLS.

A packet capture may therefore show:

```text
TCP
TLS
Application Data
```

rather than readable HTTP requests and responses.

Wireshark can still reveal useful metadata, including:

```text
Source and destination IP addresses
Source and destination ports
Packet timing
Packet sizes
TCP behavior
TLS handshake information
TLS versions
Some certificate information
```

Depending on the TLS version and configuration, additional handshake metadata may also be visible.

What Wireshark usually cannot do from an ordinary passive capture is simply display the plaintext contents of properly encrypted HTTPS application traffic.

This distinction is fundamental to network analysis:

**Encryption can hide content without hiding the existence of communication.**

We may know that two systems communicated, when they communicated, how much data moved, and what transport behavior occurred even when we cannot read the application payload.

## The TLS Handshake

Before encrypted application data can be exchanged, TLS establishes the cryptographic parameters needed to protect the session.

The exact handshake depends on the TLS version, but a capture may contain messages associated with operations such as:

```text
Client Hello
Server Hello
Certificate
Encrypted handshake messages
Application Data
```

Modern TLS 1.3 encrypts more of the handshake than older TLS versions, so what remains visible depends on the protocol version and configuration.

The important point is not to memorize every TLS handshake message yet. It is to recognize that HTTPS involves multiple layers:

```text
 HTTP
  ↓
 TLS
  ↓
 TCP
  ↓
  IP
```

for traditional HTTP/1.1 and HTTP/2 over TLS/TCP.

HTTP/3 changes this architecture by using QUIC over UDP.

Wireshark helps make these relationships visible rather than leaving them as abstract protocol diagrams.

## Following a TCP Stream

Individual packets are useful, but applications communicate through conversations rather than isolated packets.

Wireshark can reconstruct TCP data belonging to the same stream.

A common workflow is:

```text
Right-click a TCP packet
→ Follow
→ TCP Stream
```

Wireshark then groups data belonging to that TCP conversation and displays it together.

For unencrypted protocols, this can make application exchanges much easier to understand. Instead of manually reading dozens of individual packets, we can inspect the reconstructed conversation.

For encrypted TLS traffic, following the stream may show encrypted bytes rather than readable application content unless the necessary session secrets are available and decryption is appropriately configured.

Do not assume that readable stream contents will always be available. Whether they are depends on the protocol and encryption.

## TCP Retransmissions

Wireshark can identify patterns that appear consistent with TCP retransmissions.

A useful display filter is:

```text
tcp.analysis.retransmission
```

Retransmissions can occur when expected data is not acknowledged and TCP sends it again.

Seeing a retransmission does not automatically indicate an attack. Ordinary causes include packet loss, congestion, unstable wireless connections, or network path problems.

A small number of retransmissions in a large capture may be completely normal. A large or unusual pattern can become worth investigating, particularly when it aligns with application failures or other network symptoms.

Wireshark's TCP analysis features are extremely useful, but they should not be treated as infallible truth. The tool is analyzing the packets available in the capture. Missing packets at the capture point, asymmetric visibility, packet reordering, and capture loss can affect its interpretation.

## TCP Resets

TCP reset packets can be filtered using:

```text
tcp.flags.reset == 1
```

A reset may appear when a connection is rejected, unexpectedly terminated, sent to a closed port, or disrupted by an application or network device.

One reset packet is not automatically suspicious.

Repeated resets involving a particular service, unusual patterns across many hosts, or resets occurring during application failures may deserve further investigation.

As with retransmissions, context determines whether the pattern is meaningful.

## Finding Connection Attempts

To identify initial TCP SYN packets:

```text
tcp.flags.syn == 1 && tcp.flags.ack == 0
```

This can be useful when looking for systems initiating TCP connections.

Imagine a source generates SYN packets toward many destination ports:

```text
10.0.0.15 → 10.0.0.20:21
10.0.0.15 → 10.0.0.20:22
10.0.0.15 → 10.0.0.20:23
10.0.0.15 → 10.0.0.20:25
10.0.0.15 → 10.0.0.20:80
10.0.0.15 → 10.0.0.20:443
```

That pattern may be consistent with port scanning or service discovery.

It still does not automatically prove malicious activity. Vulnerability scanners, asset-discovery systems, monitoring platforms, administrators, and authorized security testing can generate similar patterns.

Wireshark gives us the network evidence. Determining intent requires context.

## ICMP Traffic

ICMP is another protocol worth recognizing because it carries network control and error information.

To display ICMP traffic:

```text
icmp
```

Common ICMP messages include:

```text
Echo Request
Echo Reply
Destination Unreachable
Time Exceeded
```

`ping`, for example, commonly uses ICMP Echo Request and Echo Reply messages.

ICMP Destination Unreachable messages can indicate problems reaching a network, host, protocol, or port. ICMP Time Exceeded messages are associated with situations where a packet's TTL reaches zero and are also fundamental to how tools such as traceroute can discover network paths.

ICMP is therefore not simply “ping traffic.” It provides information about network conditions and failures.

## ARP Traffic on the Local Network

Before an IPv4 host can send an Ethernet frame to another local system, it may need to determine which MAC address corresponds to a local IP address.

The **Address Resolution Protocol**, or ARP, performs this function.

To display ARP traffic:

```text
arp
```

A request may conceptually ask:

```text
Who has 192.168.1.20?
Tell 192.168.1.10
```

and the owner of that address may respond with its MAC address.

ARP operates locally and is not routed across the Internet.

From a security perspective, ARP is important because local-network attacks such as ARP spoofing or ARP poisoning attempt to manipulate the association between IP addresses and MAC addresses.

Normal ARP traffic is extremely common, so seeing ARP packets is not itself suspicious. What matters is whether the observed mappings and changes make sense for the environment.

## Protocol Identification Is More Than Port Numbers

Wireshark uses **protocol dissectors** to interpret network traffic.

A dissector understands the structure of a protocol and attempts to decode its fields into something meaningful.

This means Wireshark is not always relying solely on a port number when displaying protocol information.

That matters because services do not have to run on their conventional ports.

An HTTP service could technically listen on TCP port 8080, 8000, 8888, or another port. SSH could be configured on a nonstandard TCP port.

When Wireshark cannot automatically recognize traffic correctly, analysts can sometimes use features such as **Decode As** to tell Wireshark how a particular conversation should be interpreted.

Protocol identification is therefore an analytical process, not simply a lookup table of ports.

## Checksums and Apparent Errors

While inspecting packets, Wireshark may occasionally report checksum warnings.

It is tempting to assume that every checksum warning means a corrupted packet was transmitted across the network.

That is not always true.

Modern network interface cards and operating systems may use **checksum offloading**, where checksum calculation is performed by the network adapter later in the transmission process.

If Wireshark captures a packet before the hardware calculates the final checksum, the packet may appear to contain an incorrect checksum even though the transmitted packet is valid.

This is an excellent example of why packet-analysis tools need context.

A warning shown by Wireshark is evidence worth understanding, not automatically proof that something is broken or malicious.

## Packet Length and Timing

Packet content is not the only useful evidence.

Wireshark records when packets were captured and how large they were.

Timing can help answer questions such as:

```text
When did communication begin?

How long did the connection last?

Were requests repeated rapidly?

Was there a long delay before a response?

Did activity occur periodically?

Did many connections begin within a short window?
```

Packet lengths can also provide useful context. Even when payloads are encrypted, the size and timing of communication remain observable.

Traffic analysis therefore does not completely disappear when encryption is present. Encryption protects content, but network metadata can still reveal patterns.

## Conversations and Endpoints

Wireshark provides statistical views that help summarize large captures.

Under the **Statistics** menu, features such as **Endpoints** and **Conversations** can provide useful high-level views.

Endpoints help identify systems present in the capture.

Conversations group communication between pairs of endpoints.

These views can help answer questions such as:

```text
Which IP addresses communicated?

Which system exchanged the most traffic?

Which TCP or UDP conversations existed?

How many packets were exchanged?

How many bytes were transferred?
```

Instead of beginning with thousands of individual packets, an analyst can sometimes start with these summaries and then drill down into interesting conversations.

This is another screenshot worth including because it demonstrates how analysts move from a large capture toward specific conversations.

## Name Resolution Can Change What You See

Wireshark can resolve numerical addresses into names, which may make captures easier to read.

However, name resolution can also hide the raw values that are important during analysis.

For example, seeing a hostname is convenient, but the actual IP address may be what needs to be compared against a firewall event, threat-intelligence result, or endpoint log.

For careful analysis, it is often useful to remain aware of both the resolved name and the underlying numerical address.

Do not let a convenient display label replace the actual network evidence.

## Saving Packet Captures

Wireshark captures can be saved for later analysis.

The commonly used capture format is:

```text
.pcapng
```

and the older widely supported format is:

```text
.pcap
```

A saved packet capture can be reopened, filtered differently, shared with authorized analysts, or used as evidence during an investigation.

Packet captures can contain sensitive information. Depending on the environment and protocols involved, they may reveal:

```text
Internal IP addresses
Hostnames
Domain queries
Usernames
Application data
Authentication information
Session metadata
Network architecture
Potentially sensitive payload contents
```

Treat capture files as security-sensitive data.

Do not upload packet captures from real environments to public repositories without understanding exactly what they contain.

For public GitHub documentation, screenshots should also be reviewed before publishing to make sure they do not expose personal addresses, internal infrastructure, usernames, tokens, or other sensitive information.

## What Wireshark Cannot Tell You by Itself

Wireshark is powerful, but packet captures are only one source of evidence.

Suppose a capture shows:

```text
10.0.0.15 → 10.0.0.20:22
```

followed by a successful TCP handshake.

We can say that a TCP connection to port 22 was established.

We cannot conclude from that alone:

```text
Which human initiated the connection
Whether authentication succeeded
Which account was used
What commands were executed
Whether the activity was authorized
Whether the connection was malicious
```

Those questions may require:

```text
Authentication logs
Process telemetry
Endpoint logs
Firewall events
Application logs
EDR data
Identity records
User context
```

Similarly, if a packet is absent from a capture, we must consider whether the capture point would have been able to see it in the first place.

Good security analysis does not ask Wireshark to answer questions the available packets cannot answer.

## Wireshark and Encryption

Encryption changes what network analysis can observe, but it does not make packet analysis useless.

With properly encrypted traffic, the application payload may not be readable. We can still often observe metadata such as:

```text
Source and destination addresses
Transport protocol
Ports
Packet timing
Packet sizes
Connection duration
TCP flags
Retransmissions
TLS handshake information
Traffic volume
Communication patterns
```

This metadata can still be valuable.

For example, an endpoint unexpectedly making repeated encrypted connections to an unfamiliar external system may deserve investigation even when the payload cannot be decrypted.

The correct conclusion is not:

> “The traffic is encrypted, so there is nothing to analyze.”

It is:

> “The payload is protected, so we need to determine what useful metadata remains and combine it with other telemetry.”

## A Practical Packet Investigation Workflow

Opening a large `.pcap` file and scrolling randomly is not an effective investigation strategy.

A more structured workflow begins with a question.

For example:

> Which system initiated communication with this server, and what happened after the connection began?

From there, the process might look like:

```text
1. Understand where the capture came from
2. Check the capture time and scope
3. Identify relevant endpoints
4. Review Conversations and Endpoints
5. Filter by the relevant IP address
6. Identify TCP, UDP, ICMP, DNS, or other protocols involved
7. Narrow the traffic by port or conversation
8. Inspect connection establishment
9. Examine application-layer information where visible
10. Look for resets, retransmissions, errors, or unusual patterns
11. Build a timeline
12. Compare the network evidence with other available telemetry
13. Record what the evidence supports
14. Separate observations from assumptions
```

The exact steps change depending on the investigation, but the principle remains the same: start with a question, narrow the evidence, and build conclusions from what can actually be observed.

## A Small Example Investigation

Imagine a capture contains the following activity:

```text
10.0.0.15 → DNS Server       DNS Query: example.com
DNS Server → 10.0.0.15       DNS Response: 203.0.113.20

10.0.0.15 → 203.0.113.20:443 SYN
203.0.113.20 → 10.0.0.15     SYN-ACK
10.0.0.15 → 203.0.113.20     ACK

10.0.0.15 ↔ 203.0.113.20     TLS Traffic
```

From this evidence, we can build a reasonable sequence of events.

The system at `10.0.0.15` queried DNS for `example.com`. The resolver returned `203.0.113.20`. The system then established a TCP connection to port 443 on that IP address, followed by TLS-protected communication.

That is already much more informative than looking at one packet in isolation.

But we still should not claim more than the evidence supports. The capture does not necessarily tell us which user initiated the activity, whether a browser was responsible, what exact application content was exchanged, or whether the activity was malicious.

Packet analysis is about building the strongest conclusion supported by the available evidence without turning assumptions into facts.

## Useful Wireshark Display Filters

The following filters cover a large portion of the traffic you will encounter while learning Wireshark:

```text
tcp
udp
icmp
arp
dns
http
tls
```

For IP addresses:

```text
ip.addr == 10.0.0.20
ip.src == 10.0.0.20
ip.dst == 10.0.0.20
```

For ports:

```text
tcp.port == 443
tcp.dstport == 22
udp.port == 53
```

For TCP behavior:

```text
tcp.flags.syn == 1
tcp.flags.syn == 1 && tcp.flags.ack == 0
tcp.flags.reset == 1
tcp.analysis.retransmission
```

Filters can also be combined:

```text
ip.addr == 10.0.0.20 && tcp.port == 443
```

The goal is not to memorize every Wireshark filter. Wireshark supports a huge number of fields and protocol-specific filters.

What matters is learning how to construct a filter from the question you are trying to answer.

## Try It in Your Own Lab

A useful first Wireshark exercise can be completed without doing anything aggressive.

Use a system and network you own or are authorized to analyze.

Start Wireshark and select the interface carrying your lab or ordinary network traffic.

Begin capturing.

Generate a small amount of known activity, such as:

```bash
ping <authorized-target>
```

performing a DNS lookup:

```bash
nslookup example.com
```

or connecting to a service in your own lab.

Stop the capture shortly afterward.

Then investigate the traffic instead of simply confirming that packets exist.

For ICMP:

```text
icmp
```

Identify the Echo Request and Echo Reply and compare their source and destination addresses.

For DNS:

```text
dns
```

Identify the query name, query type, resolver, response, and returned record.

For TCP:

```text
tcp
```

Find a three-way handshake and identify:

```text
Client IP
Server IP
Client source port
Server destination port
SYN
SYN-ACK
ACK
```

Then choose one packet and expand:

```text
Ethernet
IP
TCP or UDP
Application Protocol
```

Finally, open:

```text
Statistics → Conversations
```

and compare the high-level conversation summary with the individual packets you inspected.

The value of these screenshots is not proving that Wireshark was opened. Each screenshot should support a specific observation you can explain.

## Reading Packets Instead of Watching Rows

The first time Wireshark opens, the amount of information can make packet analysis look far more complicated than it actually is.

The solution is not to understand every protocol at once.

Start with a question.

Find the relevant systems.

Determine whether the traffic is TCP, UDP, ICMP, DNS, HTTP, TLS, or something else.

Filter the capture.

Follow the conversation.

Inspect the fields that matter.

Then ask what the packets actually prove.

Once that process becomes familiar, Wireshark stops being a screen full of rapidly changing rows and becomes what it actually is: a way to observe network behavior directly.

That ability becomes especially useful when the communication involves web traffic, because HTTP and HTTPS show the difference between traffic we can read directly and traffic whose contents are protected by encryption.

The next topic is:

**HTTP & HTTPS: What Security Beginners Need to Understand**
