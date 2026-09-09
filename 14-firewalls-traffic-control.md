# Firewalls: What They Actually Do to Your Traffic

A system can have the correct destination address, a valid route, and a service ready to accept connections, yet communication can still fail. Somewhere along the path, a rule may decide that the traffic is not permitted to continue.

That is the basic job of a firewall.

Firewalls are often described as barriers between trusted and untrusted networks, but modern firewalls can exist almost anywhere. They can run directly on a laptop, protect a server, sit between internal network segments, control traffic entering or leaving a cloud workload, or inspect communication at the edge of an organization.

What makes something a firewall is not where it sits. It is the decision it makes about traffic.

At its simplest, a firewall evaluates network communication against a policy and decides whether that communication should be allowed, blocked, rejected, logged, translated, or processed in some other way. The information available for that decision depends on the firewall. Some work primarily with addresses, protocols, and ports. Others understand connection state, applications, users, URLs, and parts of application traffic.

The interesting part is not learning that firewalls "allow or block traffic." It is understanding exactly what information they use, where the decision happens, and why a connection can behave differently depending on the rules and state maintained by the firewall.

---

## A Firewall Is a Policy Enforcement Point

Consider a client trying to reach a web server.

```text
   Client
192.168.10.25
      |
      v
   Firewall
      |
      v
  Web Server
 10.20.30.40
```

When traffic reaches the firewall, it can compare information about that traffic with its configured policy.

A simplified rule might say:

```text
Source:       192.168.10.0/24
Destination:  10.20.30.40
Protocol:     TCP
Port:         443
Action:       Allow
```

Another might say:

```text
Source:       Any
Destination:  10.20.30.40
Protocol:     TCP
Port:         22
Action:       Deny
```

The firewall is not deciding whether TCP 22 is inherently dangerous or TCP 443 is inherently safe. It is enforcing the policy defined for that environment.

That distinction matters because the same traffic can be completely appropriate in one network and prohibited in another. SSH may be allowed only from an administrative network. A database may accept connections only from application servers. Employee workstations may be allowed to browse the Internet but prevented from initiating connections directly into a server-management network.

Firewall policy turns those communication requirements into enforceable rules.

---

## Where the Firewall Actually Sits

The word "firewall" often creates an image of one device between an internal network and the Internet.

```text
Internal Network
       |
       v
    Firewall
       |
       v
    Internet
```

That is one common placement, but it is only one possibility.

A firewall can run directly on an endpoint:

```text
  Network
     |
     v
Host Firewall
     |
     v
Operating System
     |
     v
 Application
```

Windows Defender Firewall and Linux firewall frameworks are examples of host-based controls. They can decide which traffic reaches or leaves that particular machine.

Organizations may also place firewalls between internal networks:

```text
 User Network
      |
      v
   Firewall
      |
      v
Server Network
```

Cloud environments can add several additional policy layers around a workload. A virtual machine might be affected by cloud security rules before traffic even reaches the firewall running inside its operating system.

A real path can therefore look more like:

```text
Client
   |
   v
Network Firewall
   |
   v
Cloud Network Policy
   |
   v
Instance Security Policy
   |
   v
Host Firewall
   |
   v
Application
```

This becomes important during troubleshooting. Saying "the firewall allows it" is not enough if several independent controls exist along the path. You need to know which control was checked and whether the traffic actually reached the next one.

---

## What a Firewall Can Examine

A basic firewall can make decisions using information already present in network and transport headers.

For example:

```text
Source IP:         192.168.10.25
Destination IP:    10.20.30.40
Protocol:          TCP
Source Port:       51842
Destination Port:  443
```

A firewall can compare those values with its rules and determine whether the packet matches a permitted communication path.

Rules can also consider the interface on which traffic arrived, the interface through which it would leave, and the direction in which communication is moving.

More advanced firewalls can evaluate additional information such as:

```text
Connection state
Application
User identity
URL category
TLS information
Protocol behavior
Threat signatures
```

The amount of information available depends on the firewall's capabilities and on whether the relevant traffic is visible to it.

This creates a spectrum. A simple packet filter may make a decision using a few header fields. A modern enterprise firewall may combine those fields with connection tracking, application identification, identity information, and security inspection before deciding what to do.

---

## Packet Filtering

The simplest firewall model is **packet filtering**.

A packet filter examines packets and compares their header information with configured rules. A rule might permit TCP traffic to a web server while denying traffic to an administrative service.

