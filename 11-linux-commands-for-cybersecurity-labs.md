# Linux Commands You Actually Need for Cybersecurity Labs

Linux shows up almost everywhere once you start doing hands-on cybersecurity. Kali Linux itself is Linux-based, many servers and cloud systems run Linux, security appliances often expose Linux-like environments, and a large number of security tools are designed to be used from a command line. In a lab, you may need to find a file, inspect a process, check a service, understand permissions, search through logs, verify a file hash, identify a listening application, or connect to another machine. Even when Linux is not the subject of the lab, being comfortable with it often determines how easily you can complete the actual security work.

At first, the command line can look like a collection of unrelated commands and flags that need to be memorized. That is not a particularly useful way to learn it. A command becomes much easier to remember when you understand the question it answers. `pwd` tells you where you are, `ls` tells you what is there, `ps` tells you what is running, `ss` tells you about sockets, `grep` helps you find relevant text, and `chmod` changes permissions. Once commands are connected to questions, the terminal starts feeling less like a memory test and more like a way to examine and control a system.

This article is part of my cybersecurity series, where I am building from foundational concepts toward practical security analysis and investigation. Each article also stands on its own, so you do not need to have read the earlier parts to follow this one. Here, the focus is specifically on Linux: how to move through a system, work with files, understand permissions and users, inspect processes and services, examine network configuration, manipulate command output, and use the shell efficiently enough that future security labs do not feel like a collection of unexplained terminal instructions.

---

## The Terminal, Shell, and Commands

When we open a terminal on Linux, the terminal itself is only the interface through which we interact with a command-line environment. The program actually interpreting what we type is called the **shell**. Bash is one of the most common Linux shells, although Zsh, Fish, and others are also widely used. The distinction becomes useful because features such as variables, command history, pipes, redirection, aliases, and scripting are largely shell features rather than properties of an individual security tool.

You can check the shell configured for your account with:

```bash
echo $SHELL
```

A result might look like:

```text
/bin/bash
```

or:

```text
/bin/zsh
```

Most commands follow a general structure:

```text
command [options] [arguments]
```

For example:

```bash
ls -la /etc
```

Here, `ls` is the command, `-la` modifies how the command behaves, and `/etc` tells it which location to examine. Options are sometimes written with a single dash, such as `-l`, and sometimes with longer names beginning with two dashes, such as `--help`. The exact options depend on the command, which is why knowing how to read command documentation is more valuable than trying to memorize every flag you encounter.

Linux is also case-sensitive. `Report.txt`, `report.txt`, and `REPORT.txt` can be three different files, and commands such as `ls` and `LS` are not interchangeable. That distinction becomes important when following instructions, searching for files, working with scripts, or troubleshooting a command that appears correct at first glance.

---

## Understanding Where You Are in Linux

Linux organizes files in a hierarchy beginning at a single location:

```text
/
```

This is the **root directory**, which sits at the top of the filesystem. Everything else exists somewhere underneath it. A typical system contains directories such as `/home` for regular users' home directories, `/etc` for a large amount of system and application configuration, `/var` for changing application and system data, `/tmp` for temporary files, and `/usr` for many installed programs and shared resources.

You will also encounter directories that behave differently from ordinary storage. `/proc`, for example, is a virtual filesystem that exposes information about processes and the kernel, while `/dev` contains special files representing devices. You do not need to memorize the entire Linux filesystem hierarchy before working in a lab. What matters initially is understanding that locations have meaning and being able to determine where you currently are.

The command for that is:

```bash
pwd
```

`pwd` stands for **print working directory**. If it returns:

```text
/home/siri
```

your shell is currently operating from `/home/siri`.

To see the contents of the current directory:

```bash
ls
```

You might see:

```text
Desktop
Documents
Downloads
notes.txt
scan-results.txt
```

A more useful view in many situations is:

```bash
ls -l
```

The `-l` option produces a long listing containing information such as permissions, ownership, size, modification time, and filename.

For example:

```text
-rw-r--r-- 1 siri siri 1842 Sep 2 10:15 scan-results.txt
```

To include hidden files as well:

```bash
ls -la
```

Linux commonly treats names beginning with a dot as hidden. This is why configuration files and directories such as `.bashrc`, `.profile`, and `.ssh` may not appear with a basic `ls` command. Hidden does not mean protected or secret. It simply means these entries are omitted from normal directory listings unless you request them.

---

## Moving Through the Filesystem

The `cd` command changes the current working directory. If a directory called `Documents` exists under your current location, you can enter it with:

```bash
cd Documents
```

Linux paths can be **absolute** or **relative**. An absolute path begins at `/` and describes the complete location:

```text
/home/siri/Documents/report.txt
```

A relative path begins from your current working directory:

```text
Documents/report.txt
```

Several shortcuts make navigation faster. A single dot, `.`, represents the current directory, while `..` represents its parent. The tilde, `~`, represents your home directory. Therefore, `cd ..` moves one directory upward, while `cd ~` returns to your home directory. Running `cd` without an argument normally takes you home as well.

These symbols appear in many commands beyond navigation. If you later see:

```bash
find . -name "*.log"
```

the dot tells `find` to begin searching from the current directory. If you see:

```bash
./script.sh
```

the same dot indicates that `script.sh` should be found in the current directory. Understanding paths early prevents a surprising amount of confusion later because many command-line problems are simply cases of looking in, writing to, or executing from the wrong location.

---

## Creating, Copying, Moving, and Removing Files

Security labs generate files constantly. You may save command output, download scripts, create notes, extract archives, collect packet captures, or organize evidence. Linux provides a small group of commands that handle most everyday file operations.

To create a directory:

```bash
mkdir investigation
```

If you need several nested directories at once:

```bash
mkdir -p lab/evidence/network
```

The `-p` option creates missing parent directories as necessary.

