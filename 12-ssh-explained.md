# SSH Explained: Connecting to Your First Remote Machine

Working with Linux does not always mean sitting directly in front of the machine you want to use. In cybersecurity labs, servers, cloud environments, and enterprise systems, the machine you need to investigate or administer may be somewhere else on the network. You still need access to its shell, files, processes, services, and tools, but now that access has to cross a network before you can interact with the operating system.

That is where **SSH**, or **Secure Shell**, comes in. SSH provides a secure way to access and manage another system remotely, and it is one of the most common technologies used for Linux administration. The basic connection command is short, but underneath it SSH handles several important security functions, including encrypted communication, server identification, user authentication, session management, and optional capabilities such as secure file transfer and port forwarding.

This article is part of my cybersecurity series, which builds from core concepts toward practical security work. Each article also stands on its own, so if this is the first one you are reading, you can follow it without going back through the earlier parts. Here, the focus is specifically on SSH: what happens when you connect to another machine, how password and key authentication work, how SSH establishes trust, where its configuration lives, how to troubleshoot connection problems, and what SSH activity looks like from a security perspective.

---

## What SSH Actually Gives You

Suppose you have two Linux machines in a lab. One is the system you are currently using, while the other is a machine you want to access remotely. Without remote access, you would need to interact with that second machine through its own console. SSH allows you to open a shell on it across the network instead.

Conceptually, the connection looks like this:

```text
Your Machine                         Remote Machine
┌─────────────────┐                 ┌─────────────────┐
│                 │                 │                 │
│   SSH Client    │ ───── SSH ───>  │   SSH Server    │
│                 │                 │                 │
└─────────────────┘                 └─────────────────┘
                                             │
                                             v
                                      Remote Shell
```

The machine initiating the connection runs an **SSH client**, while the remote machine runs an **SSH server** that accepts incoming SSH connections. On many Linux systems, both are provided by **OpenSSH**. The client command is normally `ssh`, while the server process is commonly called `sshd`, where the `d` refers to a daemon, or a program that runs in the background and provides a service.

Once the connection is established and authentication succeeds, commands entered through your terminal are executed on the remote machine. Their output travels back through the SSH connection and appears in your terminal. The terminal window may still be displayed on your own computer, but the shell inside that SSH session belongs to the remote system.

---

## Why SSH Exists

Remote command-line access existed before SSH. Protocols such as **Telnet** were historically used to interact with remote systems, but Telnet does not provide the protections expected for modern administrative access because its traffic is not inherently protected against someone observing the network.

SSH was designed to provide remote communication over networks that should not automatically be trusted. Instead of sending the session as readable plaintext, SSH establishes a protected channel between the client and server. Commands, command output, authentication exchanges, file transfers, and forwarded traffic can then travel through that encrypted connection.

SSH also provides mechanisms for authenticating the remote server. This is important because protecting the traffic is only useful if the client can establish that it is communicating with the intended server rather than simply encrypting information to whichever system answered the connection. SSH therefore addresses both the protection of the communication channel and the identities involved in using it.

---

## The SSH Client and SSH Server

Typing an SSH command on one machine does not automatically make another machine accessible. The remote system must have an SSH server installed, the server must be running, and the network must allow the client to reach the address and port where the service is listening.

On Debian-based systems such as Ubuntu, the OpenSSH server package can commonly be installed with:

```bash
sudo apt install openssh-server
```

After installation, you can inspect the service with:

```bash
systemctl status ssh
```

Depending on the Linux distribution, the service may instead be named `sshd`. If the service needs to be started, you can use:

```bash
sudo systemctl start ssh
```

To configure it to start automatically according to the system's service configuration:

```bash
sudo systemctl enable ssh
```

The service state can then be compared with the machine's listening sockets:

```bash
sudo ss -tulpn
```

These commands answer different questions. `systemctl` tells you how the operating system currently sees the SSH service, while `ss` can show whether a process actually created a listening network socket. When troubleshooting SSH, checking both can help distinguish a service-management problem from a network-listening problem.

---

## Making Your First SSH Connection

The basic SSH command follows this structure:

```bash
ssh username@hostname
```

An IP address can be used instead of a hostname:

```bash
ssh username@ip-address
```

For example, suppose an authorized lab machine has the address `192.168.56.10` and provides an account named `student`. You could connect with:

```bash
ssh student@192.168.56.10
```

The username is an important part of the request because SSH is not simply asking to connect to a computer. It is asking the SSH server to create a session associated with a particular account on that computer. Authentication therefore happens in the context of the username you provide.

At a high level, the process looks like this:

```text
SSH Client
    |
    | Connect to SSH service
    v
SSH Server
    |
    | Establish protected connection
    v
Verify Server Identity
    |
    | Authenticate User
    v
Create Remote Session
    |
    v
Remote Shell
```

After authentication succeeds, you receive a shell associated with the remote account. From that point forward, commands entered in that session operate on the remote system until the session is closed.

To leave the remote shell normally:

```bash
exit
```

You can also commonly end an interactive shell with `Ctrl+D`, which sends an end-of-file condition to the shell.

---