```text
ALLOW
Source:       Any
Destination:  10.20.30.40
Protocol:     TCP
Port:         443
```

and:

```text
DENY
Source:       Internet
Destination:  10.20.30.40
Protocol:     TCP
Port:         22
```

This works well for straightforward policies, but communication between systems is usually more than a collection of unrelated packets. A client initiates a connection, the server responds, data moves in both directions, and the connection eventually ends.

If every packet is evaluated without remembering anything about the communication that came before it, the firewall has limited understanding of the conversation.

That is where stateful inspection changes the model.

---

## Stateful Inspection

A **stateful firewall** keeps track of active connections.

Suppose an internal system initiates a connection:

```text
192.168.10.25:51842 -> 203.0.113.20:443
```

The firewall can create an entry describing that communication in a **state table**, also called a connection table.

A simplified entry might look like:

```text
Protocol:      TCP
Source:        192.168.10.25:51842
Destination:   203.0.113.20:443
State:         Established
```

When traffic returns:

```text
203.0.113.20:443 -> 192.168.10.25:51842
```

the firewall can recognize that it belongs to an existing connection that was already permitted.

```text
Internal Client
      |
      | New permitted connection
      v
   Firewall
      |
      | State entry created
      v
External Server


External Server
      |
      | Return traffic
      v
   Firewall
      |
      | Matches existing state
      v
Internal Client
```

This is why a stateful firewall can allow an internal machine to initiate a connection and receive responses without requiring a broad rule allowing arbitrary external systems to initiate new connections toward that machine.

The firewall is distinguishing between **a new inbound connection** and **return traffic belonging to an existing connection**.

---

## Understanding Connection States

Firewall platforms represent connection state in different ways, but several terms appear frequently:

```text
NEW
ESTABLISHED
RELATED
INVALID
```

`NEW` generally describes traffic attempting to begin a new connection.

`ESTABLISHED` describes traffic associated with a connection the firewall is already tracking.

`RELATED` can describe a new connection that the firewall recognizes as being associated with an existing connection.

`INVALID` describes traffic that the connection-tracking system cannot associate with a valid known state.

These states allow rules to express behavior that would be difficult using only addresses and ports.

A firewall can conceptually say:

```text
Allow new outbound connections from internal clients
Allow return traffic for established connections
Deny unsolicited inbound connections
```

instead of defining every possible response packet individually.

Connection tracking also means that stateful firewalls maintain temporary operational information in addition to static rules. Those entries consume resources and expire according to timers. High connection volumes, unusual traffic patterns, attacks, or incorrect timeout settings can affect how that state is maintained.

---

## Inbound, Outbound, and Forwarded Traffic

Firewall policies frequently separate traffic according to where it is going.

For a host:

```text
             Inbound
Network ----------------> Host

             Outbound
Host -------------------> Network
```

Inbound rules apply to traffic arriving at the system. Outbound rules apply to traffic leaving it.

A device that connects multiple networks introduces **forwarded traffic**:

```text
Network A
    |
    v
 Firewall
    |
    v
Network B
```

The packet arrives at the firewall, but the firewall itself is not the final destination. It forwards the traffic toward another network.

This distinction becomes visible in Linux firewalling, where traffic destined for the local machine can be processed differently from traffic passing through the machine.

For example, an SSH connection to the firewall's own management interface and an SSH connection passing through the firewall toward another server may be subject to completely different rules even though both involve SSH.

---

## Interfaces and Security Zones

A network firewall often connects to several networks through different interfaces.

A simple environment might contain:

```text
                  Internet
                     |
                     v
                 Firewall
                /        \
               /          \
        Internal LAN      DMZ
```

Enterprise firewalls often organize interfaces into **security zones** rather than forcing administrators to reason about every interface independently.

Zones might include:

```text
Inside
Outside
DMZ
Guest
Management
```

Policy can then describe which communication is permitted between those areas.

For example:

```text
Inside -> Internet
Allow required outbound services

Internet -> DMZ
Allow public web traffic

Guest -> Inside
Deny

Management -> Servers
Allow approved administrative access
```

A DMZ is useful for systems that must accept connections from less trusted networks but should not have unrestricted access to internal resources.

A public web server might therefore be reachable from the Internet while communication from that server toward internal systems remains tightly restricted.

```text
Internet
   |
   | Public web traffic
   v
  DMZ
   |
   | Specific required connections
   v
Internal Network
```

If the public-facing system is compromised, the network design can limit which internal systems are directly reachable from it.

---

## Default-Allow and Default-Deny

