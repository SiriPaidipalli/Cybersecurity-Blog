# DNS From a Security Perspective

Opening a website, connecting to a server by hostname, sending an email, downloading a package, or accessing a cloud service often begins with the same problem: the application knows a name, but the network eventually needs information that can be used to reach the destination.

DNS, the Domain Name System, is the infrastructure that helps bridge that gap. It is commonly introduced as the system that translates domain names into IP addresses. That description is correct, but it leaves out most of what DNS actually does. DNS is a distributed database that stores different kinds of information about domains, divides responsibility across a hierarchy of servers, caches answers for efficiency, and helps systems locate websites, mail servers, services, and other resources.

That makes DNS important to security as well. Malicious infrastructure depends on DNS just as legitimate infrastructure does. Attackers can manipulate DNS responses, abuse exposed resolvers, create deceptive domains, hide communication inside DNS traffic, or change where a legitimate name resolves. At the same time, DNS records and query activity can reveal valuable information about infrastructure and system behavior.

This article is part of my cybersecurity series, which builds from core concepts toward practical security analysis and investigation. Each article stands on its own, so we will build DNS from its basic structure through resolution, records, troubleshooting, and the security problems that appear around it.

---

## DNS Is More Than Name-to-IP Translation

Suppose an application wants to reach:

```text
www.example.com
```

The name is convenient for people and applications, but the network connection eventually needs information about where that destination can be reached. DNS can provide an IP address associated with the name, but addresses are only one type of information DNS stores.

A domain can contain records describing IPv4 and IPv6 addresses, aliases, mail servers, authoritative name servers, verification information, service locations, certificate policies, and other data. It is therefore more accurate to think of DNS as a distributed directory containing structured records.

Conceptually, the same domain might contain information such as:

```text
example.com        -> IPv4 address
example.com        -> IPv6 address
www.example.com    -> another hostname
example.com        -> mail server
example.com        -> authoritative name servers
```

Applications and administrators ask DNS specific questions by requesting particular record types. Understanding DNS therefore requires more than knowing that names become addresses. You need to understand where DNS information is stored, how a resolver finds it, what different records mean, and how long those answers remain usable.

---

## The DNS Namespace and Hierarchy

DNS names are organized hierarchically. Consider:

```text
www.example.com
```

Reading the hierarchy from right to left gives us:

```text
.
└── com
    └── example
        └── www
```

The invisible dot at the far right represents the **DNS root**. Beneath the root is a **top-level domain**, or TLD, such as `.com`, `.org`, or `.net`. Beneath `.com` is `example`, and beneath that is `www`.

A fully qualified name can technically be written as:

```text
www.example.com.
```

The final dot explicitly represents the root, although it is normally omitted in everyday use.

Terms such as **domain**, **subdomain**, and **hostname** are sometimes used loosely. `example.com` can represent a domain, while `www.example.com` is a name beneath it. A hostname identifies a host or service in a particular context, although modern DNS infrastructure means that a hostname does not necessarily correspond to one physical machine.

This hierarchy exists because no single DNS server contains every record on the Internet. Responsibility is distributed.

```text
                    DNS Root
                       |
           ┌───────────┼───────────┐
           |           |           |
         .com        .org        .net
           |
       example.com
           |
       ┌───┴────┐
       |        |
      www      mail
```

Root servers help resolvers locate the appropriate TLD infrastructure. TLD servers can direct resolvers toward the authoritative servers responsible for particular domains. Those authoritative servers provide DNS information for the zones they serve.

This delegation model lets different organizations manage different portions of DNS without requiring one central system to maintain every hostname and record on the Internet.

---

## How a DNS Lookup Actually Happens

Several components can participate in resolving a name, and each has a different responsibility.

The **stub resolver** exists on the client side and provides name-resolution functionality to applications. Instead of walking through the entire DNS hierarchy itself, it normally sends a request to a configured **recursive resolver**.

The recursive resolver performs resolution on behalf of the client. That resolver might belong to an ISP, organization, cloud provider, VPN service, or public DNS provider. If it does not already have a usable cached answer, it can interact with root, TLD, and authoritative DNS servers to obtain one.

The basic relationship looks like this:

```text
Application
    |
    v
Stub Resolver
    |
    v
Recursive Resolver
    |
    +----> Root Server
    |
    +----> TLD Server
    |
    +----> Authoritative Server
    |
    v
Answer Returned to Client
```

Suppose the client needs the address associated with `www.example.com`. The client asks its recursive resolver for the answer. If the resolver does not already know it, the resolver can ask the root infrastructure where to find information about `.com`.

The root server does not normally return the final IP address. Instead, it points the resolver toward the appropriate `.com` name servers. The resolver then asks the TLD infrastructure, which can point it toward the authoritative servers for `example.com`. Finally, the resolver asks an authoritative server for the requested record and returns the resulting information to the client.

```text
Client
  |
  v
Recursive Resolver
  |
  | Ask Root
  v
Root
  |
  | "Ask .com"
  v
Recursive Resolver
  |
  | Ask .com
  v
TLD Server
  |
  | "Ask example.com's servers"
  v
Recursive Resolver
  |
  | Ask Authoritative Server
  v
Authoritative Server
  |
  | Answer
  v
Recursive Resolver
  |
  v
Client
```

This also explains the terms **recursive** and **iterative** resolution. The client generally asks the recursive resolver to obtain the final result on its behalf. During that process, other DNS servers may provide referrals that tell the resolver where it should ask next rather than resolving the entire request themselves.

Not every lookup follows the complete path every time. DNS caching can eliminate several of these steps.

---

## Zones, Delegation, and Authority

A **DNS zone** is an administrative portion of the DNS namespace managed as a unit. A zone is related to a domain, but the two terms are not always interchangeable because portions of a domain can be delegated elsewhere.

Suppose an organization controls:

```text
example.com
```

It may manage:

```text
www.example.com
mail.example.com
vpn.example.com
```

inside the same zone. The organization could then delegate:

```text
research.example.com
```

to another set of authoritative name servers.

The parent zone contains information directing resolvers toward the servers responsible for the delegated child zone. The child zone then maintains its own DNS records.

This creates an administrative boundary inside the DNS namespace. It also becomes important when troubleshooting because an incorrect record inside a zone and an incorrect delegation to that zone are two different failures.

---

## DNS Records You Need to Understand

DNS stores information in **resource records**, commonly called DNS records or RRs. A record contains fields such as a name, TTL, class, record type, and record-specific data.

For example:

```text
www.example.com.    3600    IN    A    192.0.2.10
```

can be read as:

```text
www.example.com.    -> name
3600                -> TTL
IN                  -> Internet class
A                   -> record type
192.0.2.10          -> record data
```

The `IN` class is overwhelmingly common for Internet DNS. The record type is particularly important because it tells us what the data represents.

### A and AAAA Records

An **A record** associates a name with an IPv4 address.

```text
www.example.com    A    192.0.2.10
```

You can request it using:

```bash
dig example.com A
```

An **AAAA record** serves the equivalent purpose for IPv6:

```text
www.example.com    AAAA    2001:db8::10
```

and can be requested with:

```bash
dig example.com AAAA
```

A domain can have both record types and can also return multiple addresses. Modern infrastructure frequently uses load balancing, cloud platforms, content-delivery networks, and geographically distributed systems, so one hostname should not automatically be imagined as one permanent physical server.

### CNAME Records

A **CNAME**, or Canonical Name record, makes one DNS name an alias of another DNS name.

```text
www.example.com    CNAME    web.example.net
```

The resolver must then resolve the target name:

```text
www.example.com
       |
       | CNAME
       v
web.example.net
       |
       | A
       v
   192.0.2.10
```

CNAME records are common with cloud platforms, hosted services, and content-delivery networks.

A CNAME is different from an HTTP redirect. DNS resolution happens before the HTTP exchange and changes how a name is resolved. An HTTP redirect is a response from a web application telling the client to request another URL.

### MX Records

An **MX**, or Mail Exchange record, identifies mail servers responsible for receiving email for a domain.

```bash
dig example.com MX
```

A result might contain:

```text
example.com.    MX    10 mail1.example.com.
example.com.    MX    20 mail2.example.com.
```

The numbers are preference values, with lower values generally receiving higher priority. The MX record contains a hostname, so the mail server's address must then be resolved through DNS as well.