An empty file can be created with:

```bash
touch notes.txt
```

Although `touch` is technically used to update file timestamps, it creates the file if it does not already exist, which makes it convenient for quickly creating an empty file.

To copy a file:

```bash
cp notes.txt notes-backup.txt
```

To copy an entire directory and its contents:

```bash
cp -r evidence evidence-backup
```

Files can be moved with:

```bash
mv report.txt reports/
```

The same command is used for renaming:

```bash
mv old-report.txt final-report.txt
```

Removing files deserves more attention because command-line deletion often does not behave like moving something to a graphical recycle bin. A file can be removed with:

```bash
rm notes.txt
```

An empty directory can be removed with:

```bash
rmdir empty-directory
```

A directory and its contents can be removed recursively with:

```bash
rm -r directory
```

Before using recursive deletion, especially with elevated privileges, verify both your current location and the target path. Commands such as `pwd` and `ls` take seconds to run and can prevent deleting the wrong directory. Linux generally assumes that if you issued a valid destructive command, you intended to do it.

---

## Identifying Files Before Opening Them

A filename does not necessarily tell you what the file contains. Something called `report.txt` could have been renamed from another format, and files encountered during security analysis may have no extension at all. Linux provides the `file` command to examine a file and estimate its type based on its contents.

For example:

```bash
file suspicious_file
```

The result could be something like:

```text
suspicious_file: ELF 64-bit LSB executable
```

or:

```text
capture.pcap: pcap capture file
```

For filesystem metadata rather than content type, use:

```bash
stat suspicious_file
```

`stat` can show the file's size, inode, ownership, permissions, and several timestamps. These commands answer different questions. `file` helps determine what kind of data you are dealing with, while `stat` tells you how the filesystem describes that object.

This distinction becomes useful when you encounter unfamiliar artifacts. Before executing or opening something with an application, identifying its type and examining its metadata can give you a much better idea of what you are actually working with.

---

## Reading Files From the Terminal

A large amount of information on Linux is stored as text, including configuration files, scripts, application output, and many logs. The simplest way to display a short text file is:

```bash
cat filename.txt
```

For a small file, `cat` works well. For something containing thousands of lines, however, dumping everything into the terminal is not particularly useful. The `less` command allows you to move through the content interactively:

```bash
less filename.txt
```

Inside `less`, you can scroll through the file, press `/` to search for text, and press `q` to exit. If you only need the beginning of a file, `head` is more convenient:

```bash
head filename.txt
```

By default, it displays the first ten lines. You can request a specific number with:

```bash
head -n 20 filename.txt
```

The corresponding command for the end of a file is:

```bash
tail filename.txt
```

or:

```bash
tail -n 20 filename.txt
```

One particularly useful variation is:

```bash
tail -f application.log
```

The `-f` option follows the file as new lines are added. If you are testing a service while watching its log, this allows you to observe new entries without repeatedly reopening the file. Deep log analysis belongs later in this series, but knowing how to locate, read, and follow text files is part of being comfortable inside Linux.

---

## Finding Files When You Do Not Know Their Location

Eventually, you will know what you need but not where it is. The `find` command searches the filesystem according to conditions you specify.

To search the current directory and everything below it for a file named `config.txt`:

```bash
find . -name "config.txt"
```

To search your home directory:

```bash
find ~ -name "config.txt"
```

Patterns can be used when you do not know the complete filename:

```bash
find . -name "*.log"
```

The `*` wildcard represents any sequence of characters, so this searches for names ending in `.log`. Filename matching with `-name` is case-sensitive. For case-insensitive matching:

```bash
find . -iname "*.log"
```

`find` can search by much more than filename. To return only regular files:

```bash
find . -type f
```

To return directories:

```bash
find . -type d
```

It can also search using properties such as ownership, permissions, size, and modification time. For example:

```bash
find /var/log -type f -mtime -1
```

searches under `/var/log` for regular files modified within roughly the last day.

This is where `find` becomes much more than a replacement for a graphical search box. During system analysis, the question may be "Which files changed recently?" or "Where are files with a particular permission?" rather than "Where is this exact filename?"

---

## Searching Inside Files With `grep`

Once you find the right file, the next problem may be finding the right information inside it. A configuration file may contain hundreds of lines, while a log can contain hundreds of thousands. `grep` searches text for matching patterns and is one of the commands you will use repeatedly in Linux security work.

Suppose `application.log` contains many events and you want lines containing `ERROR`:

```bash
grep "ERROR" application.log
```

To make the search case-insensitive:

```bash
grep -i "error" application.log
```

To include line numbers:

```bash
grep -n "error" application.log
```

To search recursively through files beneath a directory:

```bash
grep -R "password" .
```

You can also invert the search:

```bash
grep -v "INFO" application.log
```

This returns lines that do not contain `INFO`.

`grep` also supports regular expressions, which allow you to describe patterns rather than exact strings. For example:

```bash
grep -E "error|failed|denied" application.log
```

searches for several alternatives. Regular expressions can become a substantial topic by themselves, so there is no reason to master complex patterns immediately. The useful starting point is understanding that `grep` can reduce a large body of text to the lines related to the question you are investigating.

---

## Pipes: Connecting Commands Together

Individual Linux commands are intentionally small, and much of their power comes from combining them. The pipe character, `|`, takes the standard output produced by one command and provides it as input to another.

Suppose:

```bash
ps aux
```

produces a long process listing, but you only care about entries containing `ssh`. You can connect the output directly to `grep`:

```bash
ps aux | grep ssh
```

Conceptually, the data flows like this:

```text
ps aux
   |
   | process listing
   v
grep ssh
   |
   v
matching lines
```

The same idea can be extended. Suppose a file contains authentication events and you want to know how many lines contain the word `Failed`:

```bash
grep "Failed" auth.log | wc -l
```