Eventually a firewall must decide what happens when traffic reaches the end of the relevant policy without matching a specific rule.

A **default-allow** policy permits traffic unless something explicitly blocks it.

```text
Allow everything
except what has been denied
```

A **default-deny** policy does the opposite.

```text
Deny everything
except what has been explicitly allowed
```

Suppose a server is intended to provide only an HTTPS service. A default-deny design might conceptually contain:

```text
Allow required HTTPS traffic
Deny everything else
```

If another service later begins listening unexpectedly, it does not automatically become reachable simply because nobody created a rule blocking its port.

Default-deny is powerful because communication must be intentionally permitted, but it requires administrators to understand what legitimate systems actually need. Applications often depend on DNS, authentication, monitoring, updates, databases, APIs, time synchronization, backup infrastructure, and other services.

A restrictive firewall policy built without understanding those dependencies can break the environment just as effectively as a permissive policy can expose it.

The objective is not maximum blocking. The objective is deliberate communication.

---

## Rule Order Changes the Result

Many firewall platforms evaluate rules in a defined order.

Consider:

```text
1. Allow TCP from 10.0.0.0/8 to 172.16.10.20 port 443
2. Deny all traffic to 172.16.10.20
```

If the firewall uses first-match processing, matching HTTPS traffic from the permitted source network reaches rule 1 and is allowed.

Now reverse them:

```text
1. Deny all traffic to 172.16.10.20
2. Allow TCP from 10.0.0.0/8 to 172.16.10.20 port 443
```

The broad deny rule now matches before the specific allow rule is reached.

The later rule may never be used.

This is known as **rule shadowing**. A broad rule earlier in the policy can make a later, more specific rule ineffective.

Different firewall platforms organize policy differently. Some use ordered rule lists, while others introduce chains, priorities, zones, groups, or multiple processing stages. Reading the rules themselves is therefore only part of understanding the policy. You also need to know how that platform evaluates them.

---

## Drop and Reject Are Not the Same

A firewall can block traffic in different ways.

One option is to **drop** the packet silently:

```text
Client ---- packet ----> Firewall
                         X

Client waits for a response
```

The client receives nothing from the firewall and may eventually time out.

Another option is to **reject** the traffic:

```text
Client ---- packet ----> Firewall
                         X
Client <---- response ---
```

The firewall actively returns a response indicating that communication cannot proceed. Depending on the protocol and implementation, that might involve a TCP reset or an ICMP error.

This difference affects what the client observes.

A silent drop can resemble packet loss, an unreachable system, or another network failure. A rejection provides evidence that something actively responded to the attempt.

Neither observation alone identifies the exact device responsible. Firewall logs and packet captures become useful when determining where the decision occurred.

---

## Source and Destination Restrictions

A rule that permits a service does not have to permit that service from everywhere.

Compare:

```text
Allow TCP 22 from any source
```

with:

```text
Allow TCP 22
from 10.10.50.0/24
to 10.20.30.15
```

The second rule restricts administrative access to a particular source network and destination.

Destination restrictions can be equally important. Employee systems may require web access but have no reason to connect directly to database servers.

A policy might express:

```text
Users -> Internet web services       Allow
Users -> Database network            Deny
Application servers -> Database      Allow required database service
```

This reduces the number of available communication paths while preserving the ones the application architecture actually requires.

---

## Egress Filtering

Firewall policy is not limited to traffic coming into a network.

**Egress filtering** controls traffic leaving a host or network.

An employee workstation might legitimately require access to:

```text
Approved DNS resolvers
Web services
Email infrastructure
Authentication services
Software update systems
Specific internal applications
```

It probably does not require unrestricted outbound communication using every protocol toward every destination.

Controlling outbound traffic can reduce the options available to a compromised system. Malware may attempt to reach external command-and-control infrastructure, transfer information to an unexpected destination, or communicate using a protocol that is unnecessary for that endpoint.

Egress policy can also enforce infrastructure design. If endpoints are supposed to use an organization's internal DNS resolvers, the firewall can permit DNS traffic toward those resolvers while preventing endpoints from sending traditional DNS queries directly to arbitrary external servers.

Useful egress filtering depends on understanding normal application behavior. If the policy is too broad, it provides little control. If it is too narrow without accounting for real dependencies, legitimate software stops working.

---

## Firewalls and Network Segmentation

Firewalls are also used to control communication **inside** an organization.

Imagine an environment divided into:

```text
User Network
Server Network
Database Network
Guest Network
Management Network
```

