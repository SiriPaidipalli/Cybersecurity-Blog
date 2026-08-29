# Nmap Part 2: Finding Services, Versions, and Understanding the Results

*Learning cybersecurity from the ground up. Finding an open port tells us where a machine is accepting connections. The next step is figuring out what is actually running there.*

Suppose an Nmap scan gives us:

```text
PORT     STATE   SERVICE
22/tcp   open    ssh
80/tcp   open    http
```

At first, this looks like we already know quite a lot. Port 22 is open and Nmap labels it SSH. Port 80 is open and Nmap labels it HTTP.

But there is still a lot we do not know.

What SSH software is actually running? Which version? What web server is behind port 80? What if a service is running on a completely unexpected port?

A basic port scan gives us a map of where a machine is responding. **Enumeration** takes that information further by trying to identify what is actually available behind those ports.

### From Open Ports to Actual Services

A port number gives us a clue about what may be running, but it is not proof.

Port 22 is commonly associated with SSH. Port 80 is commonly associated with HTTP. Port 443 is commonly associated with HTTPS.

Those conventions are useful, but software does not have to follow them. SSH could be configured on port `2222`, a web application might run on `8080`, and an internal service might use a port you have never seen before.

So if we find:

```text
8080/tcp   open
```

the useful question is not simply, “What usually runs on 8080?”

It is:

> ***What is actually responding on TCP port 8080 on this machine?***

That is the shift from basic port discovery toward service enumeration.

### Service and Version Detection with `-sV`

Nmap provides service and version detection with:

```bash
nmap -sV <target-ip>
```

The `-sV` option tells Nmap to investigate open ports and try to identify the software responding behind them.

A basic scan might show:

```text
PORT     STATE   SERVICE
22/tcp   open    ssh
80/tcp   open    http
```

With service detection enabled, the result may become more detailed:

```text
PORT     STATE   SERVICE   VERSION
22/tcp   open    ssh       OpenSSH 8.9p1
80/tcp   open    http      Apache httpd 2.4.x
```

The exact output depends on the target, so we should never assume in advance that Nmap will identify everything perfectly.

The difference between the two scans is important. The basic scan tells us which ports appear open and associates common service names with them. `-sV` goes further and actively probes those ports in an attempt to identify the software and, when possible, its version.

That additional information becomes much more useful when we are trying to understand the actual attack surface of a system.

### How Nmap Identifies a Service

Nmap does not simply look at the port number when service detection is enabled.

It uses a database of service probes and known response signatures. At a simplified level, the process looks like this:

```text
Open port discovered
        ↓
Nmap sends service-specific probes
        ↓
The service responds
        ↓
Nmap compares the response with known fingerprints
        ↓
A possible service and version are identified
```

Different services behave differently. Some reveal identifying information as soon as a connection is made, while others respond only after receiving a particular request.

This is why service detection can sometimes recognize software running on an unusual port. If an SSH server is listening on TCP port `2222`, Nmap may still recognize the SSH protocol from the way the service responds.

That is much more reliable than assuming that the port number alone tells us what is running.

### Banners and Fingerprints

Some services reveal identifying information known as a **banner**.

An SSH server, for example, may return something resembling:

```text
SSH-2.0-OpenSSH_8.x
```

A web server may expose information in an HTTP response header such as:

```text
Server: Apache
```

This can be useful, but banners are not always complete or trustworthy. Administrators can suppress or modify them, proxies can sit in front of the real application, and software can deliberately return misleading information.

Nmap therefore does more than just read obvious banners. It can compare service behavior against known fingerprints to make a more informed identification.

The result is still evidence, not absolute truth. If a service version becomes important later, we may need to confirm it using additional methods.

### Focusing on the Ports We Already Found

Once we know which ports are interesting, we do not always need to repeat service detection across every possible port.

If a previous scan found ports 22, 80, and 443, we can focus on those:

```bash
nmap -sV -p 22,80,443 <target-ip>
```

If we found an unusual port such as `8080`, we could investigate only that one:

```bash
nmap -sV -p 8080 <target-ip>
```