`grep` extracts the relevant lines, then `wc -l` counts them. Neither command individually answers the complete question, but together they do.

This composability is one of the most important ideas in the Linux command line. Instead of expecting one enormous program to perform every possible task, Linux provides many focused tools whose input and output can be connected.

---

## Standard Input, Standard Output, and Standard Error

Pipes make more sense once we understand the streams Linux programs commonly use. A process normally begins with three standard file descriptors:

```text
0 = standard input  (stdin)
1 = standard output (stdout)
2 = standard error  (stderr)
```

Standard input is where a program receives ordinary input. Standard output is where normal results are written, while standard error provides a separate stream for error messages. In an interactive terminal, both output streams usually appear on the screen, which can make them look identical even though the operating system treats them separately.

This separation allows the shell to redirect information precisely. For example, normal command output can be written to a file:

```bash
ip addr > network-info.txt
```

The `>` operator replaces the file's existing contents if it already exists. To append instead:

```bash
ip addr >> network-info.txt
```

Errors can be redirected independently:

```bash
command 2> errors.txt
```

If you want both normal output and errors written to the same destination:

```bash
command > output.txt 2>&1
```

The `2>&1` syntax tells the shell to send file descriptor `2`, standard error, to the same destination currently used by file descriptor `1`, standard output. Understanding this becomes useful when saving tool output, troubleshooting scripts, or collecting command results for later review.

---

## Using `tee` When You Want to See and Save Output

Normal redirection creates one inconvenience: when output is redirected to a file, you no longer see it in the terminal in the usual way. `tee` solves that by displaying the information while also writing it to a file.

For example:

```bash
command | tee results.txt
```

The output remains visible and is saved to `results.txt`.

To append rather than overwrite:

```bash
command | tee -a results.txt
```

This is particularly convenient when documenting lab work. You can observe a tool normally while preserving its output for later analysis instead of rerunning the command or copying text manually.

---

## Working With Large Amounts of Text

Once command output starts becoming large, a few small text-processing utilities become extremely useful. `wc` can count lines, words, or bytes, with line counting being particularly common:

```bash
wc -l file.txt
```

`sort` arranges text:

```bash
sort names.txt
```

`uniq` identifies adjacent duplicate lines, which means it is commonly paired with `sort`:

```bash
sort names.txt | uniq
```

To count how often each value appears:

```bash
sort names.txt | uniq -c
```

If you had extracted a list of usernames, hostnames, or IP addresses into a file, this combination could quickly show which values occur repeatedly.

Another useful utility is `cut`, which extracts fields from structured text. If values are separated by colons, for example:

```bash
cut -d ':' -f 1 filename
```

the `-d` option defines `:` as the delimiter and `-f 1` selects the first field.

For more complicated processing, Linux provides tools such as `awk` and `sed`, while structured formats may be easier to handle with tools such as `jq` or with Python. You do not need to learn all of them at once. The broader skill is recognizing when raw output can be filtered or transformed instead of manually reading every line.

---

## Wildcards, Quoting, and Shell Expansion

Before a command runs, the shell may interpret parts of what you typed. Wildcards are a common example. The asterisk `*` can represent any sequence of characters, so:

```bash
ls *.txt
```

matches filenames ending in `.txt`.

The question mark matches a single character:

```bash
ls file?.txt
```

This could match `file1.txt` or `fileA.txt`, but not `file10.txt`.

Quoting determines how the shell interprets special characters and spaces. Suppose a filename is:

```text
security notes.txt
```

Running:

```bash
cat security notes.txt
```

causes the shell to pass two separate arguments. Quoting keeps the filename together:

```bash
cat "security notes.txt"
```

Single and double quotes also differ when variables are involved. If:

```bash
name="siri"
```

then:

```bash
echo "$name"
```

expands the variable and prints its value, while:

```bash
echo '$name'
```

prints the literal characters `$name`.

These differences become especially important when commands contain paths, variables, wildcard characters, or data received from another source. A command that looks visually correct can behave very differently depending on how the shell expands it before execution.

---

## Running Commands Based on Success or Failure

The shell also allows commands to be chained according to whether earlier commands succeeded. A semicolon simply runs commands sequentially:

```bash
pwd; ls
```

The second command runs regardless of the result of the first.

With `&&`, the second command runs only if the first succeeds:

```bash
mkdir results && cd results
```

This is useful because entering a directory only makes sense if creating it succeeded.

The opposite behavior is available with `||`:

```bash
command1 || command2
```

Here, `command2` runs if `command1` does not succeed.

These operators depend on a command's **exit status**. After a program finishes, you can inspect its status with:

```bash
echo $?
```

By convention, an exit status of `0` means success, while non-zero values indicate another result or some type of failure. The exact meanings of non-zero values depend on the program.

This becomes increasingly important when commands are automated. A person can look at the screen and interpret an error message, but a script needs a machine-readable way to decide whether the previous operation succeeded before continuing.

---

## Users, Groups, and Identity

Linux is designed as a multi-user operating system, so identity is built into how files, processes, and permissions work. Before investigating access or ownership, you often need to know which identity your current shell is using.

The simplest command is:

```bash
whoami
```

For more information:

```bash
id
```

Example output might look like:

```text
uid=1000(siri) gid=1000(siri) groups=1000(siri),27(sudo)
```

Linux uses numeric user IDs and group IDs internally, even though usernames and group names are normally displayed for humans. The output above therefore tells us the current user's UID, primary group, and additional group memberships.

Basic user account information is traditionally stored in:

```text
/etc/passwd
```

A line might look like:

```text
siri:x:1000:1000:Siri:/home/siri:/bin/bash
```

The fields include the username, numeric identifiers, home directory, and configured login shell. Despite the filename, modern Linux systems do not normally store password hashes directly in `/etc/passwd`; password-related hashes are typically stored in `/etc/shadow`, which has much stricter access permissions. Group information is maintained separately in `/etc/group`.