Creating separate subnets gives those networks different address spaces, but separation becomes much more meaningful when policy controls what is allowed to cross between them.

A web application might require a path such as:

```text
Users
  |
  | HTTPS
  v
Application Servers
  |
  | Required database connection
  v
Database Servers
```

Users do not need direct access to the database simply because the application does.

A guest network can similarly be prevented from directly reaching internal systems:

```text
Guest Network
      |
      X
      |
Internal Servers
```

This limits unnecessary communication and can reduce opportunities for **lateral movement** after a system is compromised.

Segmentation is therefore more than dividing an address range into smaller networks. The security value comes from enforcing which relationships are permitted between those networks.

---

## NAT and Firewalls Are Related, but Different

Firewalls frequently perform **Network Address Translation**, which makes NAT and firewalling easy to treat as the same thing.

They are not.

A firewall policy answers:

```text
Should this traffic be permitted?
```

NAT answers:

```text
Should an address or port be translated?
```

Suppose an internal machine uses:

```text
192.168.10.25
```

and reaches the Internet through a device with:

```text
203.0.113.5
```

The device can replace the private source information before forwarding the traffic.

```text
Before translation

192.168.10.25:51842
        |
        v
203.0.113.20:443


After translation

203.0.113.5:62001
        |
        v
203.0.113.20:443
```

The device maintains the translation information required to send response traffic back to the correct internal system.

A common implementation is **Port Address Translation**, or PAT, where many internal systems share one public IP address and translated port numbers help distinguish their connections.

NAT changes addressing information. Firewall policy decides whether communication is permitted. A single device may perform both functions, but understanding them separately prevents NAT behavior from being mistaken for security policy.

---

## Destination NAT and Port Forwarding

Translation can also modify the destination.

Suppose an organization has:

```text
Public address:  203.0.113.5
Internal server: 10.20.30.40
```

Traffic arriving at:

```text
203.0.113.5:443
```

could be translated toward:

```text
10.20.30.40:443
```

Conceptually:

```text
Internet
   |
   | 203.0.113.5:443
   v
Firewall / NAT
   |
   | 10.20.30.40:443
   v
Web Server
```

This type of destination translation is commonly called **port forwarding** in home and small-network environments.

Creating the translation does not automatically mean that the corresponding connection should be allowed. Many platforms treat translation and security policy as separate parts of processing.

A configuration can therefore contain the correct NAT mapping while a firewall rule still prevents the connection from reaching the internal server.

---

## Host-Based Firewalls

A **host-based firewall** runs directly on the system it protects.

This remains useful even when a network firewall already exists.

Consider two systems connected to the same internal network:

```text
Host A ---------------- Host B
```

Depending on the network architecture, traffic between them may never cross a perimeter firewall. Host B can still use its own firewall to decide which connections it accepts.

Host firewalls can also apply policy that is specific to the endpoint. They may distinguish applications, services, interfaces, network profiles, remote addresses, and local ports in ways that a distant network firewall cannot easily reproduce.

This creates another layer of protection close to the service itself.

---

## Linux Firewalling: Netfilter, nftables, iptables, and UFW

Linux firewall terminology can look confusing because several related technologies appear in documentation.

**Netfilter** is the packet-processing framework in the Linux kernel. It provides hooks that allow network traffic to be filtered, translated, and otherwise processed.

**nftables** is the modern Linux packet-filtering framework and configuration system that replaced much of the older iptables architecture.

**iptables** is the older interface and rule model. It remains widely encountered in existing systems, documentation, scripts, and compatibility environments.

**UFW**, or Uncomplicated Firewall, provides a simpler interface for managing firewall policy and is commonly encountered on Ubuntu-based systems.

The relationship can be viewed broadly as:

```text
Administrator
     |
     v
UFW / nft / iptables
     |
     v
Linux firewall configuration
     |
     v
Kernel packet processing
```

The implementation details vary between distributions and compatibility layers, so the names should not be treated as four different firewalls doing exactly the same job. They represent different parts or generations of the Linux firewalling stack.

---

## Reading a UFW Policy

Before changing a firewall, it is useful to inspect what is already configured.

On a Linux system using UFW:

```bash
sudo ufw status verbose
```

This can show whether UFW is active, its default behavior, and configured rules.

A system might permit SSH with:

```bash
sudo ufw allow 22/tcp
```

but a more restrictive rule could limit access to a particular network:

```bash
sudo ufw allow from 192.168.56.0/24 to any port 22 proto tcp
```