## SSH Uses TCP, Usually on Port 22

SSH normally operates over **TCP**, and the conventional SSH server port is:

```text
22
```

Port 22 is a convention rather than a requirement. Administrators can configure an SSH server to listen on another port, which means the client must know where the service is actually available.

If a server listens on port `2222`, for example, specify that port with:

```bash
ssh -p 2222 student@192.168.56.10
```

The SSH client's port option is lowercase `-p`. This detail becomes useful later because `scp`, another OpenSSH utility, traditionally uses uppercase `-P` when specifying a custom SSH port.

The underlying networking concepts have their own places in this series, so there is no need to rebuild them here. For SSH, the practical requirement is straightforward: the client must be able to reach the address and TCP port on which the SSH server is listening.

---

## What Happens During an SSH Connection

An SSH connection involves more than immediately displaying a password prompt. Before user authentication occurs, the client and server establish the SSH transport and negotiate how the session will be protected.

At a high level:

```text
TCP Connection
      |
      v
SSH Protocol Identification
      |
      v
Algorithm Negotiation
      |
      v
Key Exchange
      |
      v
Server Host Authentication
      |
      v
Encrypted Channel Established
      |
      v
User Authentication
      |
      v
SSH Session
```

The client and server first establish a TCP connection and exchange information about the SSH protocol versions and cryptographic algorithms they support. They then perform a key exchange that allows them to derive session keys used to protect the communication. The server also proves possession of its host private key, allowing the client to verify the server's identity.

Only after the protected SSH transport has been established does the server authenticate the user. This separation is important because SSH is solving two identity problems: the client needs to know which server it reached, and the server needs to decide whether the user requesting access should be allowed to log in.

---

## The First Connection and the Host-Key Prompt

The first time you connect to an SSH server, the client may stop before authentication and display a message similar to:

```text
The authenticity of host '192.168.56.10' can't be established.
ED25519 key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

This happens because the server has presented a **host key**, but your SSH client does not yet have a trusted record associating that server with that key. SSH therefore shows you the key's fingerprint and asks whether you want to continue.

If you accept the host key, the client commonly stores information about it in:

```text
~/.ssh/known_hosts
```

On future connections, SSH can compare the key presented by the server against the one it previously recorded. This allows the client to notice if a familiar hostname or address suddenly presents a different cryptographic identity.

The basic process is:

```text
First Connection
       |
       v
Server Presents Host Key
       |
       v
No Previous Record Exists
       |
       v
Client Shows Fingerprint
       |
       v
User Verifies and Accepts
       |
       v
Host Key Is Remembered
```

When there is no stronger external verification mechanism, this model is commonly described as **Trust On First Use**, or TOFU. The first-time prompt is therefore not SSH declaring that the remote server is trustworthy. It is telling you that the client has no previous identity record for that destination and asking you to make the initial trust decision.

In a controlled lab, you may be able to verify the fingerprint directly from the target VM's console. In a production environment, an organization may provide fingerprints or use other trusted methods to verify server identities before users accept them.

---

## Host Keys Identify the Server

SSH host keys belong to the server and allow it to establish a persistent cryptographic identity. On an OpenSSH server, these keys are commonly stored under:

```text
/etc/ssh/
```

You may encounter files such as:

```text
ssh_host_ed25519_key
ssh_host_ed25519_key.pub
```

The file without `.pub` contains the server's private host key and must remain protected. The corresponding `.pub` file contains the public key.

A server can display the fingerprint of a public host key using a command such as:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

If you have trusted access to the server through another path, you can compare this fingerprint with the one shown by the SSH client during the first connection. This gives you a way to verify that the host key being presented across the network actually belongs to the intended machine.

Host keys answer the question **"Which server am I connecting to?"** User authentication answers a different question: **"Which user is trying to access this server?"** Separating those two identities makes the SSH trust model much easier to understand.

---

## When a Known Host Key Changes

Suppose you previously connected to a server and accepted its host key. Later, you connect to the same hostname or address and receive a warning similar to:

```text
WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!
```

The client is warning you because the server is presenting a different host key from the one previously associated with that destination. There are legitimate reasons this can happen. A lab VM may have been deleted and recreated, a server may have been reinstalled, an IP address may have been reassigned, or administrators may have intentionally rotated host keys.

A changed key can also mean that the machine responding is not the machine you previously trusted. For that reason, deleting the old entry should not be the automatic first response. Determine why the identity changed and verify the replacement key through a trusted method.

After confirming that the change is legitimate, a stale known-host entry can be removed with:

```bash
ssh-keygen -R <hostname-or-ip>
```

For example:

```bash
ssh-keygen -R 192.168.56.10
```

The next connection will treat the destination as a new host and ask you to verify its identity again. This situation appears frequently in home labs because virtual machines are rebuilt and addresses are reused, but the warning itself represents an important SSH security control.

---

## Password Authentication

One way an SSH server can authenticate a user is through a password. After connecting with:

```bash
ssh student@192.168.56.10
```

the server may prompt:

```text
student@192.168.56.10's password:
```

The password is used to authenticate the requested account through the protected SSH connection. While entering it, the terminal commonly displays no characters, dots, or asterisks. This is normal behavior for many Linux password prompts and does not mean the keyboard has stopped working.

If authentication succeeds, the server creates a session associated with that user. What the user can do after login is then governed by the account's privileges, group memberships, file permissions, `sudo` configuration, application permissions, and any additional security controls on the system.

Password authentication is simple and remains common, but SSH also supports another authentication mechanism that is widely used for servers, automation, cloud systems, and administrative access: **public-key authentication**.

---

## Public-Key Authentication

SSH public-key authentication uses a pair of mathematically related keys: a **private key** and a **public key**. The public key can be placed on systems where the account should recognize that key as authorized, while the private key remains under the user's control.

Conceptually:

```text
Client Machine
┌───────────────────────────┐
│                           │
│  Private Key              │
│  Keep this protected      │
│                           │
│  Public Key               │───────────────┐
│                           │               │
└───────────────────────────┘               │
                                            v
                                 Remote Machine
                          ┌──────────────────────────┐
                          │ Authorized Public Key    │
                          └──────────────────────────┘