To see currently logged-in users, Linux also provides:

```bash
who
```

and:

```bash
w
```

The `w` command provides additional information about logged-in sessions and activity. Historical login information may be available through:

```bash
last
```

depending on the system's configuration and retained records. Together, these commands let you distinguish between which accounts exist, which identity you are currently using, and which users have active or recorded sessions.

---

## Root, `sudo`, and Privileged Commands

Linux has a highly privileged account named `root`. Root can perform operations that ordinary users cannot, including modifying many protected system files, managing services, changing ownership, and altering system configuration.

You can verify your current identity with:

```bash
whoami
```

If the result is:

```text
root
```

your current shell is operating as root.

Many distributions allow authorized users to run individual commands with elevated privileges using `sudo`:

```bash
sudo apt update
```

The important part of `sudo` is not that it somehow makes commands work. It changes the privilege context in which the command executes. If a command fails because the syntax is incorrect, a file does not exist, the application is misconfigured, or a remote host is unreachable, adding `sudo` does not fix the underlying problem.

This is worth understanding early because security labs frequently require elevated privileges for specific operations. Instead of treating `sudo` as a universal troubleshooting prefix, use it when the operation actually requires additional privileges.

---

## Understanding Linux File Permissions

A long directory listing contains a permission string at the beginning of each entry:

```text
-rw-r--r-- 1 siri siri 1842 Sep 2 10:15 notes.txt
```

The first ten characters can be divided like this:

```text
- | rw- | r-- | r--
    user  group other
```

The first character describes the type of object. A regular file normally begins with `-`, while a directory begins with `d`. The remaining characters represent permissions for the owner, the group, and everyone else.

The basic permissions are:

```text
r = read
w = write
x = execute
```

For a regular file, read allows its contents to be read, write allows modification, and execute allows the file to be run when it contains executable content in an appropriate format. Directory permissions behave somewhat differently. Read allows directory entries to be listed, write allows entries to be created or removed under the appropriate conditions, and execute allows traversal through the directory.

Permissions can also be represented numerically:

```text
read    = 4
write   = 2
execute = 1
```

The values are added together. Therefore, `7` represents read, write, and execute; `6` represents read and write; and `5` represents read and execute.

A permission such as:

```text
755
```

corresponds to:

```text
Owner:  rwx
Group:  r-x
Others: r-x
```

while:

```text
644
```

corresponds to:

```text
Owner:  rw-
Group:  r--
Others: r--
```

Permissions can be changed with `chmod`. For example:

```bash
chmod 600 private.txt
```

gives the owner read and write access while removing permissions for group and others.

Symbolic notation can also be used:

```bash
chmod u+x script.sh
```

Here, `u` refers to the owner, `+` means add a permission, and `x` means execute. This is often easier to read when you want to make one specific change rather than replace the complete permission set.

---

## Ownership and Groups

Permissions only make sense together with ownership. Linux files normally have both an owner and a group, which can be seen in `ls -l` output.

To change a file's owner:

```bash
sudo chown user filename
```

To change both owner and group:

```bash
sudo chown user:group filename
```

Group ownership can also be changed independently:

```bash
sudo chgrp group filename
```

These operations require appropriate privileges because allowing arbitrary users to assign ownership would undermine the permission model.

When a program cannot access a file, a script cannot execute, or an account can read information that should have been restricted, permissions and ownership are often worth examining together. Looking only at `rwx` without checking which user and group those permissions apply to gives you only half of the picture.

---

## SUID, SGID, and the Sticky Bit

The standard read, write, and execute permissions are not the entire Unix permission model. Linux also supports special permission bits, including **SUID**, **SGID**, and the **sticky bit**.

SUID, or Set User ID, can cause an executable to run with the effective user identity of the file's owner rather than the identity of the person who launched it. This is used legitimately by certain system programs that need carefully controlled access to privileged resources, but it also makes incorrectly configured or vulnerable SUID executables interesting during security assessments.

You can search for regular files with the SUID bit set using:

```bash
find / -type f -perm -4000 2>/dev/null
```

The command searches from `/`, restricts results to regular files, checks for the SUID permission bit, and redirects permission errors to prevent inaccessible directories from overwhelming the useful output.

SGID provides related behavior involving group identity and also has special behavior when applied to directories. The sticky bit is commonly seen on shared directories such as `/tmp`, where multiple users may create files but should not normally be able to delete files belonging to other users.

You do not need to memorize every special permission representation immediately. Recognizing that these mechanisms exist is enough to understand why two files with apparently similar ordinary permissions may still behave differently.

---

## Processes: Understanding What Is Running

A process is a running instance of a program, and Linux assigns each process a numeric **PID**, or process ID. When troubleshooting or examining a machine, process information can tell you which programs are active, which users are running them, and how they were launched.

A common process listing is:

```bash
ps aux
```

The output contains information such as the process owner, PID, CPU and memory usage, and command line. Because a full process list can be large, filtering is common:

```bash
ps aux | grep ssh
```

Another option is:

```bash
pgrep ssh
```

which searches for matching processes directly. To include command information:

```bash
pgrep -a ssh
```

For a continuously updating system view:

```bash
top
```

Some systems also provide `htop`, which offers a more interactive interface, although it may not be installed by default.

Process relationships are also useful. A process can launch another process, creating parent-child relationships that can be displayed with:

```bash
pstree
```

A simplified tree could look like:

```text
systemd
 ├─sshd
 │   └─sshd
 │       └─bash
 │           └─python3
 └─cron
```

A flat process list tells you that `python3` is running. A process tree can help explain how it came to be running. That relationship becomes useful when distinguishing an expected application process from something launched through an unexpected shell, service, or scheduled task.