The second rule expresses more than "SSH is allowed." It defines where an SSH connection is permitted to originate.

Numbered rules can be displayed with:

```bash
sudo ufw status numbered
```

This is useful when reviewing the order of existing rules or identifying a specific rule that needs to be removed.

Firewall changes deserve extra care on remotely administered systems. If your current connection depends on SSH and you remove or restrict the rule permitting that connection, the firewall can lock you out immediately.

---

## Reading an nftables Ruleset

On systems using nftables, the current ruleset can be displayed with:

```bash
sudo nft list ruleset
```

A simplified ruleset might contain:

```text
table inet filter {
    chain input {
        type filter hook input priority 0;
        policy drop;

        ct state established,related accept
        tcp dport 22 accept
    }
}
```

The important part is translating the configuration into behavior.

The `input` chain applies to traffic destined for the local machine. Its default policy is `drop`, so traffic reaching the end of the chain without being accepted will be discarded.

The rule:

```text
ct state established,related accept
```

allows traffic associated with connections that the firewall already recognizes.

The rule:

```text
tcp dport 22 accept
```

allows TCP traffic whose destination port is 22.

Read together, the policy roughly means:

```text
Allow traffic belonging to existing permitted connections.
Allow TCP connections to port 22.
Drop other inbound traffic that reaches the end of the chain.
```

Reading firewall configuration this way is far more useful than memorizing syntax. The objective is to understand what traffic the machine will actually accept.

---

## Windows Defender Firewall

Windows provides a host firewall that can apply different policies according to the network profile in use.

The common profiles are:

```text
Domain
Private
Public
```

A corporate laptop can therefore use different firewall behavior when connected to an organization's managed domain network than when connected to a public network.

PowerShell can display the profiles with:

```powershell
Get-NetFirewallProfile
```

Firewall rules can be examined with:

```powershell
Get-NetFirewallRule
```

A system can contain many rules, so filtering is often more useful than displaying everything at once. For example:

```powershell
Get-NetFirewallRule -Enabled True
```

limits the output to enabled rules.

Windows firewall policy can also be associated with applications and services rather than only with port numbers. That allows an administrator to permit communication for a specific program without necessarily creating a broad rule for every process that could use the same port.

---

## Stateless ACLs and Stateful Firewalls

Not every network control remembers connections.

An **Access Control List**, or ACL, can evaluate packets using information such as source, destination, protocol, and port without maintaining the same connection state as a stateful firewall.

This creates an important operational difference.

A stateful firewall can recognize:

```text
Internal client initiated connection
        |
        v
Return traffic belongs to that connection
        |
        v
Allow according to state
```

A stateless control evaluates traffic independently and may require explicit rules for both directions.

This distinction appears often in routers and cloud environments. Some security controls are stateful while others are stateless even when both are represented as lists of allowed and denied traffic.

Before troubleshooting one of these controls, determine which model it uses. Otherwise, a rule set that looks correct can behave differently from what you expect.

---

## Firewalls in Cloud Environments

Cloud networking moves many firewall functions into software-defined infrastructure.

A cloud workload may be protected by several controls before traffic reaches its operating system:

```text
Internet
   |
   v
Cloud Firewall
   |
   v
Subnet Policy
   |
   v
Instance Security Policy
   |
   v
Host Firewall
   |
   v
Application
```

Different cloud platforms use different names for these controls. Common concepts include security groups, network ACLs, managed firewall services, load-balancer policies, and virtual network filtering.

Suppose a web server is correctly listening on TCP port 443 inside a virtual machine. The application can still be unreachable because an instance-level security rule does not permit inbound TCP 443.

The reverse can also happen. The cloud policy permits the connection, but the operating system firewall blocks it before the application receives anything.

Cloud troubleshooting therefore requires following the traffic through each applicable layer instead of assuming that one security rule represents the entire path.

---

## Application-Aware Firewalls

Traditional firewall policy focuses heavily on addresses, protocols, ports, and connection state.

Modern applications make that increasingly limiting.

Many unrelated applications use HTTPS and therefore communicate over the same commonly permitted destination port. A rule that sees only TCP 443 cannot necessarily distinguish ordinary web browsing from a file-sharing service, remote-access application, or another service using encrypted web traffic.

A **Next-Generation Firewall**, or NGFW, can add capabilities such as:

```text
Application identification
User-aware policy
URL filtering
Intrusion prevention
Threat intelligence
TLS inspection
Content controls
```

Instead of creating a rule that simply says:

```text
Allow TCP 443
```