This is a good habit because scanning should become more targeted as we learn more about the system.

The first scan helps us discover where to look. The next scan asks a more specific question about what is running there.

That is better than immediately throwing every Nmap option at every port and sorting through a huge amount of output afterward.

### Controlling Version Detection

Nmap’s version detection can be adjusted using **version intensity**.

You may see:

```text
--version-intensity <0-9>
```

Lower values use fewer probes, while higher values allow Nmap to try more probes in an attempt to identify difficult services.

For example:

```bash
nmap -sV --version-intensity 9 <target-ip>
```

There are also shortcuts such as:

```text
--version-light
```

for lighter probing, and:

```text
--version-all
```

for trying all relevant service probes.

For most lab work, normal `-sV` is a good starting point. These options matter because they show that service detection itself has a depth setting. More aggressive identification may produce better results in some cases, but it also means sending more probes.

The goal is not to use the maximum setting automatically. It is to know that the option exists when normal version detection is not giving enough information.

### Understanding Why Nmap Reported a State

The option:

```text
--reason
```

asks Nmap to show why it assigned a particular host or port state.

For example:

```bash
nmap --reason <target-ip>
```

or:

```bash
nmap -sV --reason <target-ip>
```

This can reveal that a port was classified based on a SYN-ACK, a reset, or another observed network response.

That is useful because it reminds us that Nmap results come from network behavior. Instead of treating `open`, `closed`, or `filtered` as labels that appear magically, we can sometimes see the evidence that caused Nmap to assign them.

### Verbose Output

Nmap can also show more information while a scan is running.

The common options are:

```text
-v
```

and:

```text
-vv
```

For example:

```bash
nmap -sV -v <target-ip>
```

Verbose output can be helpful during longer scans because it gives more visibility into what Nmap is doing and what it has discovered so far.

But more output is not automatically better. If a normal scan already answers the question, adding verbosity simply creates more text to read.

Use it when the additional detail helps.

### OS Detection

Nmap can also attempt to identify the target’s operating system with:

```bash
nmap -O <target-ip>
```

This is different from service detection.

`-sV` asks what appears to be running behind open ports.

`-O` tries to determine what operating system the target’s network behavior resembles.

Nmap performs **OS fingerprinting** by sending carefully constructed probes and examining characteristics of the responses. Different operating systems and network stacks can respond differently, and Nmap compares those responses against known fingerprints.

The important word here is **fingerprinting**.

Nmap is not remotely opening the target’s system settings and reading “Windows” or “Linux.” It is making an inference based on network behavior.

That means the result can be uncertain or incorrect. Firewalls, virtualization, unusual configurations, and incomplete responses can all affect OS detection.

If Nmap suggests that a host is likely running Linux, that is useful information. It is not the same as independently confirming that the host is Linux.

### Nmap Scripting Engine

Nmap can do more than basic scanning and fingerprinting. It also includes the **Nmap Scripting Engine**, usually shortened to **NSE**.

NSE scripts extend Nmap with protocol-specific and service-specific functionality. Depending on the script, they can gather additional information, inspect configurations, query services, and perform other checks.

One option you will see frequently is:

```text
-sC
```

which runs Nmap’s default set of NSE scripts.

For example:

```bash
nmap -sC <target-ip>
```

It is also common to combine default scripts with service detection:

```bash
nmap -sC -sV <target-ip>
```

That combination can be useful because Nmap first learns more about the services and then runs applicable default scripts that may return additional information.

NSE is much larger than the few examples we need here, so the important idea is not to memorize script categories. It is to understand that Nmap can perform service-specific enumeration beyond simply asking whether a port is open.

### Running a Specific NSE Script

Individual scripts can also be selected directly:

```bash
nmap --script <script-name> <target-ip>
```

This is usually better than blindly running large groups of scripts.

A sensible workflow is:

**identify the service → decide what you want to learn → choose an appropriate script**

For example, if you know a particular service is running and want additional information about its configuration, you can look through Nmap’s script documentation for a script designed for that purpose.