---

## Signals and Stopping Processes

Linux processes can receive **signals**, which are notifications used to request or force certain behavior. The `kill` command sends a signal to a process identified by PID.

For example:

```bash
kill 1234
```

normally sends `SIGTERM`, which requests that the process terminate gracefully. This gives the program an opportunity to perform cleanup before exiting.

You may also encounter:

```bash
kill -9 1234
```

which sends `SIGKILL`. The target process cannot handle or ignore this signal, so it is useful when a process cannot be terminated normally, but it should not automatically be the first option.

Processes can also be targeted by name with:

```bash
pkill processname
```

Before terminating something, verify what the process actually is. On a real Linux system, stopping the wrong process can interrupt applications, network connectivity, security tooling, or system services.

---

## Services and `systemctl`

Many Linux systems use **systemd** to manage background services and other system components. A service is typically a program intended to run in the background and provide some function to the system or network.

To inspect a service:

```bash
systemctl status ssh
```

Depending on the distribution, the corresponding service may have a slightly different name, such as `sshd`.

A service can be started with:

```bash
sudo systemctl start ssh
```

stopped with:

```bash
sudo systemctl stop ssh
```

and restarted with:

```bash
sudo systemctl restart ssh
```

You may also see:

```bash
sudo systemctl enable ssh
```

Starting and enabling are different operations. `start` changes the service's current runtime state, while `enable` configures the service to start automatically according to its systemd configuration.

To list running services:

```bash
systemctl --type=service --state=running
```

When a lab expects a service to be available but it is not behaving as expected, `systemctl status` is often more useful than immediately changing configuration. It can show whether the service is running, stopped, failed, or repeatedly restarting.

---

## Reading Service Events With `journalctl`

Systems using systemd commonly record events in the systemd journal. The command used to query it is:

```bash
journalctl
```

Running it without filters can return a very large amount of information, so querying a specific service is often more useful:

```bash
journalctl -u ssh
```

To show a limited number of recent entries:

```bash
journalctl -u ssh -n 50
```

To follow new entries as they appear:

```bash
journalctl -u ssh -f
```

Time can also be used as a filter:

```bash
journalctl --since "1 hour ago"
```

Logging differs across Linux distributions and applications, and a later article in this series will focus specifically on logs as security evidence. For now, the Linux skill is knowing where to look when a process managed as a service fails or behaves unexpectedly. `systemctl` tells you about the service's state, while `journalctl` can provide the events surrounding that state.

---

## Checking System Information

Before troubleshooting or analyzing a Linux system, it helps to establish what system you are actually working with. The hostname can be displayed with:

```bash
hostname
```

Kernel and architecture information can be viewed with:

```bash
uname -a
```

If you only need the machine architecture:

```bash
uname -m
```

Typical results include:

```text
x86_64
```

or:

```text
aarch64
```

Many distributions provide identifying information in:

```bash
cat /etc/os-release
```

This can show the distribution name, version, and related identifiers. Knowing the distribution matters because package managers, service names, default configurations, filesystem locations, and available tools can differ.

Basic resource information is also easy to retrieve. To inspect memory:

```bash
free -h
```

To view filesystem disk usage:

```bash
df -h
```

To calculate the size of a particular directory:

```bash
du -sh directory
```

If a program is failing because storage is exhausted or the system is under severe memory pressure, checking those conditions can prevent you from spending time troubleshooting the wrong layer of the problem.

---

## Network Configuration From Linux

Linux provides direct access to the machine's network configuration. To inspect interfaces and their assigned addresses:

```bash
ip addr
```

The shorter form:

```bash
ip a
```

produces the same type of information.

To inspect routes:

```bash
ip route
```

This shows how the machine plans to reach different networks, including the default route when one is configured.

Connectivity can be tested with:

```bash
ping <ip-address>
```

For example:

```bash
ping 192.168.56.10
```

A successful response shows that ICMP echo traffic is being exchanged with the destination. A failed ping does not prove that the remote machine is offline because ICMP may be filtered or disabled.

The networking concepts behind addresses, gateways, ports, TCP, and UDP have their own places in this series. Here, the important Linux-specific skill is knowing how to ask the operating system what interfaces, addresses, and routes it currently has instead of guessing from what you expect the configuration to be.

---

## Listening Ports and Sockets With `ss`

When you need to know which network sockets exist on the local machine, `ss` is one of the most useful commands available.

A common starting point is:

```bash
ss -tuln
```

The options mean:

```text
-t  TCP
-u  UDP
-l  listening
-n  numeric addresses and ports
```

To include process information when your permissions allow it:

```bash
sudo ss -tulpn
```

This can connect a listening socket to the program responsible for it.

For example, a service might be listening on:

```text
0.0.0.0:22
```

while another might appear on:

```text
127.0.0.1:8080
```

Those bindings are different. `127.0.0.1` is the local loopback interface, so a service bound only there is intended to be reached locally. `0.0.0.0` generally means the application is listening on all available IPv4 interfaces.

This becomes useful when a service appears to be running but cannot be reached from another machine. Checking the service state alone may not be enough; you may also need to verify whether it created the expected socket and which interface it bound to.

Older tutorials frequently use:

```bash
netstat
```

You may still encounter it on older systems, but `ss` is generally the modern tool used for this purpose on current Linux distributions.

---

## Open Files and Network Connections With `lsof`

Linux treats many resources through file-like interfaces, and `lsof`, short for **list open files**, can reveal which files a process currently has open.

A basic invocation is:

```bash
lsof
```

but the complete output can be enormous. More targeted use is normally helpful.

To see which processes are using a particular file:

```bash
lsof /path/to/file
```

You may also encounter:

```bash
sudo lsof -i
```

which focuses on network-related open files and sockets.

If you know a port:

```bash
sudo lsof -i :8080
```