### NS and SOA Records

An **NS record** identifies authoritative name servers associated with a DNS zone.

```bash
dig example.com NS
```

For example:

```text
ns1.example.net.
ns2.example.net.
```

These records are central to delegation and can also reveal where an organization's authoritative DNS is hosted.

Every DNS zone also contains an **SOA**, or Start of Authority record.

```bash
dig example.com SOA
```

The SOA contains operational information including the primary name server, responsible-party information, a serial number, and several timing values. The serial number acts as a version of the zone data and helps secondary authoritative servers determine whether newer information is available.

### TXT Records

A **TXT record** stores text associated with a DNS name.

```bash
dig example.com TXT
```

TXT records are widely used for domain verification and policies related to technologies such as SPF, DKIM, and DMARC. They can also contain verification values for cloud services and other platforms.

Because of this, TXT records can reveal services and security mechanisms associated with a domain. Their presence, however, only shows what has been published in DNS. It does not prove that every referenced system is active or securely configured.

### PTR and Reverse DNS

Normal DNS resolution starts with a name. **Reverse DNS** starts with an IP address and attempts to find an associated hostname.

IPv4 reverse DNS uses:

```text
in-addr.arpa
```

while IPv6 uses:

```text
ip6.arpa
```

The relevant record type is **PTR**.

A reverse lookup can be performed with:

```bash
dig -x 192.0.2.10
```

or:

```bash
host 192.0.2.10
```

Reverse DNS can provide useful information about infrastructure and mail systems, but the returned hostname should not be treated as definitive proof of ownership. Forward and reverse DNS are administered separately and can be incomplete or outdated.

### Other Records Worth Recognizing

**SRV records** describe the location of services and can contain information such as a target hostname, port, priority, and weight. They are especially important in systems that use DNS for service discovery.

**CAA records** allow domain owners to specify certificate authorities that are permitted to issue certificates for the domain.

You may also encounter **DNSKEY**, **DS**, and **RRSIG** records. These belong to DNSSEC and will make more sense once we examine how DNSSEC establishes trust.

---

## Caching, TTLs, and Why DNS Changes Take Time

Repeating the complete DNS resolution process for every connection would create unnecessary latency and load. DNS therefore relies heavily on caching.

Records contain a **Time to Live**, or TTL:

```text
example.com.    3600    IN    A    192.0.2.10
```

A TTL of `3600` means that the information can normally be cached for 3,600 seconds, or one hour.

The first request might require a resolver to obtain the information through DNS infrastructure:

```text
Client -> Resolver -> DNS Hierarchy -> Answer
```

Later requests can potentially be answered from cache:

```text
Client -> Resolver Cache -> Answer
```

This explains why changing a DNS record does not guarantee that everyone immediately sees the new value. A resolver that still has the old record cached can continue returning it until the cached information expires.

DNS can also cache failures. **Negative caching** allows information such as an authoritative response indicating that a name does not exist to be retained temporarily. If an administrator creates that previously missing record a few minutes later, some clients may continue receiving the cached negative result until it expires.

Caching can occur at several places depending on the operating system and application. Applications, local resolver components, caching services, and recursive resolvers can all retain DNS-related information. This is why clearing one cache does not necessarily guarantee that every other system will immediately obtain fresh authoritative data.

Caching also affects visibility. An application can make several connections using an address obtained from an earlier cached lookup without generating a new DNS query for every connection. Conversely, an application can resolve a domain and never connect to the returned address. DNS events and later network connections therefore do not necessarily have a one-to-one relationship.

---

## DNS Transport: UDP, TCP, and EDNS

Traditional DNS commonly uses port `53` and can operate over both UDP and TCP.

Many traditional queries use UDP because DNS requests and responses are often small. TCP is also a normal part of DNS and can be used when required. One familiar case occurs when a response cannot be returned appropriately through the original UDP exchange and the client retries using TCP. Traditional zone transfers such as AXFR also use TCP.

Modern DNS also uses **EDNS**, or Extension Mechanisms for DNS. EDNS extends the capabilities of DNS without replacing the protocol and allows systems to communicate information such as support for larger UDP DNS messages than the original 512-byte limit.

When using `dig`, you may see:

```text
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 1232
```