Some NSE scripts are simple information-gathering scripts, while others can be more intrusive. So `--script all` should not be treated as a harmless “show me everything” option.

Know what a script does before running it, and only use it against systems you are authorized to test.

### What `-A` Actually Enables

Another command that appears constantly in Nmap tutorials is:

```bash
nmap -A <target-ip>
```

`-A` enables several features together, including:

- OS detection
- Service/version detection
- Script scanning
- Traceroute

This can be convenient when that combination is exactly what we want.

But `-A` does not mean “make Nmap advanced” or “scan better.” It simply turns on several features at once.

If we only need service detection, then:

```bash
nmap -sV <target-ip>
```

is clearer.

If we only want OS fingerprinting, then:

```bash
nmap -O <target-ip>
```

expresses that directly.

Understanding the individual features first makes combined options like `-A` much easier to use intentionally.

### Saving Scan Results

Once scans become part of a real lab or investigation, keeping everything only in terminal history becomes inconvenient.

Nmap provides several output formats.

For normal human-readable output:

```bash
nmap -sV -oN scan.txt <target-ip>
```

For XML:

```bash
nmap -sV -oX scan.xml <target-ip>
```

XML becomes useful when scan results need to be processed by another tool or script.

There is also:

```bash
nmap -sV -oA lab-scan <target-ip>
```

`-oA` saves the scan in Nmap’s major output formats using the same base filename.

Saving results is useful even in a home lab. If you scan the same system later and something has changed, you can compare the results rather than relying on memory.

A new port may have appeared, a service may have disappeared, or a version may have changed.

That turns a scan from a temporary terminal output into an artifact you can actually analyze.

### Reading Richer Results

Suppose an authorized scan returns:

```text
PORT       STATE    SERVICE    VERSION
22/tcp     open     ssh        OpenSSH 8.x
80/tcp     open     http       Apache httpd 2.4.x
3306/tcp   open     mysql      MySQL
```

There is much more we can reason about now than we could from a basic port scan.

Port 22 appears to expose an SSH service.

Port 80 appears to expose a web server.

Port 3306 appears to expose a MySQL database service.

That third result is especially interesting because databases are often intended to be reachable only from specific application servers or internal networks. Whether this exposure is appropriate depends entirely on the environment.

That is the kind of question enumeration helps us ask.

We are moving from:

```text
3306/tcp open
```

to:

```text
A MySQL service appears reachable on TCP port 3306 from my scanning position.
```

That is a much more meaningful description of the system’s network exposure.

### Version Information Is Not a Vulnerability Finding

Once a version number appears, it is very tempting to search it and immediately attach vulnerabilities to the system.

Suppose Nmap identifies an OpenSSH version.

That gives us useful information for further research, but it does not automatically prove that every vulnerability associated with that version affects the machine.

Actual vulnerability exposure may depend on the exact package build, operating system, configuration, enabled features, vendor patches, and other conditions.

Linux distributions may also backport security fixes while retaining version strings that appear older than the upstream software release.

So service/version detection gives us a **lead**.

It does not replace vulnerability analysis.

That distinction becomes increasingly important once we start working with CVEs and vulnerability assessment later.

### A Practical Enumeration Workflow

Suppose our authorized target is:

```text
192.168.56.20
```

We might start with a normal scan:

```bash
nmap 192.168.56.20
```

Imagine that returns:

```text
22/tcp   open   ssh
80/tcp   open   http
```

Now we know where to focus.

Instead of running another huge scan, we can ask specifically for service/version information on those ports:

```bash
nmap -sV -p 22,80 192.168.56.20
```

If we want the default NSE scripts as well:

```bash
nmap -sC -sV -p 22,80 192.168.56.20
```

If we are documenting the lab and want to save the output:

```bash
nmap -sC -sV -p 22,80 -oA target-enum 192.168.56.20
```

The command becomes more detailed because our question has become more detailed.

That is the important part.

We are not starting with a giant Nmap command because it looks powerful. We are discovering information, noticing what is missing, and adding options that help answer the next question.

### When Services Run on Unexpected Ports