can help identify the process associated with it.

This provides another way to connect operating-system activity with network behavior. `ss` is excellent for socket information, while `lsof` is useful when your question is centered on which process currently has a particular resource open.

---

## `curl` and `wget`

A browser is not the only way to interact with resources over a network. `curl` is a command-line data transfer tool that supports numerous protocols and is particularly common when working with HTTP services and APIs.

A basic request looks like:

```bash
curl https://example.com
```

To include response headers:

```bash
curl -i https://example.com
```

To request headers only where appropriate:

```bash
curl -I https://example.com
```

For verbose connection information:

```bash
curl -v https://example.com
```

`curl` can also control methods, headers, request bodies, authentication, certificates, proxies, and many other aspects of a request. The HTTP mechanics behind those requests were covered separately in this series. The Linux skill here is recognizing that `curl` lets you interact directly with a service from the command line, which makes it extremely useful for testing endpoints, checking responses, working with APIs, and troubleshooting applications.

For straightforward file downloads, you will also commonly encounter:

```bash
wget https://example.com/file.txt
```

`wget` and `curl` overlap in some capabilities, but their interfaces and common uses differ. Security labs, installation instructions, and software documentation frequently use one or the other, so being comfortable recognizing both is useful.

---

## DNS Queries From the Terminal

Linux also provides tools that allow DNS to be queried directly. One common command is:

```bash
dig example.com
```

For a shorter result:

```bash
dig +short example.com
```

You may also encounter:

```bash
nslookup example.com
```

These tools help you see how names are being resolved without relying on a browser or another application to perform the lookup invisibly in the background.

DNS has its own behavior and security implications, which will be covered later in this series. At this point, `dig` and `nslookup` belong in the Linux toolkit because they let you directly query and troubleshoot name resolution when a lab or application depends on it.

---

## Environment Variables and `PATH`

Programs frequently need information about their environment. Linux shells expose much of this through **environment variables**.

You can display the environment with:

```bash
env
```

Individual values can be printed using their variable names:

```bash
echo $HOME
```

or:

```bash
echo $USER
```

One particularly important variable is:

```bash
echo $PATH
```

`PATH` contains a list of directories the shell searches when you type a command without specifying its full location. It might contain entries such as:

```text
/usr/local/bin:/usr/bin:/bin
```

If you type:

```bash
python3
```

the shell searches these directories for an executable with that name.

To see what will be resolved:

```bash
command -v python3
```

You may also encounter:

```bash
which python3
```

although `command -v` is generally more closely tied to shell command resolution.

Variables can be created for the current shell environment with `export`:

```bash
export LAB_TARGET="192.168.56.10"
```

and referenced later:

```bash
echo "$LAB_TARGET"
```

Environment variables are widely used by scripts, development tools, cloud utilities, applications, and security tools. They are also sometimes used to hold credentials or API tokens, so their contents should not automatically be treated as harmless configuration data.

---

## Installing Packages With APT

Kali Linux and Ubuntu are Debian-based distributions, so their package-management commands commonly use APT.

Before installing packages, you will frequently see:

```bash
sudo apt update
```

This refreshes information about packages available from configured repositories. It does not mean that all installed software has just been upgraded.

Upgrading installed packages is a separate operation:

```bash
sudo apt upgrade
```

To install a package:

```bash
sudo apt install <package-name>
```

For example:

```bash
sudo apt install curl
```

To search available packages:

```bash
apt search <term>
```

To remove a package:

```bash
sudo apt remove <package-name>
```

Other Linux distributions use package managers such as `dnf`, `yum`, `pacman`, or `zypper`. This is one reason installation instructions copied from a random tutorial may fail even when the package itself exists. Before troubleshooting the package command, confirm which Linux distribution the instructions were written for and which one you are actually using.

---

## Archives and Compression

Downloads used in labs are often packaged as archives. One of the most common formats is:

```text
.tar.gz
```

A tar archive groups files together, while gzip provides compression.

To extract one:

```bash
tar -xzf archive.tar.gz
```

To create one:

```bash
tar -czf archive.tar.gz directory/
```

Before extracting an unfamiliar archive, you can inspect its contents:

```bash
tar -tzf archive.tar.gz
```

ZIP archives are also common:

```bash
unzip archive.zip
```

To create a ZIP archive recursively:

```bash
zip -r archive.zip directory/
```

Understanding these commands becomes useful when downloading tools, collecting lab artifacts, moving evidence between systems, or unpacking datasets. It also prevents the common situation where a tutorial provides an archive and the extraction step becomes a separate obstacle from the task you were actually trying to complete.

---

## File Hashes and Integrity

A cryptographic hash function produces a fixed-size digest from input data. Linux provides simple command-line tools for calculating hashes, which makes them useful when verifying downloads or comparing files.

To calculate a SHA-256 hash:

```bash
sha256sum filename
```

If a trusted software publisher provides an expected SHA-256 value for a downloaded ISO or package, you can calculate the local hash and compare the two values. If the file changes, its calculated digest should also change.

You may also encounter:

```bash
md5sum filename
```

and:

```bash
sha1sum filename
```

MD5 and SHA-1 still appear in legacy systems and file-identification workflows, but they should not be treated as collision-resistant choices for modern cryptographic security. When verifying a download, use the algorithm and trusted reference value provided by the legitimate publisher, preferring modern algorithms such as SHA-256 when available.

Hashes also become useful later when handling security artifacts because they provide a compact way to identify and compare files without relying only on filenames.

---

## Looking Inside Binary Files

Not every file contains readable text. If you run `cat` against a binary executable, the result will usually be meaningless terminal output. The `strings` command extracts sequences of printable characters from binary data:

```bash
strings binaryfile
```

Depending on the file, the output may reveal filenames, URLs, error messages, library names, paths, or other embedded text.

