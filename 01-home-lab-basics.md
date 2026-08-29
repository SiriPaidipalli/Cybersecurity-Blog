# Everything You Should Know Before Starting Your First Cybersecurity Home Lab

*A beginner-friendly guide to VMs, Kali Linux, networking modes, vulnerable machines, and what you actually need before building your first lab.*

## Before We Start

Getting started with hands-on cybersecurity can be surprisingly confusing.

There are countless resources explaining individual tools, commands, and concepts, but when you’re completely new, the difficult part is often figuring out what to learn first and how everything connects. A tutorial might tell you to scan a target, inspect a service, analyze some traffic, or investigate an alert, while quietly assuming you already understand the fundamentals behind each step.

That is the reason I’m starting this series.

I want to build a practical path through cybersecurity fundamentals, starting from the very beginning and gradually moving into tools, techniques, labs, and real security workflows. The focus will not just be on which command to run or which button to click, but on understanding what we are doing, why we are doing it, and what is actually happening behind the scenes.

This is also part of my own journey of learning cybersecurity more deeply through hands-on practice. As I learn, experiment, build, and explore, I’ll document the process here in a way that I hope makes the path a little clearer for someone starting from the same place.

No assumed expertise. No jumping ten steps ahead.

We’ll start with the foundations and build from there.

And the best place to start is by creating somewhere safe to experiment: a cybersecurity home lab.

When I first decided to get more hands-on with cybersecurity, I thought the process would be simple. Pick a lab, follow the instructions, learn a tool, done.

Instead, I kept running into instructions like:

> *Deploy the machine, connect to the VPN, scan the target IP, identify the open ports, and enumerate the services.*

If you already have experience with security labs, that sentence probably sounds completely normal.

If you are just starting, it raises about ten more questions.

What exactly am I deploying? What is the target? Where am I supposed to run the scan? Why is everyone using Kali Linux? What does the VPN have to do with this? Can I just scan another computer? And how am I supposed to understand the results when I barely understand what I am doing in the first place?

I realized that there are plenty of tutorials that tell you **how to set up a cybersecurity home lab**, but many of them assume you already understand what you are building.

So before downloading Kali Linux, installing a dozen security tools, or running your first Nmap scan, let’s understand the environment first.

## What Exactly Is a Cybersecurity Home Lab?

A cybersecurity home lab is simply a controlled environment where you can safely practice security concepts, tools, attacks, and defenses.

And no, you do not need a room full of servers and networking equipment to have one.

For a beginner, your laptop is enough to get started.

A home lab can eventually be used to practice things like:

- Network scanning
- Packet analysis
- Linux administration
- Vulnerability assessment
- Log analysis
- Security monitoring
- Firewall configuration
- Web application security
- Incident investigation

The important part is that you are working inside an environment that you own or have permission to test.

The easiest way to create this environment is with **virtual machines**.

## Physical Machines vs. Virtual Machines

Let’s say you have a laptop running Windows.

That is your physical computer.

Normally, if you wanted another computer running Linux, you would need another physical machine.

Virtualization changes that.

Using software such as VirtualBox or VMware, your computer can simulate another computer inside itself.

That simulated computer is called a **Virtual Machine**, or VM.

A simple setup might look like this:

```text
Your Laptop
      |
Virtualization Software
      |
Kali Linux VM
      |
Test Machine VM
```

Your actual laptop is called the **host**.

The virtual machines running inside it are called **guests**.

The software responsible for creating and managing those virtual machines is called a **hypervisor**.

Virtual machines are incredibly useful for cybersecurity because you can create multiple systems, connect them together, experiment with them, and even intentionally break them without needing multiple physical computers.

## One Feature You Should Know About: Snapshots

Imagine spending two hours configuring a machine and then accidentally breaking something.

Normally, that could mean starting over.

Virtual machines give us something much better: **snapshots**.

A snapshot saves the state of your VM at a particular point in time.

You can:

1. Configure your machine
2. Take a snapshot
3. Experiment with it
4. Break something
5. Restore the snapshot

You are essentially creating a checkpoint before experimenting.

If you are going to use VMs for cybersecurity labs, get into the habit of taking snapshots.