Consider this:

```text
2222/tcp   open   ssh   OpenSSH 8.x
```

If we relied only on common port associations, port `2222` might not immediately tell us much.

Service detection gives us a better clue. The software responding there behaves like an SSH service.

This is one of the clearest reasons enumeration matters. The actual service response is more useful than simply guessing based on the port number.

The opposite can also happen. A service listening on port 80 does not have to be a normal web server just because port 80 is commonly associated with HTTP.

Port numbers are useful context, but service behavior provides stronger evidence.

### When Nmap Cannot Identify the Service

Not every scan will return a neat product and version.

Sometimes Nmap may return a generic service name, multiple possible matches, or something it cannot identify confidently.

That is not necessarily a failed scan.

It means the service did not provide enough information for Nmap to reach a confident fingerprint match.

At that point, we may need another tool or a more protocol-specific method of enumeration.

A web service might need to be examined with a browser or HTTP tools. An SSH service might reveal useful information through its connection behavior. Other protocols may require their own clients or queries.

One tool rarely gives us every answer.

### Discovery, Enumeration, and Vulnerability Analysis

These stages often get mixed together because tutorials sometimes run them back-to-back without explaining the difference.

**Discovery** is about finding reachable hosts and ports.

**Enumeration** is about learning what services, applications, versions, users, shares, resources, or other useful information are exposed through those systems.

**Vulnerability analysis** takes that information and asks whether any of the identified software or configurations contain weaknesses that actually apply.

Keeping those stages separate prevents a common mistake:

**open port → service name → CVE → vulnerability**

There are several verification steps missing in that shortcut.

Good enumeration fills in some of those gaps before we move further.

### Try It in Your Own Lab

Use an authorized target that exposes at least one network service.

Begin with:

```bash
nmap <target-ip>
```

Choose one or more open ports from the result and run:

```bash
nmap -sV -p <ports> <target-ip>
```

Compare the two outputs. Look for what additional information Nmap was able to identify and whether the service is running where you expected it.

Then try:

```bash
nmap -sC -sV -p <ports> <target-ip>
```

Pay attention to which additional information comes from the default scripts and which service each result belongs to.

If your environment supports OS detection, you can separately try:

```bash
sudo nmap -O <target-ip>
```

Treat the OS result as fingerprint-based identification rather than unquestionable fact.

Finally, save one of your scans:

```bash
nmap -sC -sV -p <ports> -oA enumeration <target-ip>
```

Now you have an actual scan artifact that can be reviewed and compared later.

### The Nmap Options Used Here

At this point, these options should map to specific purposes:

**`-sV`**  
Service and version detection.

**`--version-intensity <0-9>`**  
Control how extensively Nmap probes services during version detection.

**`--version-light`**  
Use lighter version detection.

**`--version-all`**  
Try all appropriate service probes.

**`--reason`**  
Show why Nmap assigned a host or port state.

**`-v` / `-vv`**  
Increase output verbosity.

**`-O`**  
Attempt operating system fingerprinting.

**`-sC`**  
Run Nmap’s default NSE scripts.

**`--script <name>`**  
Run a selected NSE script.

**`-A`**  
Enable OS detection, version detection, script scanning, and traceroute together.

**`-oN`**  
Save normal output.

**`-oX`**  
Save XML output.

**`-oA`**  
Save the major Nmap output formats using one base filename.

There is no need to memorize all of them at once. What matters is knowing which option answers which question.

### The Next Layer Is the Transport Protocol

Service enumeration can tell us far more than a basic port scan, but there is still something underneath all of these results that deserves a proper explanation.

When Nmap reports:

```text
22/tcp
```

or we eventually scan:

```text
53/udp
```

the protocol is not just extra text beside the port number.

TCP and UDP behave differently, respond differently to scans, and provide very different levels of feedback when something is open, closed, or filtered.

That difference is especially important when we start scanning UDP, where receiving no response can mean several different things.

So before we go further with network traffic, we need to understand the transport protocols themselves:

**TCP vs UDP: Understanding What Your Scans Are Actually Testing**