Because `strings` produces text, it can be combined with other commands:

```bash
strings binaryfile | grep -i "http"
```

This searches the extracted printable content for references containing `http`.

Another way to inspect binary data is:

```bash
xxd filename
```

which displays bytes in hexadecimal alongside a textual representation where possible. To inspect only the beginning:

```bash
xxd filename | head
```

You may also encounter:

```bash
hexdump -C filename
```

These tools do not explain everything a binary does, but they provide a way to inspect data when normal text-oriented commands are no longer appropriate. They become particularly useful later in malware analysis, reverse engineering, file-format inspection, and artifact analysis.

---

## Command History and Sensitive Information

Interactive shells commonly maintain command history, which can be displayed with:

```bash
history
```

You can combine it with `grep` to find previous commands:

```bash
history | grep ssh
```

Bash commonly stores persistent history in:

```text
~/.bash_history
```

although the exact behavior depends on shell configuration.

History is convenient because long commands do not need to be reconstructed every time, but it also creates an artifact of previous activity. Passwords, API keys, access tokens, or other sensitive values should not be casually placed directly into commands because they may remain in shell history or become visible through other process and logging mechanisms.

Most shells also support interactive history search. In Bash, pressing `Ctrl+R` allows you to search backward through previous commands, which becomes one of the fastest ways to reuse a complicated command once you start spending more time in the terminal.

---

## Scheduled Tasks

Linux systems often need commands to run automatically at particular times. One traditional scheduling mechanism is **cron**.

A user's cron entries can be displayed with:

```bash
crontab -l
```

and edited with:

```bash
crontab -e
```

System-wide cron configuration may also exist in locations such as:

```text
/etc/crontab
/etc/cron.d/
```

Modern Linux systems may additionally use systemd timers for scheduled execution.

Scheduled tasks are ordinary administrative mechanisms used for jobs such as maintenance, backups, monitoring, and automated scripts. They are also relevant during security analysis because recurring or persistent execution can be implemented through the same mechanisms. If a program appears to start periodically without a user manually launching it, scheduled execution is one of the places that may explain why.

---

## Background Jobs and Terminal Control

Commands normally run in the foreground, which means the shell waits for them to finish before returning the prompt. Adding `&` starts a command as a background job:

```bash
python3 server.py &
```

You can see jobs associated with the current shell using:

```bash
jobs
```

A background or suspended job can be returned to the foreground with:

```bash
fg
```

When a foreground process is running, pressing `Ctrl+Z` normally suspends it. The suspended job can then be continued in the background with:

```bash
bg
```

This is different from `Ctrl+C`, which usually sends an interrupt signal to the foreground process.

Job control becomes useful when several tools need to remain active at the same time. For example, you might keep a local service running in the background while using the same terminal session to inspect files or test its behavior.

---

## Watching System State Change

Sometimes the useful information is not a single snapshot but how something changes over time. The `watch` command repeatedly executes another command and refreshes the displayed result.

For example:

```bash
watch ss -tuln
```

can repeatedly display listening sockets.

To monitor memory information:

```bash
watch free -h
```

The refresh interval can be changed:

```bash
watch -n 2 <command>
```

This runs the command approximately every two seconds.

`watch` is useful when starting or stopping services, reproducing a problem, or observing whether system state changes as expected. Instead of manually rerunning the same command, you can keep the relevant information visible while performing another action.

---

## Shell Scripts: Turning Commands Into Repeatable Work

Once you find yourself running the same commands repeatedly, shell scripting becomes a natural next step. A Bash script is essentially a text file containing commands that the shell can execute in sequence.

A small example might be:

```bash
#!/bin/bash

echo "Hostname:"
hostname

echo "Current user:"
whoami

echo "Network interfaces:"
ip addr
```

The first line:

```bash
#!/bin/bash
```

is called a **shebang**. It tells the operating system which interpreter should be used when the script is executed directly.

After saving the file as `system-info.sh`, you can add execute permission:

```bash
chmod +x system-info.sh
```

and run it with:

```bash
./system-info.sh
```

The `./` matters because the current directory is commonly not included in `PATH`. Typing only `system-info.sh` may therefore produce `command not found` even though the file is visibly present. `./system-info.sh` explicitly tells the shell to execute the file located in the current directory.

Shell scripting can eventually include variables, conditions, loops, functions, and much more, but those features do not need to be mastered before continuing with security labs. The useful idea at this stage is that anything you repeatedly perform manually from the command line can potentially become a repeatable script.

---

## Linux Access Control Goes Beyond `rwx`

Traditional owner, group, and `rwx` permissions are foundational, but they are not the only security controls a Linux system may enforce. Depending on the distribution and configuration, additional mechanisms such as **SELinux**, **AppArmor**, and **Linux capabilities** may affect what a process is allowed to do.

Linux capabilities are particularly useful to recognize because they divide privileges traditionally associated with root into smaller units. A program can therefore possess a particular privileged capability without running with unrestricted root privileges.

Capabilities assigned to an executable can be inspected with:

```bash
getcap <filename>
```

A broader search may use:

```bash
getcap -r / 2>/dev/null
```

SELinux and AppArmor use different models to apply additional policy restrictions to processes and resources. Their configuration and analysis are deeper topics, but knowing these mechanisms exist prevents an important misunderstanding: a permission problem is not always completely explained by the `rwx` bits shown by `ls -l`.

---

## Learning to Read Linux Errors

When a Linux command fails, the error message often tells you which part of the problem deserves attention. Learning to distinguish common errors is more useful than immediately trying random flags or adding `sudo`.

For example:

```text
Permission denied
```

points toward access permissions, ownership, execution rights, or another security control.

```text
No such file or directory
```

suggests that the path is incorrect or the expected object does not exist.

```text
command not found
```