```

During authentication, the client proves that it possesses the appropriate private key. The server checks that proof using the corresponding authorized public key. The private key itself does not need to be sent to the remote server as part of the authentication process.

This is an important distinction from simply storing the same reusable secret on both sides. The remote account needs the public portion required to verify the authentication proof, while the private credential can remain on the client.

---

## Generating an SSH Key Pair

OpenSSH provides `ssh-keygen` for generating SSH keys. A common modern choice is Ed25519:

```bash
ssh-keygen -t ed25519
```

The command asks where the key should be stored and whether you want to protect the private key with a passphrase. If the default location is used, the files commonly appear under:

```text
~/.ssh/
```

For an Ed25519 key pair, you may see:

```text
id_ed25519
id_ed25519.pub
```

These two files serve very different purposes:

```text
id_ed25519       -> private key
id_ed25519.pub   -> public key
```

The public key is designed to be distributed to systems or services where it should authorize you. The private key is the credential that proves your side of the authentication relationship and should remain under your control.

You can display the public key with:

```bash
cat ~/.ssh/id_ed25519.pub
```

The private key should not be copied into Git repositories, screenshots, public documentation, messages, or other locations where unauthorized people could obtain it.

---

## Protecting a Private Key With a Passphrase

When generating an SSH key, `ssh-keygen` can protect the private-key file with a passphrase. Without that protection, someone who obtains a usable copy of the private key may be able to attempt authentication anywhere the corresponding public key is authorized.

A passphrase adds another barrier by protecting the private-key material stored on disk. Someone who steals the file would also need to overcome that protection before being able to use the key normally.

The private-key passphrase and the remote account password are separate credentials. An account password is evaluated by the remote system when password authentication is used. A private-key passphrase protects the local private-key file and may be requested by the SSH client before that key can be used.

Passphrases introduce a usability tradeoff because entering one for every connection can become inconvenient. SSH provides mechanisms such as `ssh-agent` to make passphrase-protected keys practical without simply removing their local protection.

---

## Installing a Public Key on the Remote Account

For public-key authentication to work, the remote account must know which public keys are authorized. OpenSSH commonly stores these keys in:

```text
~/.ssh/authorized_keys
```

If password authentication is already available, a convenient way to install a public key is:

```bash
ssh-copy-id student@192.168.56.10
```

This copies the appropriate public key into the remote account's SSH authorization configuration. Afterward, a normal connection:

```bash
ssh student@192.168.56.10
```

can attempt public-key authentication.

The relationship looks like this:

```text
Client Has Private Key
          |
          v
Server Has Matching Public Key
          |
          v
Client Proves Possession
          |
          v
Server Verifies Proof
          |
          v
Authentication Succeeds
```

The private key remains on the client. Only the public key needs to be installed on the remote account.

If the SSH files are configured manually, their permissions and ownership also matter. OpenSSH commonly expects the `.ssh` directory and `authorized_keys` file not to be writable by inappropriate users. Typical permissions are:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

If the files are owned by the wrong account or have unsafe permissions, the SSH server may refuse to use them.

---

## Using a Specific Private Key

SSH automatically checks several conventional identity locations, but you may have multiple keys or use filenames that do not follow the defaults. The `-i` option allows you to specify which identity file should be used.

For example:

```bash
ssh -i ~/.ssh/lab_key student@192.168.56.10
```

A user working across several environments might have:

```text
~/.ssh/
├── personal_key
├── personal_key.pub
├── lab_key
├── lab_key.pub
├── work_key
└── work_key.pub
```

Using `-i` makes the intended identity explicit instead of depending on which keys the client happens to discover or offer automatically.

Private-key permissions also matter locally. OpenSSH may refuse to use a private key that is readable by other users because that file represents an authentication credential. A common restrictive setting is:

```bash
chmod 600 ~/.ssh/lab_key
```

If SSH reports that a private-key file has permissions that are too open, changing the permissions should be done because you understand which file needs protection, not simply because a copied troubleshooting command happened to contain `chmod 600`.

---

## Managing Keys With `ssh-agent`

A passphrase-protected private key provides additional protection on disk, but repeatedly entering the passphrase can become inconvenient. `ssh-agent` can hold usable key identities for a user's session so that the key can be reused without entering the passphrase for every individual connection.

You can inspect identities currently available to the agent with:

```bash
ssh-add -l
```

A key can be added with:

```bash
ssh-add ~/.ssh/id_ed25519
```

If the key is passphrase-protected, you will normally be prompted for that passphrase when adding it. The agent can then make the unlocked identity available to SSH clients operating within the appropriate session.

This improves usability without requiring the private-key file itself to be stored unencrypted. It does not remove the need to protect the user's session, however, because access to an active agent may itself become valuable to an attacker who has already compromised the user's environment.

---

## SSH Client Configuration

Typing a long SSH command repeatedly becomes inconvenient when a server uses a custom username, port, hostname, or private key. OpenSSH supports a per-user client configuration file commonly located at:

```text
~/.ssh/config
```

Suppose you normally connect using:

```bash
ssh -i ~/.ssh/lab_key -p 2222 student@192.168.56.10
```

You could create an entry such as:

```text
Host lab-server
    HostName 192.168.56.10
    User student
    Port 2222
    IdentityFile ~/.ssh/lab_key