The OPT entry is not an ordinary DNS record stored in the zone. It represents information associated with EDNS capabilities in the DNS exchange.

The practical DNS-specific point is that normal DNS should not be reduced to "UDP port 53." Both UDP and TCP participate in DNS, and newer mechanisms can change how DNS traffic appears on a network.

---

## Reading DNS With `dig`

`dig` is especially useful because it exposes the structure of a DNS response rather than displaying only the final address.

A basic query is:

```bash
dig example.com
```

A simplified result may contain:

```text
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 12345
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;example.com.            IN      A

;; ANSWER SECTION:
example.com.     3600    IN      A       192.0.2.10
```

The **QUESTION SECTION** tells you what was requested. The **ANSWER SECTION** contains records answering that question, including their TTL, class, type, and data.

The header contains useful flags as well. `qr` indicates a response, `rd` represents **Recursion Desired**, and `ra` represents **Recursion Available**. An `aa` flag means **Authoritative Answer**, while the DNSSEC-related `ad` flag can indicate **Authenticated Data** when a validating resolver has successfully validated DNSSEC information according to its trust configuration.

The response status is equally important. Common values include:

```text
NOERROR
NXDOMAIN
SERVFAIL
REFUSED
```

`NOERROR` means the request was processed without a DNS-level error. It does not guarantee that the requested record exists.

`NXDOMAIN` indicates that the queried name does not exist according to the response. `SERVFAIL` means the server could not successfully complete the request, which can result from upstream resolution problems, DNSSEC validation failures, or other conditions. `REFUSED` indicates that the server declined to perform the requested operation.

An important distinction exists between a nonexistent name and a name that exists but does not contain the requested record type.

```text
Name does not exist
        ->
     NXDOMAIN
```

is different from:

```text
Name exists
but requested record does not
        ->
     NOERROR
with no matching answer
```

That distinction can make the difference between troubleshooting the wrong hostname and troubleshooting a missing record.

---

## Choosing Where to Send the DNS Query

By default, `dig` generally uses the resolver configured for the local system. You can choose a specific resolver by placing `@` before its address:

```bash
dig @8.8.8.8 example.com
```

Comparing different resolvers can help determine whether an unexpected answer is specific to one resolver or is being returned more broadly.

Internal environments make this particularly relevant. An organization's DNS server may know about private names that public resolvers cannot resolve, while public infrastructure may contain completely different information for externally accessible services.

You can also ask an authoritative server directly. First identify the authoritative name servers:

```bash
dig example.com NS
```

Then query one of them:

```bash
dig @ns1.example.net example.com A
```

If the authoritative server returns a new address while your normal recursive resolver still returns an older one, caching becomes a likely explanation.

To expose the delegation path itself, `dig` provides:

```bash
dig +trace example.com
```

This approximates the path from the DNS root toward the authoritative infrastructure:

```text
       Root
        |
        v
       TLD
        |
        v
Authoritative Servers
        |
        v
 Final DNS Information
```

If you only need the returned record data, you can use:

```bash
dig +short example.com
```

That is useful for scripts and quick checks, but it removes details such as flags, TTLs, response codes, and authority information. Full `dig` output is therefore more useful when the reason behind an answer matters.

Tools such as `host` and `nslookup` can also perform DNS queries. They are convenient for quick lookups, while `dig` generally provides more visibility into the DNS message itself.

---

## DNS Resolution on the Local Machine

DNS is not necessarily the only mechanism involved when an application resolves a hostname.

Linux systems traditionally represent resolver configuration through:

```text
/etc/resolv.conf
```

You might see:

```text
nameserver 192.168.1.1
nameserver 1.1.1.1
search example.local
```

The `nameserver` entries identify resolvers the system can query. A `search` entry can influence how incomplete names are expanded.

Modern Linux systems may manage this file through NetworkManager, `systemd-resolved`, DHCP software, VPN clients, or other components. Editing `/etc/resolv.conf` manually can therefore be temporary or inappropriate depending on the system.

Search domains can also affect what actually gets queried. If a system has:

```text
corp.example.com
```

as a search domain and an application tries to resolve:

```text
server1
```

the resolver may attempt:

```text
server1.corp.example.com
```

This can explain DNS queries that appear different from the exact name a user typed.

