# IP Addresses, Subnets, Gateways & DNS: The Networking You Need for Labs

*Learning cybersecurity from the ground up. We have a machine to work from. Before we start scanning anything, we need to understand how machines actually find and communicate with each other.*

Soon, we are going to run commands that expect us to provide something called a **target IP**.

You will see instructions like:

> *“Find the target IP and scan it.”*

That sounds simple until you realize how much is being assumed.

What exactly is an IP address? Why does my laptop have one? Why might a virtual machine have a different one? What does something like `192.168.1.25/24` mean? And if I type a website name instead of an IP address, how does my computer know where to send the traffic?

Before we start scanning machines, I want those things to make sense.

## Start With a Simple Problem: Computers Need to Find Each Other

Imagine we have two machines in our lab:

```text
Kali VM  ───────────────  Linux VM
```

We want Kali to communicate with the Linux machine.

Knowing that the Linux machine exists is not enough. Kali needs some way to identify where to send the network traffic.

That is where **IP addresses** come in.

## What Is an IP Address?

An IP address is an address assigned to a network interface so it can communicate using the Internet Protocol.

For now, we are going to focus on **IPv4**, because addresses like `192.168.1.10`, `10.0.0.5`, and `172.16.0.20` will appear constantly in beginner labs.

An IPv4 address contains four numbers separated by periods. Each number can range from 0 to 255.

So our lab might eventually look like this:

```text
Kali VM                         Linux VM
192.168.56.10                  192.168.56.20
        │                            │
        └──────── Lab Network ───────┘
```

Now each machine has an address it can use for network communication.

If a lab tells us the target IP is `192.168.56.20`, we finally know what that means.

It is telling us which network address belongs to the machine we are supposed to interact with.

That is the value we will eventually give tools such as Nmap.

## Your Computer Can Have More Than One IP Address

There is one detail worth understanding early.

We often casually say:

> *“This computer’s IP address is…”*

But technically, an IP address is associated with a **network interface**.

A laptop might have a Wi-Fi interface, an Ethernet interface, virtual interfaces created by virtualization software, and other networking interfaces.

Those interfaces can have different addresses.

This becomes especially relevant when we start using virtual machines because our host and our VMs may each have their own network interfaces and IP addresses.

So if you run a command later and see several addresses, that does not automatically mean something went wrong.

The real question becomes:

**Which network interface and which IP address are relevant to the network I am working with?**

That is much more useful than simply looking for the first IP-looking number on the screen.

## Public and Private IP Addresses

You may notice that home labs frequently use addresses beginning with numbers such as `10.x.x.x`, `172.16.x.x`, or `192.168.x.x`.

Many of these are **private IP addresses**.

IPv4 reserves three ranges for private networks:

```text
10.0.0.0       - 10.255.255.255
172.16.0.0     - 172.31.255.255
192.168.0.0    - 192.168.255.255
```

Private addresses are used inside networks rather than being globally routable across the public internet.

Your home network, for example, may give different devices addresses such as `192.168.1.5`, `192.168.1.20`, and `192.168.1.34`.

Meanwhile, your network also communicates with the wider internet through a **public IP address**.

We do not need to unpack NAT or public internet routing yet.

For now, the distinction is simply this:

**Private IPs are commonly used inside local networks and are not directly routable on the public internet. Public IPs are globally routable and are used to identify systems or networks on the internet.**

This is why the target addresses you see in home labs often look very different from public addresses you might encounter elsewhere.

## But What Does the `/24` Mean?

Eventually, you will see an address written like this: `192.168.1.20/24`.

The IP address itself is `192.168.1.20`.

The `/24` tells us information about the **network that address belongs to**.

This is where subnets come in.

## What Is a Subnet?

A network may contain many devices, and we need some way to determine which portion of an IP address identifies the network and which portion can vary between devices on that network.

A **subnet** is a logical subdivision of an IP network.

Let’s use `192.168.1.20/24`.

For a `/24` IPv4 network, the first 24 bits identify the network portion.

At the level we need right now, you can think of this network as `192.168.1.x`.

So we might have:

```text
192.168.1.10     Laptop
192.168.1.20     Kali VM
192.168.1.30     Linux VM
```

All of these addresses can belong to the same `/24` subnet.

The subnet itself would commonly be written as `192.168.1.0/24`.

This tells us something much more useful than simply knowing one machine’s IP address.

It tells us the network that machine is part of.

## Why Does the Subnet Matter in a Security Lab?

Suppose Kali has `192.168.56.10/24` and our target has `192.168.56.20/24`.

They appear to be on the same `/24` network.

Now imagine Kali instead has `10.0.2.15/24`, while the target still has `192.168.56.20/24`.

Something about our network setup is different.

That does not automatically mean the machines can never communicate. Routers can connect different networks.

But it tells us that communication is no longer simply happening between two addresses on the same local subnet.

This is one reason understanding IP addresses and subnets becomes useful when a lab does not work.

Instead of only thinking:

> *“Why can’t Kali reach the target?”*

we can start asking:

> *“What network is Kali on? What network is the target on? Is there a path between them?”*

That is a much better troubleshooting question.

## Do I Need to Learn Subnetting Math Right Now?

No.

Subnetting can become much more detailed. You can calculate address ranges, convert addresses to binary, work with different prefix lengths, and determine exactly how many addresses a subnet contains.

Right now, I just want `/24` to stop looking like a mysterious number attached to an IP address.

If you see `192.168.56.10/24`, you should at least recognize:

`192.168.56.10` is the interface's IPv4 address.

`/24` describes the network prefix.

That is enough for where we are.

## What Happens When the Destination Is on Another Network?

So far, our examples have mostly involved machines on the same network.

But your laptop does not only communicate with devices inside your home network.

It also needs to reach systems elsewhere.

For that, it needs somewhere to send traffic that is destined for another network.

That brings us to the **default gateway**.

## What Is a Default Gateway?

A default gateway is the device your machine normally sends traffic to when the destination is outside its directly connected network and it does not have a more specific route.

On a typical home network, that gateway is usually your router.

Imagine:

```text
Your Laptop
192.168.1.20
     │
     ▼
Home Router
192.168.1.1
     │
     ▼
Other Networks / Internet
```

Your laptop can communicate directly with devices on its local network.

But when it needs to reach an address outside that network, the router can act as the next step in the path.

That is why you may see network information that looks something like:

```text
IP Address:       192.168.1.20
Subnet:           255.255.255.0
Default Gateway:  192.168.1.1
```

These values are related.

The IP address tells us the address assigned to the interface.

The subnet information helps determine which addresses are considered local.

The default gateway provides a route toward destinations outside that local network.

## And Then There Is DNS

There is still another problem.

Humans do not usually browse the internet by remembering IP addresses.

We type names.

For example, `example.com`.

But computers still need an address to communicate with the destination.

**DNS**, or the **Domain Name System**, helps translate domain names into information such as IP addresses.

Conceptually:

```text
You enter: example.com
         ↓
      DNS lookup
         ↓
An IP address is returned
         ↓
Your computer can communicate with the destination
```

This is why DNS is often compared to a directory.

We remember the name.

DNS helps us find the address associated with it.

The real DNS system is much more detailed than this, and later we will look at DNS specifically from a security perspective.

For now, the connection I want to understand is:

**A domain name and an IP address are not the same thing. DNS helps connect the two.**

## IP Address, Subnet, Gateway, and DNS Are Solving Different Problems

These terms are often shown together in network settings, which can make them blur into one big networking concept.

They are doing different jobs.

Suppose your computer has:

```text
IP Address:       192.168.1.20
Subnet:           255.255.255.0
Default Gateway:  192.168.1.1
DNS Server:       192.168.1.1
```

At a beginner level, I would read that as:

**IP address:** Who am I on this network?

**Subnet information:** Which addresses are part of my local network?

**Default gateway:** Where do I send traffic when the destination is elsewhere?

**DNS server:** Who can I ask when I know a name but need the corresponding address?

That mental model is much more useful than memorizing four definitions separately.

## Let’s Look at This on Your Own Computer

We can actually inspect some of this without building the lab yet.

We are not scanning anything. We are simply asking our own computer about its network configuration.

## On Windows

Open Command Prompt and run:

```bash
ipconfig
```

We are trying to find the network configuration assigned to your computer.

Look for the network adapter you are currently using. You may see information such as:

```text
IPv4 Address
Subnet Mask
Default Gateway
```

Do not worry if there are several adapters. VPNs, virtualization software, Bluetooth, Wi-Fi, Ethernet, and other software can create additional network interfaces.

## On macOS or Linux

You can inspect your network interfaces with:

```bash
ifconfig
```

On many Linux systems, you can also use:

```bash
ip addr
```

Again, the goal is not to understand every line.

Try to identify the interface you are actually using and its IP address.

If you see something you do not recognize, leave it alone for now. Networking commands can produce much more information than we currently need.

## One Small Test: What Does DNS Actually Return?

We can also ask DNS about a domain name.

For example:

```bash
nslookup example.com
```

Before running it, here is what we are trying to find out:

If I know the name `example.com`, what address information can DNS give me for that name?

Your exact output may vary.

Look for the address or addresses returned for the domain.

You may see IPv4, IPv6, or both.

The important part is not memorizing the output. It is seeing the connection we just discussed:

**Domain name → DNS → address information**

*Running **`nslookup example.com`** shows DNS resolving the domain name to IP addresses.*

## One Detail That Will Matter Once We Use Virtual Machines

When we create our Kali VM, it will have its own network configuration.

That means the IP address you see on your physical laptop is not automatically the IP address you will see inside Kali.

For example, we might eventually have:

```text
Physical Laptop
192.168.1.20
```

```text
Kali VM
10.0.2.15
```

Or the addresses might look completely different depending on how the virtual network is configured.

This is why blindly copying an IP address from somewhere in a tutorial can cause so much confusion.

Before interacting with a machine, we need to know **which machine the address belongs to and which network we are working on.**

## What You Do Not Need to Worry About Yet

You do not need to calculate complicated subnets in your head or understand internet routing protocols.

For now, being able to look at a basic network configuration and distinguish the **IP address, subnet information, default gateway, and DNS server** is enough.

## An IP Address Gets Us to the Machine. What Happens Next?

We are getting closer to that first scan.

If a lab gives us `192.168.56.20`, we now know that this is an IP address identifying a network interface on the target machine.

But knowing the machine’s address still does not tell us **what is available on that machine**.

A computer can be running a web server, accepting SSH connections, providing DNS services, or listening for many other kinds of network communication.

How can all of those services use the same IP address?

That is where **ports and protocols** come in.

Before we run Nmap, we need to understand what it will actually be looking for.

That is next:

**Ports & Protocols: What You’re Actually Looking for When You Scan a Machine**