Future you will be grateful.

## What Is an ISO?

This was one of those terms I kept seeing everywhere without anyone stopping to explain it.

You’ll see instructions such as:

> *Download the Kali Linux ISO.*

An ISO file is essentially an image containing the files needed to install an operating system.

Think about installing Windows or Linux on a brand-new computer. The computer needs installation media containing the operating system.

In a virtual machine, an ISO can act as that installation media.

You may also come across **prebuilt virtual machine images**.

The difference is useful to know:

**ISO:** You create the VM and install the operating system yourself.

**Prebuilt VM:** Much of that setup has already been done for you.

Neither option is automatically better. It depends on what you are trying to learn and how much setup you want to do yourself.

## Why Does Everyone in Cybersecurity Use Kali Linux?

Spend five minutes watching cybersecurity tutorials and Kali Linux will appear somewhere.

So what actually makes Kali special?

Kali Linux is a Linux distribution designed primarily for penetration testing and security research. It comes with a large collection of security tools that would otherwise need to be installed separately.

Some names you will eventually encounter include:

```text
Nmap
Wireshark
Burp Suite
Metasploit
Gobuster
John the Ripper
```

But there is something important to understand:

**Kali Linux is not magic.**

Installing Kali does not suddenly turn your computer into some kind of hacking machine.

It is simply an operating system that gives security professionals convenient access to many useful tools.

And if you are a beginner, you absolutely do not need to learn every tool inside Kali.

In fact, trying to do that would probably make learning cybersecurity harder.

Start with the problem you are trying to solve. Then learn the tool that helps you solve it.

## What Are the “Attacker” and “Target” Machines?

Security tutorials frequently use these terms:

**Attacker machine**

and

**Target machine**

They sound dramatic, but the concept is straightforward.

Suppose we create two virtual machines.

One runs Kali Linux.

The other is a machine intentionally configured for us to practice against.

Our environment might look like this:

```text
       Kali Linux
    Attacker Machine
           |
           |
           | Scan / Test / Analyze
           |
           v
      Target Machine
```

The attacker machine is simply the system from which we perform our security testing.

The target is the system we are testing.

Calling something an “attacker machine” does not automatically mean we are doing something malicious.

In a home lab, both machines belong to us and exist specifically so we can learn.

There are even operating systems and applications intentionally designed to contain vulnerabilities so security students can safely practice finding and understanding them.

## How Do These Virtual Machines Talk to Each Other?

This is where things start getting slightly more interesting.

If Kali Linux is one virtual computer and our target is another virtual computer, how does traffic move between them?

They need a network.

Virtualization software allows us to create virtual networks, and you will commonly encounter terms such as:

### NAT

NAT commonly allows your VM to access external networks through your host computer.

Your VM gets network connectivity without simply becoming another directly exposed device on the physical network.

### Bridged Networking

In bridged mode, the VM connects more directly to the same physical network as your host.

To other devices on that network, the VM can behave much more like another computer connected to the network.

### Host-Only Networking

Host-only networking creates a more isolated network involving your host and virtual machines.

This can be useful for security labs because we often want our intentionally vulnerable machines separated from the rest of the network.

You do not need to master networking modes right now.

Just understand this:

**How you connect your virtual machines matters.**

If you intentionally create a vulnerable machine, you should understand where that machine is accessible before turning it on.

We will explore networking and isolation properly when we build the actual lab.

## What Should Your First Home Lab Actually Look Like?

This is where I think beginners can easily overcomplicate things.

Search for cybersecurity home labs and you will find setups involving:

- Kali Linux
- Windows Server
- Windows 11
- Active Directory
- Splunk
- Security Onion
- pfSense
- Ubuntu servers
- IDS/IPS systems
- Several network segments

Those are great environments.

Eventually.

But if you are learning what an IP address and an open port actually mean, building an enterprise SOC in your bedroom probably isn’t the best first step.

Your first lab can be extremely simple:

```text
              YOUR COMPUTER
                   |
          Virtualization Software
                   |
          +---------+---------+
          |                   |
     Kali Linux          Test Machine
   Security Tools       Practice Target
```