The local hosts file adds another layer:

```text
/etc/hosts
```

For example:

```text
127.0.0.1       localhost
192.168.56.20   labserver
```

Depending on the operating system's name-service configuration, the hosts file may be checked before DNS. On Linux, that behavior is commonly influenced by:

```text
/etc/nsswitch.conf
```

An entry such as:

```text
hosts: files dns
```

indicates that local files are considered before DNS.

A local hosts-file entry can therefore make a hostname resolve even when no DNS record exists, or cause one machine to resolve a name differently from other systems.

---

## Zone Transfers and Recursive DNS Exposure

Authoritative DNS servers sometimes need to replicate zone information. Traditional DNS supports **AXFR** for full zone transfers and **IXFR** for incremental transfers.

A request can be made using:

```bash
dig AXFR example.com @authoritative-server
```

against systems you own or are explicitly authorized to test.

Zone transfers are legitimate DNS functionality. The security problem appears when a server allows zone information to be transferred to systems that should not receive it. An exposed transfer can reveal hostnames, service naming patterns, mail infrastructure, and other information contained in the zone.

Recursive resolvers have a different exposure problem. A resolver intended for internal clients should normally limit who can use its recursive service. A resolver that accepts recursive requests from arbitrary Internet clients can become an **open resolver** and may be abused in DNS-based attacks.

This distinction reinforces why authoritative and recursive DNS roles should not be confused. Public authoritative servers may need to answer questions about the zones they host. That does not mean unrestricted recursive resolution should also be available to everyone.

---

## How DNS Can Be Manipulated

Traditional DNS was designed in an environment very different from today's Internet and did not originally provide cryptographic validation of ordinary DNS responses. Several attacks take advantage of where DNS information is trusted or how it is delivered.

In **DNS cache poisoning**, an attacker attempts to cause a recursive resolver to cache false DNS information. If successful, later clients relying on that resolver may receive the malicious answer until it expires or is removed.

```text
Client asks for:
bank.example
      |
      v
Recursive Resolver
      |
      | poisoned cached answer
      v
Attacker-controlled address
```

Modern DNS implementations use protections such as unpredictable transaction identifiers and source ports to make forged responses more difficult.

The term **DNS spoofing** is broader and can describe situations where a victim receives a forged or manipulated DNS response. An attacker who can observe or influence DNS traffic might attempt to provide a false response before the legitimate one arrives.

These attacks differ from compromising authoritative DNS itself. If an attacker obtains control of DNS hosting credentials and changes the actual records, the resolver may simply be returning the information that the authoritative infrastructure now legitimately publishes.

**DNS hijacking** is another broad term describing situations where DNS configuration or resolution is altered to redirect users. That can happen through a compromised registrar account, DNS hosting account, local resolver configuration, router configuration, or interference elsewhere in the resolution path.

Although the symptoms can look similar, the affected component determines what actually happened. A poisoned recursive cache, changed authoritative record, modified hosts file, and compromised registrar delegation all influence DNS at different points.

---

## DNSSEC: Adding Authenticity and Integrity

**DNSSEC**, or Domain Name System Security Extensions, adds cryptographic signatures that allow validating resolvers to verify DNS data.

DNSSEC does not encrypt DNS queries or responses. Its purpose is to provide authenticity and integrity for DNS data.

Several record types participate in this process.

A **DNSKEY** record publishes public-key information associated with a signed zone. An **RRSIG** record contains a cryptographic signature over a set of DNS records. A **DS**, or Delegation Signer record, appears in a parent zone and helps connect trust between a parent and a signed child zone.

Conceptually, validation follows a chain:

```text
Trusted Root
     |
     v
 Signed TLD
     |
     v
Signed Domain
     |
     v
Validated DNS Data
```

A validating resolver can follow that chain toward a configured trust anchor and verify the signatures associated with the DNS data.

DNSSEC protects a specific part of the DNS problem. If the legitimate owner intentionally publishes:

```text
example.com -> 192.0.2.10
```

DNSSEC can help establish that this is authentic signed DNS data. It cannot determine whether the application running at that address is secure.

Likewise, if an attacker compromises the infrastructure responsible for legitimately managing or signing the zone, DNSSEC cannot repair that administrative compromise. Its purpose is to validate DNS data, not judge the safety of the service behind it.

