# Virtual Machines Explained: What They Are and Why Security Labs Use Them

*Learning cybersecurity from the ground up. Before we build our first lab, there is one piece we need to understand properly: virtual machines.*

Virtual machines let us run systems like Kali Linux and a target machine on the same physical computer.

But there are still some questions worth answering.

When we give a VM 4 GB of RAM, where does that memory come from? When the VM sees a disk or a network adapter, are those actual pieces of hardware? And how can one laptop behave like several different computers at the same time?

Before we start creating machines for our cybersecurity lab, I want to understand that properly.

## What Is a Virtual Machine?

Start with the computer you already have.

Your laptop has a processor, RAM, storage, and network hardware. Your operating system, whether that is Windows, macOS, or Linux, uses those physical resources to run everything on your computer.

A **virtual machine**, usually shortened to **VM**, lets you create another computer environment using those same physical resources.

That VM can run its own operating system. You can install applications on it, create users, give it storage, connect it to a network, restart it, shut it down, and configure it separately from your normal operating system.

So if I have a MacBook, for example, I could continue using macOS normally while also running Linux inside a VM.

I did not suddenly add another physical computer to my desk. My laptop is still providing the actual hardware. The VM is being given virtual versions of those resources.

That distinction becomes easier to see with an example.

Suppose your laptop has:

```text
16 GB RAM
8 CPU cores
500 GB storage
```

You could create a VM and assign it:

```text
4 GB RAM
2 CPU cores
40 GB virtual disk
```

That 4 GB of memory is still coming from your laptop. The same applies to the CPU and storage.

This is also why running several VMs can slow your computer down. Each VM needs resources, and your physical computer only has so much to share.

So when we eventually create a VM and see settings asking how much RAM or how many CPU cores to assign, we will know what we are actually deciding.

## Host and Guest

There are two words that come up constantly when working with virtual machines: **host** and **guest**.

The **host** is the physical computer providing the resources.

The **guest** is the operating system running inside the virtual machine.

If I am using a Windows laptop and running Kali Linux in a VM, Windows is on my host machine and Kali is the guest operating system.

One host can also run more than one guest:

```text
                 Physical Laptop
                      HOST
                        │
              ┌─────────┼─────────┐
              │         │         │
           Kali VM   Ubuntu VM  Windows VM
            GUEST      GUEST      GUEST
```

This diagram is worth remembering because it explains one of the main reasons virtualization is so useful for a cybersecurity lab. One physical laptop can give us several separate systems to work with.

## Where Does the Hypervisor Fit In?

But how does the laptop actually create and manage these VMs?

That is where a **hypervisor** comes in.

A hypervisor is the software layer responsible for creating and managing virtual machines and giving them access to resources such as CPU, memory, storage, and networking.

If you have looked into creating VMs before, you may have seen names such as VirtualBox, VMware, or UTM.

For the kind of setup we might use on a personal computer, the basic idea looks like this:

```text
     Physical Hardware
             │
             ▼
   Host Operating System
             │
             ▼
   Virtualization Software
             │
         ┌───┴────┐
         │        │
         ▼        ▼
     Kali VM   Ubuntu VM
```

So when virtualization software asks us to choose:

```text
Memory:      4 GB
Processors:  2
Disk:        40 GB
```

we are deciding what resources that VM will have available to it.

## Why Not Just Install Kali on My Laptop?

This was one of the questions I had when I first started looking at cybersecurity labs. **If I want to use Kali Linux anyway, why not just install it directly on my laptop?**

You technically can, but a VM is much more convenient for the kind of lab we are trying to build.

For one thing, you can keep your normal computer exactly as it is. You can continue using Windows or macOS for everyday work and run Kali separately whenever you want to practice.

More importantly, we are eventually going to need more than one machine.

Imagine that we want to learn network scanning. We could create a Kali VM and another Linux VM and connect them through a virtual network.

```text
Kali VM  ───── Virtual Network ───── Linux VM
```

Now Kali has another system to communicate with.

Later, we could have something like:

```text
                   Virtual Network
                          │
              ┌───────────┼───────────┐
              │           │           │
           Kali VM    Windows VM   Linux VM
```

All three systems could be running on the same physical laptop.

This is where the idea of a cybersecurity home lab starts making more sense. We do not necessarily need several physical computers sitting around us. Virtualization lets us create a small environment of our own.