That's enough to start learning.

Later, as we understand more concepts, we can expand the environment.

For example:

```text
            Cybersecurity Home Lab
                      |
        +-------------+-------------+
        |             |             |
       Kali        Windows        Linux
        |             |             |
        +-------------+-------------+
                      |
                 Logs / SIEM
```

There is no reason to build all of this on day one.

A good lab should grow with your knowledge.

## Do You Need a Powerful Computer?

Not necessarily, but virtual machines do consume your computer’s resources.

Every VM needs some combination of:

- RAM
- CPU
- Storage
- Network resources

If your computer has **8 GB of RAM**, you can still start with a small lab, but you will need to be careful about how many machines you run simultaneously.

With **16 GB or more**, running multiple beginner VMs becomes much more comfortable.

Storage is another thing beginners sometimes forget about.

Virtual machines can easily consume tens of gigabytes, especially as you create more of them and start taking snapshots.

Before building a large lab, check how much free disk space you actually have.

### A Note for Apple Silicon Users

If you have a newer Mac with an M-series processor, your computer uses the ARM architecture.

That matters because not every virtual machine image designed for traditional x86 computers will run directly on ARM.

It does not mean you cannot build a cybersecurity lab on a Mac.

It simply means you need to pay attention to architecture compatibility when downloading operating systems and VM images.

## One Important Rule Before You Start Scanning Anything

Learning cybersecurity tools comes with responsibility.

Once you install tools such as Nmap, it becomes technically possible to point them at systems outside your home lab.

That does not mean you should.

As a beginner, keep your testing to:

- Systems you own and are authorized to test
- Virtual machines inside your own lab
- Intentionally vulnerable training environments
- Cybersecurity platforms that explicitly give you permission to test their machines
- Systems for which you have clear authorization

Your home lab gives you something extremely valuable: **a place where you are free to experiment without wondering whether you should be touching the system in the first place.**

Security tools themselves are not inherently malicious. Context and authorization matter.

Build good habits from the beginning.

## A Few Words You Should Know Before Moving Forward

You do not need textbook definitions for all of these yet.

You just need to recognize them.

**Host -** Your physical computer

**Guest -** An operating system running inside a VM

**VM -** A virtual computer

**Hypervisor -** Software used to create and manage VMs

**ISO -** An image commonly used to install an operating system

**Kali Linux -** A Linux distribution containing many security tools

**Target -** The system you are testing

**IP Address -** An address used to identify and communicate with a device/interface on a network

**Snapshot -** A saved state of a virtual machine

**NAT -** A networking configuration commonly used to give VMs network access through the host

**Bridged -** Connects a VM more directly to the physical network

**Host-Only -** Creates an isolated network between the host and participating VMs

You will understand these much better once you actually start using them.

## What You Do NOT Need to Worry About Yet

You do not need to master every cybersecurity tool before starting. Begin with the basics: understand your lab environment, learn a few essential Linux commands, and use Nmap to scan systems you are authorized to test.

You can learn advanced tools such as Metasploit, exploit development, Active Directory, and additional virtual machines later as your skills grow.

Right now, you need to understand your environment.

When you eventually type:

```bash
nmap <target-ip>
```

I don’t want it to be a command you copied from a tutorial.

You should understand **which computer is running Nmap, which computer is being scanned, how those computers are connected, what the target IP represents, and why you are performing the scan in the first place.**

Once that foundation makes sense, the tools become much easier to learn.

## Where Do We Go From Here?

At this point, we haven’t hacked anything.

We haven’t even installed anything.

And that’s intentional.

Before using security tools, I wanted to understand what environment those tools were going to operate in.

Now we know what a virtual machine is, why Kali Linux is commonly used, what a target machine is, how virtual machines can communicate, why isolation matters, and what a basic home lab should look like.

So now we can actually build one.

In the next part of this series, we’ll set up the virtualization environment and create our first virtual machine. Instead of blindly following installation instructions, we’ll look at what the important settings actually mean and why we’re choosing them.

After that, we’ll start networking the machines together.

And eventually, we’ll run that first Nmap scan.

One step at a time.