---

## DNS Privacy: DoH and DoT

Traditional DNS traffic can reveal the names a system attempts to resolve to parties able to observe the relevant network path. **DNS over HTTPS**, or DoH, and **DNS over TLS**, or DoT, address this confidentiality problem between the client and its resolver.

DoT carries DNS through TLS and conventionally uses TCP port `853`.

DoH carries DNS through HTTPS, commonly over port `443`.

```text
Traditional DNS
Client -------- DNS Resolver

DoT
Client ===== TLS ===== DNS Resolver

DoH
Client ==== HTTPS ==== DNS Resolver
```

These technologies protect DNS communication between the client and resolver from ordinary observation or modification along that path. The resolver itself still receives and processes the queries, so encrypted DNS does not make DNS resolution anonymous.

Encrypted DNS also changes network visibility. A monitoring system that previously inspected traditional DNS queries may no longer be able to see the requested domain names when a client sends them through encrypted DNS to an external resolver.

DNSSEC and encrypted DNS therefore solve different problems:

```text
DNSSEC
    -> Authenticity and integrity of DNS data

DoH / DoT
    -> Confidentiality and integrity of communication
       between client and resolver
```

A deployment can use one, both, or neither. Calling all three technologies "secure DNS" without distinguishing their purposes hides an important difference in what each mechanism protects.

---

## DNS as an Attack Channel

DNS is useful to attackers not only because it can redirect traffic, but because DNS itself is widely available and deeply integrated into networks.

One example is **DNS reflection and amplification**. An attacker can send DNS requests with the victim's IP address forged as the source. DNS servers then send their responses toward the victim.

```text
          forged source = victim

Attacker --------------------> DNS Servers
                                  |
                                  |
                                  v
                                Victim
```

If the response is significantly larger than the request, amplification occurs. Repeating this through many DNS servers can direct a large volume of traffic toward the victim. Open resolvers and unnecessarily exposed DNS services can contribute to this kind of abuse.

Another technique is **DNS tunneling**. Because DNS traffic is often permitted through networks, data can be encoded into DNS names or responses.

Instead of a normal request such as:

```text
api.example.com
```

a compromised system could generate names resembling:

```text
aGVsbG8x.attackerdomain.example
aGVsbG8y.attackerdomain.example
aGVsbG8z.attackerdomain.example
```

An authoritative server controlled by an attacker can receive the queries and extract the encoded information. DNS responses can also carry information in the opposite direction, allowing DNS to be abused for command-and-control communication or data transfer.

Long names alone do not identify tunneling. Cloud applications, tracking systems, CDNs, and security products can generate complicated DNS names too. Analysis can instead consider combinations of properties such as unusually long labels, high query volume, repeated requests beneath one parent domain, high-entropy subdomains, unusual record types, and abnormal response behavior.

---

## Malicious Domains and Changing Infrastructure

Some malware uses **Domain Generation Algorithms**, or DGAs, to generate large numbers of candidate domain names instead of relying on one fixed command-and-control domain.

The generated names might resemble:

```text
xjskqpa.example
qmxpazr.example
vklqwer.example
```

An attacker only needs to register a subset of the generated domains. Infected systems produce the same candidates and attempt to resolve them until one succeeds.

This can create patterns involving many failed lookups followed by occasional successful resolutions. High NXDOMAIN rates, repeated algorithmic-looking names, registration age, and similar behavior across multiple hosts can help identify activity worth examining.

Malicious infrastructure can also use **fast-flux DNS**, where a domain rapidly rotates through multiple IP addresses, sometimes with short TTLs:

```text
malicious.example
       |
       +----> IP A
       +----> IP B
       +----> IP C
       +----> IP D
```

Rapidly changing DNS answers are also common in legitimate CDNs, cloud services, and load-balanced applications. Distinguishing fast flux from normal distributed infrastructure requires examining characteristics such as address ownership, TTL behavior, geographic distribution, infrastructure relationships, and associated activity.

Attackers can also register **lookalike domains** using misspellings, additional words, alternative TLDs, or visually similar characters. A newly registered domain appearing unexpectedly in an authentication flow, email, or endpoint connection may deserve closer examination, but domain age by itself does not establish maliciousness.