```

Afterward, the same connection can be initiated with:

```bash
ssh lab-server
```

`Host` defines the alias you type, `HostName` identifies the actual destination, `User` selects the remote account, `Port` specifies the SSH server port, and `IdentityFile` points to the private key.

This becomes particularly useful when you regularly access several machines. Instead of reconstructing connection parameters each time, the SSH client can apply the appropriate configuration based on the host alias.

---

## Client Configuration and Server Configuration

SSH configuration exists on both sides of the connection, but similarly named files perform different roles. A user's client configuration commonly lives at:

```text
~/.ssh/config
```

and controls how that user's SSH client connects to other systems.

System-wide OpenSSH client configuration is commonly stored in:

```text
/etc/ssh/ssh_config
```

The OpenSSH server commonly uses:

```text
/etc/ssh/sshd_config
```

The distinction can be summarized as:

```text
ssh_config   -> SSH client behavior
sshd_config  -> SSH server behavior
```

Modern installations may also load additional configuration fragments, so the effective server configuration can involve more than one file. When editing server configuration, it is useful to validate the syntax before restarting the service.

On many OpenSSH installations:

```bash
sudo sshd -t
```

checks the server configuration for syntax errors. If validation succeeds, the command normally exits without displaying an error. This is particularly important on remote systems because breaking the SSH service can remove the access method you were using to administer the machine.

---

## Important SSH Server Controls

The OpenSSH server configuration contains many directives, and defaults can vary between distributions and versions. Rather than treating one copied configuration as universally correct, it is more useful to understand what the major controls are responsible for.

The listening port can be configured with:

```text
Port 22
```

Password authentication can be controlled with:

```text
PasswordAuthentication yes
```

or:

```text
PasswordAuthentication no
```

Public-key authentication can be controlled with:

```text
PubkeyAuthentication yes
```

Direct root login behavior is governed by:

```text
PermitRootLogin
```

and the possible values determine under which circumstances, if any, root authentication is accepted.

SSH can also restrict which accounts or groups are allowed to connect using directives such as:

```text
AllowUsers
```

and:

```text
AllowGroups
```

The appropriate values depend on how the system is intended to be administered. For example, disabling password authentication before verifying that authorized administrators can successfully use another authentication method could lock legitimate users out. Configuration should therefore follow the access design of the environment rather than being applied as a collection of unexplained hardening commands.

---

## Key Exchange, Host Keys, and User Keys Are Different

SSH uses several types of cryptographic keys during a connection, and their similar terminology can make them easy to confuse. They do not all perform the same job.

**Key exchange** allows the client and server to establish shared session secrets used to protect the connection. Depending on the negotiated algorithms, this may involve mechanisms based on Diffie-Hellman or elliptic-curve cryptography.

**Host keys** belong to the SSH server and are used to authenticate the server's identity to the client. These are the keys associated with fingerprints and `known_hosts`.

**User key pairs** are credentials that can be used to authenticate a user to the server. The client keeps the private key, while the remote account authorizes the corresponding public key.

Conceptually:

```text
Key Exchange
    |
    └── Establish protected session keys

Host Key
    |
    └── Authenticate the SSH server

User Key Pair
    |
    └── Authenticate the user