means the shell could not resolve the command, which may indicate that the software is not installed, the executable is not available through `PATH`, or the name was entered incorrectly.

```text
Connection refused
```

belongs to a different layer entirely. It indicates that a connection attempt reached a point where it was actively rejected, commonly because nothing is listening on the destination or something is explicitly rejecting the connection.

These errors should lead to different questions. If the problem is a path, inspect the filesystem. If it is permissions, inspect ownership and access rights. If a service is unavailable, check the service state and listening sockets. Treating every failure as the same problem removes useful information that Linux is already giving you.

---

## Commands Become More Useful When They Answer Questions

The easiest way to become comfortable with Linux is to stop thinking of commands as isolated syntax and start connecting them to questions about the system.

Suppose you receive shell access to a Linux machine in a lab. You might begin with:

```bash
whoami
```

to establish which user you are operating as, followed by:

```bash
pwd
```

to identify your current location and:

```bash
ls -la
```

to understand what exists there.

If you find an unfamiliar file, you could ask what kind of file it is:

```bash
file suspicious_file
```

Then inspect its metadata:

```bash
stat suspicious_file
```

If it contains text, you might read it with:

```bash
less suspicious_file
```

or search a large file for something specific:

```bash
grep -i "error" suspicious_file
```

If the problem concerns a running application, the next question changes. You might use:

```bash
ps aux
```

to inspect processes, then:

```bash
systemctl status <service>
```

if the application is managed as a service.

If the service has failed, you can examine its recent journal entries:

```bash
journalctl -u <service>
```

If it should be reachable over the network, you may then check:

```bash
ss -tulpn
```

to determine whether a socket exists and which process owns it, followed by:

```bash
ip addr
```

and:

```bash
ip route
```

to verify the machine's network configuration.

The value is not in running that exact sequence every time. Different situations require different commands. The important shift is that each command should answer a question created by what you currently know about the system. That turns the terminal into an investigative interface rather than a place where commands are entered because a tutorial told you to type them.

---

## A Compact Reference for Later

After understanding what these commands do, a reference becomes useful because you can return to it without rereading the entire article.

| Task | Useful Commands |
|---|---|
| Current location | `pwd` |
| List files | `ls`, `ls -l`, `ls -la` |
| Move between directories | `cd`, `cd ..`, `cd ~` |
| Create files/directories | `touch`, `mkdir`, `mkdir -p` |
| Copy/move | `cp`, `cp -r`, `mv` |
| Remove files/directories | `rm`, `rmdir`, `rm -r` |
| Identify files | `file`, `stat` |
| Read text | `cat`, `less`, `head`, `tail`, `tail -f` |
| Find files | `find` |
| Search text | `grep` |
| Count/sort/process text | `wc`, `sort`, `uniq`, `cut` |
| Save command output | `>`, `>>`, `tee` |
| Current identity | `whoami`, `id` |
| User sessions | `who`, `w`, `last` |
| Permissions | `ls -l`, `chmod` |
| Ownership | `chown`, `chgrp` |
| Processes | `ps`, `pgrep`, `pstree`, `top` |
| Stop processes | `kill`, `pkill` |
| Services | `systemctl` |
| Systemd events | `journalctl` |
| System information | `hostname`, `uname`, `/etc/os-release` |
| Memory and storage | `free`, `df`, `du` |
| Network configuration | `ip addr`, `ip route` |
| Listening sockets | `ss` |
| Open files/sockets | `lsof` |
| Network requests | `curl`, `wget` |
| DNS queries | `dig`, `nslookup` |
| Environment | `env`, `echo $PATH`, `command -v` |
| Packages | `apt` |
| Archives | `tar`, `zip`, `unzip` |
| File hashes | `sha256sum` |
| Binary inspection | `strings`, `xxd`, `hexdump` |
| Command history | `history` |
| Scheduled tasks | `crontab` |
| Background jobs | `jobs`, `fg`, `bg` |
| Repeated monitoring | `watch` |
| Documentation | `man`, `--help`, `apropos` |

The table is intentionally a reference rather than the main article. If a command is unfamiliar later, return to the relevant section instead of treating the table as something that needs to be memorized.

---

## Getting Help From Linux Itself

Even after becoming comfortable with Linux, you will regularly encounter commands whose options you do not remember. Linux provides documentation directly from the terminal.

Many commands support:

```bash
command --help
```

For example:

```bash
grep --help
```

For more detailed documentation:

```bash
man grep
```

Manual pages are usually displayed through a pager, so you can search inside them using `/` and exit with `q`.

If you know what you want to do but do not know the command, `apropos` can search manual-page descriptions:

```bash
apropos password
```

You may also use:

```bash
whatis grep
```

for a short description when the manual database is available.

The goal is not to reach a point where you never need documentation. Linux tools have too many options, variations, and distribution-specific behaviors for that to be realistic. A stronger skill is knowing what you want the system to tell you, identifying the appropriate tool, and verifying the exact syntax before using it.

---

## From Working Locally to Working Remotely

By this point, the Linux command line should look less like a wall of unfamiliar syntax and more like a collection of ways to ask the operating system specific questions. You can establish your location and identity, navigate the filesystem, inspect files and permissions, search and transform text, examine processes and services, view network configuration, identify listening applications, verify file integrity, manage packages, and combine commands when one tool alone does not provide the answer you need.

So far, however, all of this assumes that you already have access to a shell on the machine you want to work with. In many real environments and cybersecurity labs, the Linux system is somewhere else. It might be another virtual machine in your isolated lab, a server on the network, a cloud instance, or a training machine that you have been authorized to access.

You still need the same shell and the same Linux skills. The difference is that now you need a secure way to reach that shell across a network. That is exactly the problem **SSH**, or Secure Shell, is designed to solve.

Next in the series:

**SSH Explained: Connecting to Your First Remote Machine**