an application-aware firewall may attempt to determine which application is actually using the connection and apply policy accordingly.

Application identification can use protocol behavior, signatures, metadata, TLS information, and other characteristics. It is not always straightforward because modern applications can use encryption, shared cloud infrastructure, dynamic addresses, and changing protocols.

This moves firewalling beyond simple packet-header filtering and closer to understanding what the communication represents.

---

## What Encryption Changes for a Firewall

When application traffic is encrypted, a firewall can still observe network-level information, connection behavior, and some metadata, but the application payload is not normally visible as readable content.

Some organizations deploy **TLS inspection** to inspect selected encrypted traffic.

Conceptually:

```text
Client
   |
   | Encrypted connection
   v
Inspection Device
   |
   | Authorized inspection
   v
Encrypted connection
   |
   v
Destination
```

The inspection system establishes trusted encrypted communication with the managed client, examines traffic according to organizational policy, and establishes protected communication toward the destination.

This requires supporting trust configuration on managed endpoints and introduces significant security, privacy, performance, and compatibility considerations. Some applications can also resist interception through mechanisms such as certificate pinning.

Organizations may exclude sensitive categories of communication from inspection according to policy.

The important firewall-specific point is that encryption changes what can be examined. It does not make traffic invisible at every level. Addresses, connection behavior, traffic volume, timing, and other metadata may still be available even when the application content remains encrypted.

---

## A Web Application Firewall Is a Different Control

A **Web Application Firewall**, or WAF, is designed specifically around web application traffic.

A network firewall might evaluate:

```text
Source address
Destination address
Protocol
Port
Connection state
```

A WAF can examine HTTP-specific information such as:

```text
Request method
URL path
Headers
Cookies
Query parameters
Request body
Application patterns
```

A deployment might look like:

```text
Internet
   |
   v
Network Firewall
   |
   v
WAF
   |
   v
Web Application
```

The network firewall may permit HTTPS communication toward the application infrastructure. The WAF can then evaluate the individual web requests carried through that permitted connection.

This separation is useful because network reachability and application behavior are different security problems. Allowing a client to establish a connection to a web server does not mean every request sent through that connection should be accepted by the application.

---

## Firewall Logging

A firewall can record the decisions it makes.

A simplified event might contain:

```text
Timestamp
Source IP
Source Port
Destination IP
Destination Port
Protocol
Action
Rule
Interface
Zone
Connection state
Bytes transferred
```

For example:

```text
2026-09-10 14:21:08
SRC=192.168.10.25
SPT=51842
DST=10.20.30.40
DPT=443
PROTO=TCP
ACTION=ALLOW
RULE=Internal-Web
```

A blocked connection might appear as:

```text
SRC=198.51.100.25
DST=10.20.30.15
DPT=22
PROTO=TCP
ACTION=DENY
RULE=Block-External-SSH
```

The second event tells us that the firewall evaluated traffic from the listed source toward TCP port 22 on the destination and denied it under a particular rule.

That is different from merely observing that the client failed to connect. The firewall record identifies a policy decision that contributed to the failure.

Firewall logs can also reveal repeated denied connections, unexpected outbound destinations, attempts to cross restricted internal boundaries, and systems communicating through paths that were not expected.

Logging everything is not always practical. Busy firewalls can process enormous amounts of traffic, so organizations decide which events need to be retained and which permitted connections are worth recording.

---

## What an Allow Event Actually Means

An `ALLOW` action has a specific meaning: the firewall permitted the traffic according to the policy it evaluated.

It does not mean the entire application interaction succeeded.

After traffic passes one firewall, several things can still happen:

```text
Another firewall blocks it
The service is not running
The application rejects the request
Authentication fails
TLS negotiation fails
The server returns an error
The response follows an unexpected path
```

A firewall log should therefore be interpreted as evidence about the firewall's decision, not as a complete description of what happened at the application.

The same separation applies to a deny event. The firewall can show that it blocked a connection, but endpoint or application evidence may still be needed to explain which process generated the connection and why it attempted to reach that destination.

---

## Common Firewall Configuration Problems

Many firewall failures are configuration problems rather than failures of the firewall itself.

An obvious example is an unnecessarily broad rule:

```text
Source:       Any
Destination:  Management Server
Port:         22
Action:       Allow
```

If administrators only connect from a dedicated management network, the source can be restricted accordingly.

Temporary rules create another problem. An administrator may open access during testing and forget to remove it after the work is finished.