And because these are systems we own and control, we have somewhere appropriate to practice techniques that we should never be trying against random machines on the internet.

## Breaking Things Is Less of a Problem

There is another practical reason I like the idea of using VMs for learning security: things are going to break.

We might change a configuration incorrectly, install something that causes a problem, or mess up a network setting while trying to understand how it works.

Doing that on the laptop you use every day is not ideal.

A VM gives us a separate environment where we can experiment without making every mistake a problem for our main operating system.

One feature makes this particularly useful: **snapshots**.

A snapshot saves the state of a VM at a particular point. The easiest comparison is a checkpoint in a game.

```text
Clean VM
   │
   │ Take Snapshot
   ▼
Experiment
   │
   │ Something breaks
   ▼
Restore Snapshot
   │
   ▼
Back to the Earlier State
```

Suppose we spend time configuring a Windows VM for a lab. Before making major changes, we could create a snapshot called `Clean Windows Lab`.

If the experiment goes badly, we may be able to restore that snapshot instead of rebuilding the machine from scratch.

Snapshots are not unlimited free copies of a machine, though. They still use storage on the host, which is something to remember once we start accumulating VMs and snapshots.

## What Is a Virtual Disk?

When we create our first VM, we will also be asked how much storage to give it.

The VM does not normally have another physical hard drive hidden somewhere inside the laptop. Instead, the virtualization software creates a **virtual disk**.

The guest operating system sees that virtual disk as normal storage. If we give Kali a 40 GB virtual disk, Kali can treat it as a disk available to the operating system.

Underneath all of this, that data still has to be stored on our physical computer.

This matters because VMs can take up quite a bit of space. Operating systems, installed applications, files, and snapshots all eventually consume storage on the host.

So before downloading several VM images because a tutorial told us to, it is worth knowing how much free storage we actually have.

## How Does a VM Connect to a Network?

A physical computer needs networking hardware to communicate with other systems. A VM needs the same capability, but we do not need another physical network card for every VM we create.

The virtualization software can provide a **virtual network adapter**.

From the guest operating system’s perspective, it has a network interface it can use to communicate.

This is how we can eventually have something like:

```text
Kali VM  ───── Virtual Network ───── Ubuntu VM
```

Each machine can have its own IP address and communicate over that virtual network.

If the words **IP address** or **network interface** still feel vague, that is fine. Networking gets its own proper explanation later in this series rather than being squeezed into a few paragraphs here.

For now, the useful thing to understand is that VMs are not just isolated windows running operating systems. We can connect them together and create an actual network of machines.

## A VM Does Not Mean “Anything I Do Here Is Safe”

This is worth making explicit before we start using VMs.

A virtual machine gives us useful separation, but it is not a magic safety box.

Depending on its configuration, a VM may have internet access. It may be able to communicate with the host or other devices on the network. Features such as shared folders, clipboard sharing, and USB access can also create connections between the guest and host.

So I do not want to go into future labs thinking:

> *“It’s inside a VM, so nothing can happen.”*

A better rule is:

> ***Isolation is something we configure and understand, not something we assume.***

That will matter much more once we start working with intentionally vulnerable systems.

## Before Installing Anything, Check Your Computer

Now that we know a VM uses the host’s resources, take a minute to check the computer you plan to use.

Know how much RAM and free storage you have, what processor or chip you are using, and whether your system is x86–64 or ARM.

Different processor architectures can affect which VM images and virtualization software we can use. For example, newer Apple Silicon Macs use ARM-based chips, while many Windows PCs use x86–64 processors.

You do not need to memorize any of this. We just need to know what resources and architecture we are working with before deciding what kind of VM to create.

## So Why Kali Linux?

Now that we understand what is actually happening when we create a virtual machine, the next question is what we should run inside it.

If you have spent any time looking at cybersecurity tutorials, one name probably keeps showing up: **Kali Linux**.

But Kali is sometimes introduced in a strange way. Tutorials tell you to install it before explaining why you need it, which can make it seem like Kali itself is what turns a computer into a “hacking machine.”

It doesn’t.

So before we install anything, I want to understand what Kali actually is, how it differs from a normal Linux distribution, why it comes with so many security tools, and whether we even need all of them.

That is next:

**Kali Linux Isn’t Magic: What It Actually Is and When You Need It**