DNS analysis can be expanded with registration information, certificate data, hosting relationships, historical DNS observations, and related subdomains to understand how the domain fits into a larger infrastructure picture.

---

## Subdomain Takeover and DNS Rebinding

DNS can also create security problems when applications or external services make assumptions about what a name represents.

A **subdomain takeover** can become possible when a DNS record continues pointing to an external service after the legitimate resource has been removed.

For example:

```text
blog.example.com
       |
       | CNAME
       v
old-site.hosting-provider.example
```

If the external provider allows someone else to claim the abandoned resource identifier, an attacker may be able to serve content through the organization's subdomain.

A dangling DNS record by itself does not prove that takeover is possible. The external resource must actually be claimable. This is why DNS cleanup is important when cloud services, SaaS applications, storage endpoints, and hosted resources are decommissioned.

**DNS rebinding** involves a different property of DNS: the same domain can resolve to different addresses over time. A domain controlled by an attacker can initially resolve one way and later return another destination, potentially including an internal address.

This becomes dangerous when an application treats the hostname itself as sufficient proof that later network requests remain safe. Defenses can involve application validation, browser behavior, DNS protections, and network restrictions around private or otherwise sensitive destinations.

---

## DNS in Phishing and Infrastructure Analysis

Phishing infrastructure often depends on DNS because victims need to reach malicious websites, mail systems, tracking endpoints, or related services.

When examining a suspicious domain, DNS can help answer questions such as:

```text
What addresses does the domain resolve to?
Which name servers host its DNS?
Does it use aliases?
Does it have mail infrastructure?
How has its DNS changed?
Does it share infrastructure with related domains?
```

Current DNS answers only show what is available now. **Passive DNS** systems retain historical observations of relationships between names and addresses.

For example:

```text
Monday      suspicious.example -> IP A
Tuesday     suspicious.example -> IP B
Wednesday   suspicious.example -> IP C
```

A current lookup may reveal only IP C. Historical DNS data may preserve the earlier relationships, which can be useful when malicious infrastructure changes before an investigation begins.

Passive DNS is observational rather than authoritative. Its coverage depends on where and how the provider collected DNS information, so absence from a passive DNS dataset does not necessarily mean that a relationship never existed.

---

## DNS Logs and Security Visibility

DNS logs can record which names systems attempted to resolve. Depending on the environment, these records may come from recursive resolvers, DNS servers, endpoint agents, cloud services, network sensors, or security products.

A DNS event may contain:

```text
Timestamp
Client address
Queried domain
Record type
Response code
Returned data
Resolver
```

That can help associate a system with a domain even when later application communication is encrypted.

Patterns that may deserve examination include:

```text
Large numbers of NXDOMAIN responses
Repeated queries to unusual domains
Very long or high-entropy subdomains
Unexpected TXT queries
Rapidly changing answers
Unusual query frequency
Suspicious naming patterns
Queries to newly observed domains
Traffic to unauthorized external resolvers
Unexpected DoH or DoT usage
```

None of these properties works as a universal malicious indicator. Legitimate cloud software can generate many domains, CDNs rotate addresses, security products may use generated subdomains, and applications can legitimately use TXT records.

DNS activity becomes much more informative when it can be connected with endpoint processes, network connections, authentication events, threat intelligence, or other activity occurring around the same time.

---

## DNS Inside Organizations

Public DNS is only part of the picture. Organizations frequently operate internal DNS records and private namespaces that are available only through approved networks or resolvers.

Examples might include:

```text
fileserver.corp.example
database.internal.example
dc01.corp.example
```

Organizations can also use **split-horizon DNS**, sometimes called split DNS, to intentionally return different answers depending on where a query originates.

An internal client might receive:

```text
portal.example.com -> 10.20.30.40
```

while a public resolver returns:

```text
portal.example.com -> 203.0.113.40
```

Both answers can be correct within their respective environments. This is why the resolver and network location of the client can matter when two systems receive different answers for the same hostname.

DNS also plays a major role in **Microsoft Active Directory**. Clients use DNS, particularly SRV records, to discover services such as domain controllers. DNS failures in an Active Directory environment can therefore appear as authentication, domain-join, Group Policy, or service-discovery problems even when the underlying service itself is functioning.