```

Keeping these roles separate makes SSH's cryptographic design much easier to follow. Saying that SSH "uses keys" is technically true, but it hides the fact that different keys solve different security problems during the same connection.

---

## Algorithm Negotiation

SSH provides a protocol for secure remote communication, but actual connections depend on cryptographic algorithms supported by both the client and server. During connection establishment, the two sides negotiate choices for areas such as key exchange, server host-key authentication, encryption, and integrity protection.

Modern OpenSSH versions disable or discourage a number of obsolete choices by default. Older servers, network appliances, or legacy lab systems may still depend on algorithms that current clients no longer accept automatically. This can result in errors indicating that the client and server could not agree on a particular algorithm.

The solution should not automatically be to enable every old algorithm globally. First identify what the remote system requires and why. If compatibility with a legacy system is genuinely necessary in an authorized environment, any exception should be scoped as narrowly as possible rather than weakening SSH behavior for every connection made by the client.

---

## Troubleshooting With Verbose Mode

A normal SSH command hides most of what happens during connection establishment. When something fails, verbose mode can expose those stages and show where the process stopped.

Start with:

```bash
ssh -v student@192.168.56.10
```

For additional detail:

```bash
ssh -vv student@192.168.56.10
```

and:

```bash
ssh -vvv student@192.168.56.10
```

Verbose output can show which configuration files were processed, which destination and port were selected, which host-key algorithms were negotiated, which keys the client attempted to use, which authentication methods the server offered, and where the connection eventually failed.

This is much more useful than repeatedly running the same command while changing unrelated settings. SSH failures can occur during network connection, host verification, algorithm negotiation, or user authentication, and verbose output helps separate those stages.

---

## Understanding Common SSH Errors

SSH error messages often reveal how far the connection progressed. If you see:

```text
Connection refused
```

the client could not establish the expected connection to a listening SSH service at that destination and port. The server may not be running SSH, it may be listening somewhere else, or a device may be actively rejecting the connection.

If you see:

```text
Connection timed out
```

the connection attempt did not receive the response required to complete within the timeout period. Routing, filtering, host availability, or another network-path problem may be involved.

An error such as:

```text
Permission denied (publickey)
```

occurs later in the process. The client reached an SSH server and attempted authentication, but the server did not accept the public-key authentication offered for that account.

You may instead see:

```text
Permission denied (publickey,password)
```

which indicates that multiple authentication methods were available but none of the attempts resulted in successful authentication.

A host-key warning belongs to a different stage again because it concerns server identity rather than user credentials. Reading the exact message therefore helps narrow the problem before you start changing the server or client configuration.

---

## Troubleshooting Public-Key Authentication

A successful public-key login depends on several pieces agreeing. The client needs the correct private key, the remote account needs the matching public key authorized, the SSH files need appropriate ownership and permissions, and the server must permit that authentication method.

On the client, you can explicitly select the intended key:

```bash
ssh -i ~/.ssh/lab_key student@192.168.56.10
```

If authentication still fails, verbose mode can reveal which identities are actually being offered:

```bash
ssh -vvv -i ~/.ssh/lab_key student@192.168.56.10
```

On the remote account, inspect the SSH directory:

```bash
ls -la ~/.ssh
```

and verify the authorized-key file:

```bash
cat ~/.ssh/authorized_keys
```

Typical permissions are commonly:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

Ownership should also belong to the intended account.

Server-side events can provide another source of information. On systems using systemd, you may be able to inspect the SSH service with:

```bash
journalctl -u ssh
```

or:

```bash
journalctl -u sshd
```

depending on the distribution. Instead of changing several settings simultaneously, determine whether the failure occurs before reaching the server, during server verification, or during user authentication. That keeps troubleshooting tied to the part of SSH that is actually failing.

---

## Running a Remote Command Without Opening a Shell

SSH does not require you to start an interactive shell. A command can be supplied directly after the destination:

```bash
ssh student@192.168.56.10 hostname
```

The client connects, authenticates, asks the remote system to execute `hostname`, returns the output, and then closes the session.

Another example is:

```bash
ssh student@192.168.56.10 "df -h"
```

This capability makes SSH useful for automation because scripts and administrative tools can perform specific operations on remote machines without requiring a person to interact with a shell manually.

More complicated commands require attention to quoting because your local shell processes part of the command before SSH runs. Depending on the syntax, you may need to determine whether variables, wildcards, pipes, or other shell operators should be interpreted locally or by the remote shell.

---

## Secure File Transfer With `scp`

SSH is not limited to interactive sessions and remote commands. OpenSSH also provides utilities for transferring files through SSH-protected connections.

One familiar utility is:

```bash
scp
```

To copy a local file to a remote machine:

```bash
scp report.txt student@192.168.56.10:/home/student/
```

To copy a file from the remote machine into your current local directory:

```bash
scp student@192.168.56.10:/home/student/report.txt .
```

Directories can be copied recursively with:

```bash
scp -r evidence/ student@192.168.56.10:/home/student/
```

If the SSH server uses a custom port, `scp` traditionally specifies it with uppercase `-P`:

```bash
scp -P 2222 report.txt student@192.168.56.10:/home/student/
```

This differs from the lowercase `-p` used by the `ssh` client for its destination port. Related OpenSSH tools share the same underlying ecosystem, but their command-line options are not always identical, so checking their documentation is preferable to assuming that every flag works the same way.

---

## SFTP

Another file-transfer mechanism available through SSH is **SFTP**, the SSH File Transfer Protocol.

An interactive session can be started with:

```bash
sftp student@192.168.56.10
```

Inside SFTP, commands such as `ls`, `pwd`, `get`, and `put` allow you to navigate and transfer files. For example:

```text
get report.txt
```

downloads a remote file, while:

```text
put evidence.txt
```

uploads a local file.

Despite the similar name, SFTP is not simply traditional FTP with encryption added to it. SFTP is a separate file-transfer protocol designed to operate through SSH. FTP, FTPS, and SFTP therefore should not be treated as interchangeable names even though all three can be encountered in file-transfer environments.

---

## Local Port Forwarding

SSH can carry network connections in addition to shells, commands, and files. One of the most useful mechanisms is **local port forwarding**, created with `-L`.

Suppose an HTTP service is available from the SSH server at `127.0.0.1:80`, but that service is not directly reachable from your machine. You could create a local forward with:

```bash
ssh -L 8080:127.0.0.1:80 student@192.168.56.10
```

The connection path can be visualized as:

```text
Your Machine
127.0.0.1:8080
       |
       v