Policies can also accumulate duplicate, overlapping, unused, and obsolete rules as infrastructure changes. A rule may still reference a server that no longer exists or a network that has been redesigned.

Rule order can make a correct-looking rule ineffective. The correct rule can also be attached to the wrong interface, zone, profile, or traffic direction.

In cloud environments, administrators may change an instance security rule while overlooking a subnet ACL or host firewall.

A firewall policy is therefore not something that can be configured once and forgotten. It needs to remain aligned with the systems and communication paths it is supposed to protect.

---

## Designing Rules Around Required Communication

A useful firewall rule describes the required communication as specifically as practical.

Instead of:

```text
Allow all traffic
from User Network
to Server Network
```

an environment might require:

```text
Allow TCP 443
from User Network
to Web Application Servers
```

The application servers might then require a separate database connection:

```text
Allow TCP 5432
from Application Servers
to PostgreSQL Servers
```

The exact service depends on the environment. The important part is that each rule represents an actual required relationship.

Before creating a rule, useful questions include:

```text
Which system initiates the connection?
Which destination must it reach?
Which service is required?
Does every source in this network need access?
Should the communication work in both directions?
Should the rule be permanent?
Should matching traffic be logged?
Who owns the application or requirement?
```

A rule that can answer those questions is easier to review later than one named something like `TEMP-ALLOW-ALL`.

---

## Firewall Rules Have a Lifecycle

Infrastructure changes continuously, and firewall rules should change with it.

A server may be retired. An application may move to another network. A temporary vendor connection may no longer be required. A cloud service may replace an internal application. An administrative subnet may change.

Without review, firewall policies can gradually accumulate:

```text
Unused rules
Temporary rules
Duplicate rules
Overlapping rules
Rules for retired systems
Rules with no clear owner
Rules whose original purpose is unknown
```

Mature firewall management therefore includes review and cleanup.

Depending on the organization and platform, a rule may be associated with information such as its owner, purpose, change request, creation date, expiration date, and usage history.

This makes the policy easier to maintain because a future administrator can understand why the communication was permitted instead of having to guess whether removing the rule will break something important.

---

## Troubleshooting Firewall-Related Connectivity

Suppose a client cannot connect to:

```text
10.20.30.40:443
```

Changing firewall rules immediately would be a poor first move because several different conditions can produce the same visible failure.

Start by establishing what should happen.

Is `10.20.30.40` the correct destination? Is the service supposed to accept connections on TCP 443? Does the client have a usable route toward the server?

If you control the Linux server, you can check whether something is actually listening with:

```bash
ss -lntp
```

If nothing is listening on the expected address and port, opening the firewall will not create the missing service.

If the service is listening, follow the traffic path.

```text
Client
   |
   v
Client Host Firewall
   |
   v
Network Firewall
   |
   v
Server Host Firewall
   |
   v
Listening Service
```

In a cloud environment, the path may contain additional policy layers.

Check the relevant firewall logs around the time of the connection attempt. Search using the expected source, destination, protocol, and port. A matching deny event can identify the rule responsible for stopping the traffic.

If the firewall reports that the traffic was allowed, move forward rather than repeatedly modifying that firewall. The failure may exist at another point in the path.

A packet capture can help determine whether traffic reached a particular interface and whether anything returned. Application logs can show whether the request reached the service. Operating-system logs can reveal local failures.

Troubleshooting becomes much easier when the connection is treated as a path and each observation identifies how far the traffic progressed.

---

## When the Forward and Return Paths Are Different

Stateful firewalls introduce another problem that is easy to miss.

Imagine that traffic from a client reaches a server through Firewall A:

```text
Client
   |
   v
Firewall A
   |
   v
Server
```

but the server's response returns through Firewall B:

```text
Server
   |
   v
Firewall B
   |
   v
Client
```

Firewall B may receive packets belonging to a connection whose beginning it never observed.

This is **asymmetric routing**.

Asymmetric routing is not automatically a broken network design, but it can create problems when a stateful device expects both directions of a connection to pass through the same inspection point.

This can produce situations where the security policy appears correct and individual network paths appear reachable, yet connections still fail because the firewall cannot associate the returning traffic with valid state.

---

## Connection Timeouts Can Break Long-Lived Sessions

Stateful firewalls cannot keep connection-table entries forever.

Entries consume resources, so the firewall eventually removes inactive state according to configured timeouts.

This can affect applications that maintain connections for long periods but sometimes remain idle.

For example, an application may establish a connection successfully, remain inactive for enough time that the firewall removes its state entry, and then attempt to continue using the old session.