Cloud and container environments depend heavily on DNS as well. Cloud platforms provide public and private DNS services, while container orchestration systems use DNS for service discovery. Applications can communicate through logical service names rather than depending on fixed addresses that may change as workloads are created, removed, or replaced.

In these environments, DNS is not simply a convenience for translating website names. It becomes part of the architecture that allows application components to discover and communicate with one another.

---

## A Practical DNS Troubleshooting Process

When DNS does not behave as expected, start by defining exactly what information should exist. A failed A-record lookup, missing MX record, stale recursive answer, and broken delegation are different problems.

Begin with the resolver normally used by the system:

```bash
dig example.com
```

Read the requested record type, response code, answer section, TTL, and responding server.

If necessary, compare another resolver:

```bash
dig @resolver-address example.com A
```

Then identify the authoritative servers:

```bash
dig example.com NS
```

and query one directly:

```bash
dig @authoritative-server example.com A
```

If the authoritative result differs from the recursive result, caching or resolver behavior may explain the difference.

When the delegation itself is uncertain, follow it:

```bash
dig +trace example.com
```

On the client, also inspect local resolution behavior. `/etc/hosts`, resolver configuration, search domains, VPN software, caching components, and network-management software can all influence what the application ultimately receives.

A useful mental model is:

```text
Application
    |
    v
Local Resolution Configuration
    |
    v
Recursive Resolver
    |
    v
DNS Delegation
    |
    v
Authoritative Zone
    |
    v
Returned Record
```

If the result is wrong, each stage represents a different place to investigate. A hosts-file modification affects the client. A poisoned resolver cache affects systems using that resolver. Incorrect authoritative records affect the zone. A registrar compromise can alter which servers are authoritative for the domain.

Understanding DNS as this chain makes troubleshooting much more precise than treating every failure as simply "DNS is broken."

---

## A Compact DNS Reference

Once the concepts are connected, the common record types are useful to keep together.

| Record | Purpose |
|---|---|
| `A` | Maps a name to an IPv4 address |
| `AAAA` | Maps a name to an IPv6 address |
| `CNAME` | Creates an alias to another DNS name |
| `MX` | Identifies mail servers |
| `NS` | Identifies authoritative name servers |
| `SOA` | Contains zone authority and administrative information |
| `TXT` | Stores text used for policies, verification, and other purposes |
| `PTR` | Supports reverse DNS |
| `SRV` | Describes service locations |
| `CAA` | Specifies permitted certificate authorities |
| `DNSKEY` | Publishes DNSSEC public-key information |
| `DS` | Connects DNSSEC trust between parent and child zones |
| `RRSIG` | Contains DNSSEC signatures |

The most useful `dig` operations can also be summarized:

| Task | Command |
|---|---|
| Query IPv4 | `dig example.com A` |
| Query IPv6 | `dig example.com AAAA` |
| Query mail servers | `dig example.com MX` |
| Query name servers | `dig example.com NS` |
| Query TXT records | `dig example.com TXT` |
| Query SOA | `dig example.com SOA` |
| Reverse lookup | `dig -x 192.0.2.10` |
| Choose a resolver | `dig @resolver example.com` |
| Follow delegation | `dig +trace example.com` |
| Compact output | `dig +short example.com` |
| Query authoritative server | `dig @authoritative-server example.com A` |
| Request authorized zone transfer | `dig AXFR example.com @authoritative-server` |

These commands are much easier to interpret once you understand which part of DNS each one interacts with. Changing the record type changes the question. Using `@server` changes who receives the question. `+trace` exposes delegation, while `-x` changes the direction of the lookup.

---

## From Resolution to Traffic Control

DNS gives systems a way to discover where services can be reached, but successful name resolution does not guarantee that communication with the resulting destination will be allowed.

A machine may resolve a server correctly and still be unable to connect because traffic can be filtered according to addresses, ports, protocols, interfaces, connection state, direction, and other rules. Those decisions can be enforced on the endpoint itself or elsewhere along the network path.

Understanding that control is the next piece of the network-security picture.

**Next in the series:**

**Firewalls: What They Actually Do to Your Traffic**