┌──────────────────┐
│    SSH Client    │
└────────┬─────────┘
         │
         │ Encrypted SSH Connection
         v
┌──────────────────┐
│    SSH Server    │
└────────┬─────────┘
         │
         v
127.0.0.1:80
from the server's perspective
```

A connection made to port `8080` on your local machine is carried through the SSH connection. The SSH server side then connects to `127.0.0.1:80` from its own perspective.

That final detail is important. In this command, the destination `127.0.0.1` refers to loopback as seen by the remote side of the tunnel, not your local computer. SSH forwarding becomes much easier to understand when every address is interpreted from the machine that actually makes that part of the connection.

---

## Remote Port Forwarding

SSH can create a **remote port forward** using `-R`. In this case, the listening side of the forwarding arrangement is created from the remote side, and connections arriving there can be carried through the SSH connection toward a destination reachable from the client side.

The general form is:

```bash
ssh -R remote_port:destination_host:destination_port user@ssh-server
```

The exact interface on which the remote listening socket is available depends on the forwarding command and SSH server configuration. Settings such as `GatewayPorts` can affect whether a remotely forwarded socket is restricted to loopback or made available more broadly.

Remote forwarding is useful in legitimate administration, development, testing, and support environments where a service available on the client side needs to become reachable from the remote side. From a security perspective, it is also worth recognizing because an SSH connection can create a network path that would not exist through normal direct routing.

---

## Dynamic Port Forwarding

SSH also supports **dynamic forwarding** with `-D`. For example:

```bash
ssh -D 1080 student@192.168.56.10
```

This commonly creates a SOCKS proxy endpoint on the local machine. Applications configured to use that proxy can send supported connections through the SSH tunnel, with the remote SSH side acting as the point from which those connections continue.

The three common forwarding forms therefore solve different problems:

```text
-L   Local port forwarding
-R   Remote port forwarding
-D   Dynamic SOCKS forwarding
```

Port forwarding is used legitimately in administration, development, segmented networks, and authorized security testing, but it changes how traffic can move between systems. Creating tunnels should therefore be limited to environments and systems where you have permission to establish those paths.

---

## SSH Is More Than a Remote Terminal

SSH is often introduced as a way to obtain a remote shell, but an SSH connection can support multiple channels and functions. The same protected connection can carry an interactive terminal, execute a single command, transfer files, or forward other network connections.

This is why SSH appears underneath many workflows that do not look like someone manually administering a Linux server. Deployment systems, backup tools, configuration-management platforms, development environments, Git operations, automation scripts, and infrastructure tooling may all use SSH.

Seeing an SSH connection on a network therefore tells you which protocol is carrying the communication, but it does not by itself tell you exactly what the user or application is doing inside that connection. Encryption protects much of the session content from ordinary network observation, while endpoint logs and system activity may provide additional information about how the connection was used.

---

## SSH and Git

Git hosting services can use SSH for authenticated repository access. An SSH-based Git remote commonly looks like:

```text
git@github.com:username/repository.git
```

rather than an HTTPS URL.

The underlying authentication uses the same basic public-key concept discussed earlier. Your machine holds the private key, while the service associates the corresponding public key with an authorized account. Git can then invoke SSH underneath the repository workflow when it needs to communicate with the remote service.

You can inspect the remotes configured for a repository with:

```bash
git remote -v
```

This is a useful example of SSH operating underneath another tool. You may never manually run `ssh` during a normal `git push`, but SSH can still provide the authenticated connection used to communicate with the repository host.

---

## SSH Logs

Because SSH frequently provides administrative access, its authentication and session events are useful system records. Their exact location depends on the Linux distribution and logging configuration.

On some Debian-based systems, authentication-related information may appear in:

```text
/var/log/auth.log
```

On some Red Hat-derived systems, you may encounter:

```text
/var/log/secure
```

Systems using systemd may also expose SSH service events through:

```bash
journalctl -u ssh
```

or:

```bash
journalctl -u sshd
```

SSH-related records can include successful and failed authentication attempts, usernames, source addresses, authentication methods, session creation, and session closure. These records can later be correlated with other activity on the system to understand what happened after remote access occurred.

Logs have their own article later in this series, so the goal here is not to perform a complete log investigation. The SSH-specific point is that remote access leaves endpoint evidence that can help explain who attempted to connect, whether authentication succeeded, and how sessions were created.

---

## SSH Brute-Force and Password Attacks

Internet-accessible SSH services commonly receive automated authentication attempts. Attackers can scan address ranges for reachable SSH servers and attempt common usernames, guessed passwords, reused credentials, or credentials obtained elsewhere.

A server may therefore contain many failed SSH authentication records. Those failures show that authentication attempts occurred and were rejected. They do not show that the attacker successfully entered the system.

If an investigation begins with repeated failed SSH logins, the next questions would include whether any attempt later succeeded, which account was involved, what authentication method was accepted, where the successful connection originated, and what occurred after the session began. This is where SSH authentication evidence starts connecting with broader host investigation.

---

## Passwords and Keys Solve Authentication Differently

Password authentication and public-key authentication have different security properties. Weak, reused, or compromised passwords can be guessed, sprayed, phished, or reused against exposed SSH services. Public-key authentication avoids presenting a reusable account password as the SSH login credential, but it introduces a different set of responsibilities.

Private keys must be protected, authorized public keys need to be removed when they should no longer grant access, and old or forgotten keys can create persistent authorization that administrators may overlook. Automation keys require particular attention because they can remain in use for long periods and may exist across multiple systems.

A secure SSH deployment therefore depends on more than selecting one authentication mechanism. Account management, key management, network restrictions, server configuration, monitoring, patching, and revocation all contribute to how remote access is controlled.

---

## Direct Root Login

The Linux `root` account has extensive privileges, so allowing direct remote authentication to it changes the risk and accountability of SSH access. OpenSSH controls this behavior through the server directive:

```text
PermitRootLogin
```

Different values allow administrators to determine whether root can authenticate and under which circumstances.

Many environments instead require administrators to authenticate using individual user accounts and elevate privileges when necessary. This creates clearer attribution because the initial remote session belongs to a specific user rather than a shared or general root identity.

The appropriate policy depends on the environment, but the configuration should reflect how administrative identity and privilege are intended to work. SSH controls who can establish the remote session, while the operating system determines what that authenticated account can do afterward.

---

## Changing the SSH Port

Administrators sometimes move SSH from its conventional port `22` to another port. This can reduce some automated traffic that blindly targets the default SSH port, but the custom port itself does not provide meaningful authentication.

A scanner that examines multiple ports and identifies services can still discover SSH running on a nonstandard port. The service has changed location, not identity.

Using a custom port can therefore be an operational choice, but it should not replace controls such as appropriate authentication, restricted network access, secure server configuration, patching, and monitoring. The security of SSH should not depend on an attacker failing to discover which port it uses.

---

## Restricting Who Can Reach SSH

SSH authentication determines whether a user can successfully establish a session, but systems can also restrict which clients are allowed to reach the SSH service in the first place. Host firewalls, network firewalls, cloud security controls, VPNs, and network segmentation can all influence whether a connection attempt ever reaches `sshd`.

SSH itself can then apply additional restrictions through users, groups, authentication methods, and other server configuration.

A simplified access path might look like:

```text
Remote Client
      |
      v