From the application's perspective, the connection may appear to fail unexpectedly after being idle.

This type of behavior can appear with remote administration sessions, database connections, VPN traffic, and persistent application connections.

When a connection repeatedly fails after a similar period of inactivity, state timeout behavior becomes one of the places worth checking.

---

## High Availability Adds Another State Problem

Enterprise firewalls are often deployed redundantly so another device can take over if the active firewall fails.

```text
             Firewall A
Network -----          ----- Network
             Firewall B
```

For a stateless policy, keeping both devices configured consistently may be enough to preserve basic filtering behavior.

Stateful firewalls have another requirement: active connection information.

If Firewall A has tracked thousands of existing connections and Firewall B suddenly takes over without knowing about those sessions, the new firewall may see packets belonging to connections it never observed being created.

Some high-availability firewall systems therefore synchronize connection state between devices.

This illustrates an important part of stateful firewall behavior. The firewall's decisions depend not only on the permanent rule configuration but also on temporary information about communication that is happening right now.

---

## Firewalls Control Reachability, Not Vulnerability

A firewall can significantly reduce exposure by limiting which systems can communicate, but permitted communication still reaches whatever service exists behind the rule.

Consider:

```text
Internet
   |
   | TCP 443 allowed
   v
Web Server
```

The firewall has made a network-level decision: clients are permitted to reach the web service.

It has not established that the web application is securely written, that the web server is correctly configured, that its software is current, or that the service contains no exploitable weakness.

The same applies internally. A firewall may correctly allow an application server to reach a database because the application requires that connection. Security problems can still exist in the database service, its configuration, credentials, software version, or the application using it.

Firewalls therefore reduce and control communication paths. They do not eliminate the need to examine the systems and services reachable through those paths.

---

## A Compact Firewall Reference

After understanding how the pieces connect, the major terms are useful to keep together.

| Concept | Meaning |
|---|---|
| Packet filtering | Evaluates traffic using packet-header information |
| Stateful inspection | Tracks connections and recognizes traffic belonging to existing sessions |
| State table | Temporary information maintained about tracked connections |
| Host firewall | Filters traffic associated with an individual machine |
| Network firewall | Controls traffic crossing between networks |
| Inbound traffic | Traffic entering a system or protected boundary |
| Outbound traffic | Traffic leaving a system or protected boundary |
| Forwarded traffic | Traffic passing through a device toward another destination |
| Egress filtering | Controls outbound communication |
| Security zone | Logical grouping used when applying policy between network areas |
| Default deny | Blocks traffic unless a rule explicitly permits it |
| Rule shadowing | An earlier rule prevents a later rule from being reached |
| Drop | Discards traffic without necessarily returning a response |
| Reject | Blocks traffic and actively returns a response |
| NAT | Translates network addresses or ports |
| SNAT | Changes source addressing information |
| DNAT | Changes destination addressing information |
| PAT | Allows multiple connections to share an address using port translation |
| ACL | Traffic-control rule set that may operate without stateful connection tracking |
| NGFW | Firewall with additional application-aware and security inspection capabilities |
| WAF | Security control focused on HTTP and web application traffic |
| TLS inspection | Authorized decryption and inspection of selected encrypted traffic |
| Asymmetric routing | Forward and return traffic follow different network paths |

A firewall policy becomes much easier to read when you stop seeing it as a list of port numbers and start translating each rule into a communication relationship: **who is trying to reach what, through which service, in which direction, and under what conditions?**

---

## Reachable Does Not Mean Secure

Once firewall behavior is understood, an open path through the network becomes much more precise. We can distinguish a service that is unreachable because policy blocks the path from a service that is deliberately exposed and accepting communication.

But reachability is only the beginning of assessing that service.

Suppose a server is intentionally reachable and we identify the software running behind it. We can interact with the service to learn more about its behavior and configuration. We can compare the software and its characteristics with known security weaknesses. Under controlled testing conditions, we may eventually need to determine whether a suspected weakness can actually be demonstrated.

Those activities answer different questions.

Learning more about a service does not automatically mean we have found a vulnerability. Finding information that suggests a known vulnerability does not automatically prove that the system is exploitable. Demonstrating exploitation goes further than either of those activities and produces a different level of evidence.

Keeping those boundaries clear is what turns "I found an open service" into a structured security assessment rather than a sequence of tools being run without understanding what each result actually establishes.

**Vulnerability Scanning vs Enumeration vs Exploitation**