Network Access Controls
      |
      v
SSH Server
      |
      v
Server Identity Verification
      |
      v
User Authentication
      |
      v
Linux Account Permissions
```

These layers perform different jobs. Network controls determine which traffic can reach the service, SSH determines whether the remote identity can establish a session, and Linux permissions determine what that account can access after login. Firewalls have their own article later in the series, so we will keep the focus here on how they fit around SSH rather than rebuilding firewall behavior inside this article.

---

## SSH Keys Need a Lifecycle

Generating an SSH key pair is only the beginning of managing it. Over time, keys may need to be added, inventoried, rotated, removed, or replaced after suspected exposure. Public keys that remain authorized after they are no longer needed can create access paths that administrators may forget exist.

Consider an account whose `authorized_keys` file contains several public keys accumulated over years. If nobody knows which person or automation process owns each key, determining who can actually authenticate becomes difficult. Removing a user's password elsewhere does not automatically remove every SSH key that may still authorize that account.

Large environments therefore need more than instructions for running `ssh-keygen`. They need a way to understand which keys exist, who owns them, where they are authorized, and when that authorization should end. This is one reason organizations may use centralized access-management systems, SSH certificates, or other mechanisms rather than manually distributing permanent keys across every server.

---

## What Is Inside `~/.ssh`?

The `.ssh` directory can contain several files involved in SSH authentication, trust, and configuration:

```text
~/.ssh/
├── authorized_keys
├── config
├── id_ed25519
├── id_ed25519.pub
└── known_hosts
```

These files do not all contain the same kind of information. `id_ed25519` is a private authentication key, while `id_ed25519.pub` is its public counterpart. `authorized_keys` lists public keys that may authenticate to the account. `known_hosts` stores information about server identities previously accepted by the client, and `config` can define connection parameters such as usernames, ports, destinations, key paths, and forwarding behavior.

Understanding the directory as a collection of different SSH state makes troubleshooting much easier. A problem with `known_hosts` concerns server identity, while a problem with `authorized_keys` concerns user authorization. A problem with the private key concerns the client's authentication credential, and a problem in `config` may affect which destination or identity the client attempts to use.

---

## SSH From a Security Investigation Perspective

Suppose you are examining a Linux system after suspicious remote activity. Discovering that SSH is installed tells you very little by itself because SSH is a normal administrative service on many Linux systems. The investigation begins when you connect service configuration, exposure, authentication, and session activity.

You might first inspect whether the SSH service is running:

```bash
systemctl status ssh
```

Then determine whether it is listening and where:

```bash
sudo ss -tulpn
```

If you need to understand account authorization, you may inspect the relevant user's SSH directory and `authorized_keys`. Login history can provide another perspective:

```bash
last
```

while SSH service events may be available through:

```bash
journalctl -u ssh
```

Depending on the distribution, traditional authentication logs may provide additional records.

Each source answers a different question. Configuration describes how SSH is intended to operate, sockets show how it is currently exposed on the host, authorization files show which keys may authenticate, and logs or session records describe actual activity. The useful conclusion comes from combining those pieces rather than expecting one command to explain the entire event.

---

## A Practical SSH Troubleshooting Approach

When SSH fails, start by identifying how far the connection progresses rather than immediately modifying the configuration. Verify the destination, username, and port first:

```bash
ssh student@192.168.56.10
```

If the server uses a custom port:

```bash
ssh -p 2222 student@192.168.56.10
```

If you control the server, check whether the SSH service is running:

```bash
systemctl status ssh
```

Then verify whether it actually created the expected listening socket:

```bash
sudo ss -tulpn
```

If the client reaches the server but authentication fails, inspect the connection with verbose output:

```bash
ssh -vvv student@192.168.56.10
```

When a particular private key should be used, make that explicit:

```bash
ssh -i ~/.ssh/lab_key student@192.168.56.10
```

On the server, the account's `.ssh` directory, `authorized_keys`, ownership, and permissions may need to be examined. SSH service events can then provide the server's view of the authentication attempt:

```bash
journalctl -u ssh
```

This approach separates the connection into stages. A network failure, a server-identity warning, and a rejected user credential are three different problems even though all of them can prevent you from obtaining a remote shell.

---

## A Compact SSH Reference

After understanding what the commands are doing, a compact reference is useful when you need to return to SSH later.

|               Task              |                Command                |
|---------------------------------|---------------------------------------|
| Connect to a server             | `ssh user@host`                       |
| Connect to a custom port        | `ssh -p 2222 user@host`               |
| Use a specific private key      | `ssh -i ~/.ssh/key user@host`         |
| Verbose troubleshooting         | `ssh -v user@host`                    |
| More detailed troubleshooting   | `ssh -vvv user@host`                  |
| Run one remote command          | `ssh user@host "command"`             |
| Generate an Ed25519 key         | `ssh-keygen -t ed25519`               |
| Copy a public key               | `ssh-copy-id user@host`               |
| List agent identities           | `ssh-add -l`                          |
| Add a key to the agent          | `ssh-add ~/.ssh/id_ed25519`           |
| Remove a stale known-host entry | `ssh-keygen -R host`                  |
| Copy a file to a remote system  | `scp file user@host:/path/`           |
| Copy a remote file locally      | `scp user@host:/path/file .`          |
| Start an SFTP session           | `sftp user@host`                      |
| Local port forwarding           | `ssh -L local:host:port user@server`  |
| Remote port forwarding          | `ssh -R remote:host:port user@server` |
| Dynamic forwarding              | `ssh -D 1080 user@server`             |
| Check the SSH service           | `systemctl status ssh`                |
| Inspect SSH journal events      | `journalctl -u ssh`                   | 
| Validate server configuration   | `sudo sshd -t`                        |

The table works best as a lookup after the underlying concepts are familiar. Commands such as `ssh-keygen -R`, `ssh -i`, and `ssh -L` are much easier to use correctly when you understand whether you are changing server trust, user authentication, or the network path carried through the SSH connection.

---

## SSH Is More Than `ssh user@host`

The visible part of SSH is often just a terminal command followed by a login prompt, but several different mechanisms are working underneath it. The client and server establish a protected connection, the server presents its cryptographic identity, the client decides whether that identity should be trusted, the server authenticates the requested user, and the operating system applies that account's privileges to the resulting session.

Public-key authentication introduces separate private and public credentials, while `known_hosts` and server host keys maintain a different trust relationship entirely. Configuration files control how clients connect and how servers accept access, logs preserve evidence of authentication and sessions, and SSH features such as file transfer and port forwarding extend the protocol well beyond interactive terminal access.

Once those pieces are separated, SSH becomes much easier to reason about. If a connection fails, you can ask whether the problem occurred while reaching the service, verifying the server, negotiating the connection, or authenticating the user. If you are examining SSH activity on a system, you know that configuration, listening sockets, authorized keys, session records, and service logs provide different parts of the picture.

The next article moves to a service that quietly supports much of what happens on a network. We have already used domain names and commands such as `dig`, but resolving a name involves its own infrastructure, records, trust assumptions, and security concerns.

Next in the series:

**DNS From a Security Perspective**
