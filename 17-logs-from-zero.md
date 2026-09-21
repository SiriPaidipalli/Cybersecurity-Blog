# Logs From Zero: What Security Analysts Actually Look For

A system is constantly doing things. Users sign in, processes start, services stop, applications receive requests, files change, network connections are created, security controls make decisions, and errors occur.

Many of those activities leave records behind.

Those records are **logs**.

A log is not a complete recording of everything that happened on a system. It is a structured record created when a particular component decides that an event is worth recording. An operating system may record an authentication attempt. A web server may record an HTTP request. A firewall may record a blocked connection. An application may record an error. An endpoint security product may record the creation of a suspicious process.

Individually, these events can look ordinary. Their value becomes much clearer when we begin connecting them.

A failed login at `02:13:41`, followed by a successful login at `02:14:07`, followed by a new process at `02:14:15`, may describe a very different situation from any one of those events viewed alone.

Learning to work with logs therefore involves more than finding suspicious-looking messages. It means understanding **who generated the record, what activity caused it, what fields describe that activity, what the record proves, and how events relate to one another over time**.

---

## A Log Is a Record of an Event

Consider a simplified authentication log:

```text
2026-09-21T14:22:18
host=web01
user=alice
src_ip=192.168.10.25
action=login
result=failed
```

Several pieces of information are immediately available.

```text
Time:
2026-09-21T14:22:18

System:
web01

User:
alice

Source:
192.168.10.25

Activity:
login

Result:
failed
```

The event tells us that a component recorded a failed login involving the account `alice` and source address `192.168.10.25` at the specified time.

That sounds simple, but even this small event raises questions.

Which service generated the log? Was the login attempted through SSH, a web application, VPN, or another authentication mechanism? Does the source address identify the actual originating device, or did the connection pass through a proxy or gateway? Was the username supplied by the person attempting authentication, or resolved from another identity source?

The log contains evidence, but interpreting that evidence requires understanding the system that produced it.

---

## Logs Are Produced by Different Layers of a System

There is no single universal "system log."

A single server can generate logs from several layers at the same time.

```text
Application
     |
     v
Web Server
     |
     v
Operating System
     |
     v
Authentication Services
     |
     v
Host Security Tools
     |
     v
Network Infrastructure
```

Each component observes a different part of the activity.

Suppose someone requests a page from a web application. The web server may record the request path, method, response code, and client address. The application may record the authenticated user and an internal operation. The operating system may record a process failure if the application crashes. A host security tool may record the process behavior. A firewall may record the network connection.

These records describe related activity from different observation points.

That is why investigations often require more than one log source. One source may tell us that a connection occurred, another may identify the account involved, and another may show what process executed afterward.

---

## Common Sources of Security-Relevant Logs

The exact logs available depend on the environment, but several categories appear repeatedly.

### Authentication Logs

Authentication systems record events related to identity and access.

They may contain information such as:

```text
Username
Authentication result
Source address
Authentication method
Session information
Failure reason
Timestamp
```

These logs can help answer questions about failed logins, successful logins, account usage, privilege changes, and authentication patterns.

### Operating System Logs

Operating systems record events related to system behavior.

Depending on the platform and configuration, this can include:

```text
Service activity
System startup and shutdown
Authentication
Process activity
Driver events
Scheduled tasks
Errors
Security policy changes
```

Windows and Linux expose this information differently, but both provide important visibility into what is happening on the endpoint.

### Application Logs

Applications create their own records based on what the developers choose to log.

A business application might record:

```text
User login
Password reset
Account creation
Administrative action
API request
Transaction
Permission failure
Application error
```

Application logs can be especially important because the operating system may know that a process was running without understanding the business action occurring inside that process.

### Web Server Logs

Web servers commonly record requests received by websites and applications.

A record may contain:

```text
Client IP
Timestamp
HTTP method
Requested path
Response status
Response size
User agent
Referrer
```

These logs can help reconstruct activity involving web applications and identify unusual request patterns.

### Network and Security Device Logs

Firewalls, VPN gateways, routers, proxies, DNS infrastructure, intrusion detection systems, endpoint security products, and other controls can produce their own records.

Each sees a different part of the environment.

A firewall may know that a connection was allowed. A DNS server may know which name was queried. A VPN gateway may know which user established a remote session. An endpoint tool may know which process created a connection.

The value comes from understanding what each source can actually observe.

---

## Logs Can Be Structured Very Differently

Not every log looks like:

```text
key=value
```

Some logs are plain text:

```text
Sep 21 14:22:18 web01 sshd[4127]: Failed password for alice from 192.168.10.25 port 51842 ssh2
```

Others use structured formats such as JSON:

```json
{
  "timestamp": "2026-09-21T14:22:18Z",
  "host": "web01",
  "user": "alice",
  "source_ip": "192.168.10.25",
  "event": "login",
  "result": "failed"
}
```

Windows events contain structured fields exposed through the Windows Event Log system. Cloud services may provide JSON records. Web servers often use configurable text formats. Security products may use proprietary schemas.

The appearance changes, but the same basic task remains: identify the fields that describe the event and determine what each field means.

Structured logging makes automated processing easier because values such as usernames, addresses, actions, and results already occupy identifiable fields rather than needing to be extracted from free-form text.

---

## The Fields Matter More Than the Sentence

When first reading logs, it is easy to focus on the human-readable message.

For example:

```text
Failed password for alice from 192.168.10.25 port 51842 ssh2
```

But the useful information can be separated into fields:

```text
event_type    = authentication_failure
user          = alice
source_ip     = 192.168.10.25
source_port   = 51842
service       = ssh
result        = failed
```

Once information is structured this way, we can search and compare it much more effectively.

Instead of reading thousands of messages manually, we can ask questions such as:

```text
Show authentication failures for alice.

Show all events from 192.168.10.25.

Count failed logins by source address.

Show successful logins after repeated failures.

Show activity involving web01 between 14:20 and 14:30.
```

Security analysis becomes much more manageable when log records are treated as collections of searchable fields rather than paragraphs of text.

---

## Time Is One of the Most Important Fields

Nearly every investigation depends on time.

Consider:

```text
14:21:52  Failed login for alice
14:21:58  Failed login for alice
14:22:04  Failed login for alice
14:22:18  Successful login for alice
14:22:31  Privileged command executed
```

The sequence tells us much more than the individual records.

Without reliable timestamps, reconstructing that sequence becomes difficult.

Several details can affect time interpretation:

```text
Time zone
UTC vs local time
Clock synchronization
Timestamp precision
Clock drift
Daylight saving changes
Different logging formats
Ingestion time vs event time
```

Two systems can record the same activity using different time zones.

For example:

```text
Server A:
2026-09-21 14:22:18 EDT

Server B:
2026-09-21 18:22:18 UTC
```

Those timestamps represent the same moment.

If the time zone is ignored, the events appear four hours apart.

This becomes especially important when activity crosses endpoints, servers, network devices, cloud platforms, and security tools. A useful timeline requires knowing what each timestamp represents.

---

## Event Time and Collection Time Are Not Always the Same

A security platform may record more than one relevant timestamp.

For example:

```text
event_time:
2026-09-21T18:22:18Z

received_time:
2026-09-21T18:22:21Z
```

The first represents when the source says the event occurred.

The second represents when another system received the event.

Usually the difference is small, but delays can occur because of network problems, buffering, system load, batch collection, or temporary outages.

An event generated at 18:22 may therefore arrive at a central platform several minutes later.

For reconstructing activity, the event timestamp is usually more useful than simply sorting by when records were collected. The distinction becomes important when logs from several systems are being compared.

---

## Source and Destination Fields Need Careful Reading

Network-related logs commonly include source and destination information.

For example:

```text
src_ip=192.168.10.25
src_port=51842
dst_ip=10.20.30.40
dst_port=443
protocol=tcp
```

These fields describe the communication as observed by the system generating the event.

But the observation point matters.

A reverse proxy may record itself as the network peer while preserving the original client address in another field. NAT may change addresses between different parts of the network. A load balancer may receive the original connection and establish a separate connection toward the backend server.

As a result, fields named `source_ip`, `client_ip`, `remote_addr`, `forwarded_for`, or similar terms may not always represent exactly the same thing.

Before building conclusions around an address, determine what that field means in that particular log source.

---

## Usernames Need the Same Care

A username in a log seems straightforward until an environment contains several identity systems.

You may encounter:

```text
alice
DOMAIN\alice
alice@example.com
alice@corp.example
S-1-5-...
uid=1001
```

Some systems record display names, others record account names, email-style identities, numeric identifiers, or security identifiers.

Service accounts and machine accounts can also appear alongside human users.

A login event involving:

```text
backup-service
```

should not automatically be interpreted the same way as an interactive employee login.

Understanding how identities are represented helps avoid treating different identifiers as different users or treating system-generated activity as human behavior.

---

## Successful Events Matter Too

Security analysis is often associated with failures, errors, blocks, and alerts.

Successful activity can be just as important.

Suppose the logs contain:

```text
02:13:10  Failed login for alice
02:13:18  Failed login for alice
02:13:26  Failed login for alice
02:14:02  Successful login for alice
```

The failed attempts attract attention, but the successful login changes the question.

We may now want to know:

```text
Did the successful login come from the same source?
Was this source expected for alice?
What happened after authentication?
Was a new session created?
Were privileges changed?
Did the account access unusual resources?
```

A security investigation is not simply a search for events containing words such as `failed`, `denied`, or `error`.

Normal-looking successful actions can become important because of what happened immediately before or after them.

---

## One Event Rarely Tells the Whole Story

Consider this record:

```text
user=alice
action=login
result=success
src_ip=192.168.10.25
```

By itself, the event may be completely ordinary.

Now add:

```text
02:13:41 failed login alice 192.168.10.25
02:13:47 failed login alice 192.168.10.25
02:13:53 failed login alice 192.168.10.25
02:14:07 successful login alice 192.168.10.25
02:14:15 privileged process started
02:14:28 outbound connection created
```

The sequence gives us a different problem to investigate.

This process of connecting related events is often called **correlation**.

Correlation does not mean that every event in a sequence has the same cause. It means that records sharing relevant characteristics such as time, user, host, process, address, or session can be examined together to determine whether they describe related activity.

---

## Building a Timeline

A timeline is one of the simplest and most useful ways to organize log evidence.

Suppose several sources produce the following records:

```text
02:13:41  Authentication log
           Failed login for alice

02:13:53  Authentication log
           Failed login for alice

02:14:07  Authentication log
           Successful login for alice

02:14:15  Endpoint log
           New process started

02:14:28  Firewall log
           Outbound connection allowed
```

Instead of investigating each product separately, arrange the events chronologically.

```text
Authentication failures
        |
        v
Successful authentication
        |
        v
Process execution
        |
        v
Network communication
```

Now the investigation can move through the sequence.

Which process started? Which account launched it? What was its parent process? Which destination did it contact? Was the destination expected? Did similar activity occur on other systems?

A timeline converts disconnected records into a sequence that can be tested against an explanation of what happened.

---

## Baselines Help Explain What Is Normal

A single event can be difficult to evaluate without knowing what normally happens in the environment.

Suppose an account logs in at 03:00.

That could be unusual for an employee who normally works during the day, but completely normal for an automated backup account.

A baseline describes expected patterns such as:

```text
Normal login hours
Common source networks
Typical processes
Regular administrative activity
Expected destinations
Normal service behavior
Usual data volume
Scheduled jobs
```

Baselines do not mean every deviation is malicious.

They provide a reference for deciding what deserves further investigation.

A process that has executed every hour for six months has a different history from a process that appeared for the first time immediately after an unusual login.

Historical behavior can therefore add meaning that is not visible in the event itself.

---

## Windows Event Logs

Windows records operating-system and security activity through the **Windows Event Log** system.

Common log channels include:

```text
Application
Security
System
Setup
```

Additional applications and Windows components can create their own channels.

The **Security** log is particularly important because it can contain auditing events related to authentication, account activity, privilege use, process creation, and other security-relevant actions when the appropriate auditing is enabled.

Windows events contain fields such as:

```text
Timestamp
Provider
Event ID
Computer
User information
Event-specific data
```

An **Event ID** identifies a type of event produced by a particular provider.

The number is useful because it allows repeated event types to be recognized and searched consistently.

But an Event ID should not be interpreted without its fields.

Two events with the same ID can involve different users, hosts, processes, addresses, or results. The Event ID tells us what kind of record we are looking at. The event data tells us what actually occurred in that instance.

---

## A Few Windows Events Worth Recognizing

There are thousands of possible Windows events, so memorizing event numbers is not a useful starting point.

A small number appear frequently in security investigations.

For example:

```text
4624
Successful account logon

4625
Failed account logon

4688
New process created
```

These become useful when combined.

Suppose we observe:

```text
4625  Failed login
4625  Failed login
4625  Failed login
4624  Successful login
4688  New process created
```

The event numbers provide a quick description of the sequence, but the investigation still depends on the fields inside those events.

For a `4624`, we may want to know which account authenticated, where the login originated, and what type of logon occurred.

For a `4688`, we may want to know which executable started, which process created it, which account was involved, and what command line was used if that information is being audited.

The numbers help us locate relevant records. The fields provide the evidence.

---

## Windows Logon Types Add More Detail

A successful Windows logon is not always the same kind of login.

Windows records **logon types** that describe how the session was created.

Examples include:

```text
2   Interactive
3   Network
10  RemoteInteractive
```

An interactive logon can represent a user signing into a machine locally.

A network logon can occur when credentials are used to access a resource over the network.

A RemoteInteractive logon is commonly associated with remote interactive sessions such as Remote Desktop.

This means:

```text
Event ID 4624
```

by itself is incomplete.

A more useful interpretation includes:

```text
Event ID
Account
Logon type
Source information
Timestamp
Target system
```

The same successful-logon Event ID can represent very different activity depending on those fields.

---

## Process Creation Logs

Process activity is one of the most useful forms of endpoint evidence.

A simplified process event might contain:

```text
Timestamp:
02:14:15

Host:
WS-17

User:
alice

Process:
powershell.exe

Parent:
winword.exe

Command line:
powershell.exe ...
```

The process name alone is not enough.

`powershell.exe` is a legitimate Windows tool used for administration and automation. Its presence does not establish malicious activity.

The relationship between processes can be much more informative.

```text
winword.exe
     |
     v
powershell.exe
     |
     v
another process
```

That parent-child relationship may deserve investigation depending on what the user was doing, what command was executed, and what happened afterward.

Useful process fields can include:

```text
Process name
Executable path
Process ID
Parent process
Parent process ID
User
Command line
Hashes
Timestamp
Host
```

Not every logging configuration records all of these fields, which is why endpoint visibility depends heavily on what auditing and security tooling have been enabled.

---

## Linux Logs

Linux logging varies by distribution, service, and logging framework.

Traditional Linux systems commonly store logs under:

```bash
/var/log/
```

Possible files include:

```text
/var/log/auth.log
/var/log/syslog
/var/log/messages
/var/log/secure
```

The exact files depend on the distribution.

For example, authentication-related activity may appear in `/var/log/auth.log` on some Debian or Ubuntu-based systems, while `/var/log/secure` is commonly encountered on some Red Hat-based systems.

Modern Linux distributions also commonly use **systemd-journald**, which stores events in the systemd journal.

The important idea is not memorizing one universal Linux log path because there is no single path that applies to every environment.

The first task is determining how that system records and exposes its events.

---

## Reading the systemd Journal

On systems using systemd, the `journalctl` command provides access to the journal.

To view journal entries:

```bash
journalctl
```

On a busy system, this can return far more information than is useful.

Filtering makes the output much more manageable.

To view records associated with the SSH service on a system where the unit is named `ssh`:

```bash
journalctl -u ssh
```

On another distribution, the unit may be named `sshd`:

```bash
journalctl -u sshd
```

Time filtering can narrow the investigation further:

```bash
journalctl --since "2026-09-21 14:00:00" --until "2026-09-21 14:30:00"
```

Now we are asking a specific question:

```text
What did this system record between 14:00 and 14:30?
```

That is much more useful than scrolling through the entire journal looking for something unusual.

---

## Authentication Logs on Linux

An SSH authentication record may resemble:

```text
Sep 21 14:22:18 web01 sshd[4127]:
Failed password for alice from 192.168.10.25 port 51842 ssh2
```

The event provides several useful pieces of information:

```text
Time:
Sep 21 14:22:18

Host:
web01

Service:
sshd

Process ID:
4127

Result:
Failed password

User:
alice

Source:
192.168.10.25

Source port:
51842
```

A successful login may produce another record containing similar identifying information.

By searching for the account or source address, related authentication activity can be collected into a timeline.

For example:

```bash
grep "alice" /var/log/auth.log
```

or:

```bash
grep "192.168.10.25" /var/log/auth.log
```

These commands are simple, but the reasoning behind them is important. We are selecting a field that connects potentially related events and using it to narrow the evidence.

---

## Web Server Logs

Web access logs provide another type of evidence.

A simplified entry might resemble:

```text
192.168.10.25 - - [21/Sep/2026:14:22:18 -0400]
"GET /login HTTP/1.1" 200 1842
```

This can tell us:

```text
Client:
192.168.10.25

Time:
21/Sep/2026 14:22:18 -0400

Method:
GET

Path:
/login

Protocol:
HTTP/1.1

Status:
200

Response size:
1842 bytes
```

A sequence of web requests can reveal how a client interacted with an application.

For example:

```text
GET  /login          200
POST /login          302
GET  /dashboard      200
GET  /admin          403
```

This sequence tells us that the client requested the login page, submitted something to the login endpoint, was redirected, reached the dashboard, and later received a forbidden response when requesting `/admin`.

The log still may not tell us whether authentication succeeded unless the application or surrounding logs provide that information. The HTTP status and request sequence provide evidence, but they should not be stretched beyond what the web server actually recorded.

---

## Status Codes Become More Useful in Sequences

A single HTTP status code can have many explanations.

For example:

```text
404
```

simply tells us that the requested resource was not found according to the server's response.

Now consider:

```text
GET /admin            404
GET /administrator    404
GET /backup           404
GET /.env             404
GET /config.php       404
GET /phpmyadmin       404
```

The individual events are still ordinary HTTP responses.

The sequence suggests that the client is trying several potentially interesting paths.

That observation can lead to additional questions:

```text
How many paths were requested?
How quickly were they requested?
Did other clients show the same behavior?
Did any request return 200?
What happened after a successful response?
```

The pattern emerges from the collection of events rather than from one status code.

---

## Firewall Logs

A firewall event may resemble:

```text
2026-09-21T14:22:18
src=192.168.10.25
dst=10.20.30.40
proto=tcp
spt=51842
dpt=22
action=deny
rule=Block-User-To-Management
```

The record tells us that the firewall evaluated traffic matching those fields and denied it according to the listed rule.

If similar records appear repeatedly:

```text
14:22:18  dst_port=22   deny
14:22:19  dst_port=23   deny
14:22:20  dst_port=80   deny
14:22:21  dst_port=443  deny
14:22:22  dst_port=445  deny
```

the sequence may indicate that one source attempted to reach several services.

The firewall does not necessarily know why those connections were attempted or which process generated them. Endpoint logs from the source system may provide that missing information.

This is a good example of why different log sources complement one another.

---

## DNS Logs

DNS infrastructure can record queries such as:

```text
time=14:22:18
client=192.168.10.25
query=example.test
type=A
response=203.0.113.25
```

This tells us that the client asked the observed DNS infrastructure to resolve an A record for `example.test` and received the listed response.

Suppose endpoint logs later show a process creating a connection toward:

```text
203.0.113.25
```

The two events can be related by time, host, and destination.

```text
DNS query
    |
    v
Address returned
    |
    v
Process starts connection
    |
    v
Firewall observes traffic
```

No individual source contains the entire sequence.

Together, they can explain how the communication developed.

---

## Process IDs Help Connect Events

A **Process ID**, or PID, identifies a running process on a system for the lifetime of that process.

Suppose a log contains:

```text
process=powershell.exe
pid=4812
```

Another event contains:

```text
pid=4812
destination=203.0.113.25
destination_port=443
```

The shared PID can help associate the network activity with that particular process.

Parent process identifiers can provide another relationship.

```text
winword.exe
PID 3104
    |
    v
powershell.exe
PID 4812
```

The parent-child relationship helps reconstruct execution.

PIDs are not permanent identities. Operating systems reuse process IDs after processes terminate, so PID evidence must be interpreted together with the host and time.

A PID of `4812` today is not necessarily related to another process with PID `4812` tomorrow.

---

## Session and Correlation Identifiers

Some applications and security products include identifiers designed specifically to connect related events.

Examples might include:

```text
session_id
request_id
correlation_id
transaction_id
logon_id
```

Suppose several application events contain:

```text
session_id=7F32A1
```

Searching for that identifier can reveal the activity associated with one session even when several users are generating events at the same time.

Similarly, a request ID may allow an HTTP request to be followed through several application components.

These identifiers can be extremely useful because timestamps alone are sometimes insufficient. Hundreds of events can occur within the same second on a busy system.

A unique identifier provides a stronger connection between records when the application makes one available.

---

## Logs Can Be Missing

An investigation should never assume that the absence of a log entry proves that an activity did not occur.

Events can be missing because:

```text
Logging was not enabled
The relevant event was not configured for auditing
The log reached its size limit
Older records were overwritten
The service failed before writing the event
Collection stopped
The endpoint was offline
A log file was deleted
The wrong log source was searched
The activity occurred somewhere else
```

Suppose no process-creation event exists for a particular time.

That could mean no process started.

It could also mean process-creation auditing was not enabled.

Those are very different conclusions.

Before interpreting absence as evidence, determine whether the system was expected to record that event in the first place.

---

## Logs Can Be Modified or Destroyed

Logs are evidence generated and stored by systems, and that evidence can itself become a target.

An attacker with sufficient privileges may attempt to:

```text
Delete log files
Clear event logs
Disable auditing
Stop a logging service
Modify local records
Prevent events from being forwarded
Generate large volumes of distracting events
```

This is one reason organizations often send important logs away from the system that originally generated them.

If an attacker compromises a server and deletes its local logs, a copy already forwarded to another system may still exist.

Central collection does not make logs automatically trustworthy, but it reduces dependence on evidence stored only on the potentially compromised machine.

---

## Log Retention Determines How Far Back You Can Investigate

Logs consume storage.

Organizations therefore define **retention periods** describing how long different records are kept.

For example:

```text
30 days
90 days
180 days
1 year
Several years
```

The appropriate period depends on operational requirements, security needs, legal obligations, compliance requirements, storage cost, and the value of the data.

Retention becomes very practical during an investigation.

If suspicious activity began four months ago but authentication logs are retained for only thirty days, the earliest part of the activity may no longer be available.

Longer retention provides more historical evidence but increases storage and management requirements.

The goal is not to keep every possible event forever. It is to preserve the evidence that the organization is likely to need for operations, security, investigation, and required record keeping.

---

## More Logs Do Not Automatically Mean Better Visibility

It is tempting to solve visibility problems by enabling every possible log source at maximum detail.

That can create a different problem.

Suppose an environment produces:

```text
5 million events per day
```

but only a small portion contains information useful for security monitoring or investigation.

Collecting everything can increase:

```text
Storage cost
Network usage
Processing requirements
Search time
Noise
Operational complexity
```

At the same time, collecting too little can leave important gaps.

Useful logging requires deciding which events matter and whether those events contain enough detail to answer likely questions.

For authentication, recording only:

```text
Login failed
```

is much less useful than recording:

```text
Timestamp
User
Source
Target
Authentication method
Failure reason
```

Logging quality matters as much as logging quantity.

---

## Parsing Turns Raw Logs Into Searchable Data

Raw logs from different systems use different formats.

One source might produce:

```text
Failed password for alice from 192.168.10.25
```

while another produces:

```json
{
  "username": "alice",
  "source": "192.168.10.25",
  "result": "failure"
}
```

To search these consistently, systems can **parse** the raw events and extract fields.

Conceptually:

```text
Raw log
   |
   v
Parser
   |
   v
Structured fields
   |
   +--> user
   +--> source_ip
   +--> action
   +--> result
```

Once events are structured, searches and comparisons become easier.

For example:

```text
result = failure
user = alice
```

can conceptually retrieve relevant authentication failures regardless of how the original text was formatted, assuming the sources have been parsed into compatible fields.

Parsing does not change what happened. It changes how the recorded information is represented and searched.

---

## Normalization Helps Different Sources Speak the Same Language

Parsing extracts information from a log.

**Normalization** goes a step further by representing similar concepts consistently across different sources.

One product might use:

```text
src_ip
```

another:

```text
sourceAddress
```

and another:

```text
client_ip
```

A normalized schema might represent all three as:

```text
source.ip
```

The same can happen with:

```text
username
user_name
account
subject_user
```

which may be mapped into a consistent user field where appropriate.

Normalization makes cross-source analysis easier because searches do not have to know every vendor's original field name.

Care is still required because similarly named fields can have different meanings. Normalization should preserve the meaning of the original data rather than forcing unrelated values into the same field simply because their names look similar.

---

## Start an Investigation With a Question

Opening a log file and scrolling until something looks strange is rarely efficient.

A better approach begins with a question.

Suppose the initial concern is:

```text
Did someone successfully access alice's account
after repeated authentication failures?
```

That question tells us what evidence to look for.

We might begin with:

```text
Account:
alice

Relevant time:
Around the reported activity

Relevant events:
Authentication failures
Authentication successes

Useful fields:
Source address
Target host
Authentication method
Session information
```

If a successful login is found, the question can expand:

```text
What happened during that session?
```

Now other log sources may become relevant.

The investigation grows from evidence rather than from randomly searching every available log.

---

## Pivoting Through Evidence

A useful investigation often moves from one observable value to another.

Suppose we begin with:

```text
user=alice
```

Authentication records reveal:

```text
source_ip=192.168.10.25
```

Searching that source reveals activity against another account.

A successful session then provides:

```text
host=WS-17
```

Endpoint logs on `WS-17` reveal:

```text
process=powershell.exe
pid=4812
```

Process-related activity reveals:

```text
destination=203.0.113.25
```

The investigation has moved through several connected values:

```text
alice
  |
  v
192.168.10.25
  |
  v
WS-17
  |
  v
powershell.exe
  |
  v
203.0.113.25
```

This movement is often called **pivoting**.

A pivot is simply using information discovered in one place to decide what to investigate next.

Useful pivot points include:

```text
Username
Hostname
IP address
Domain
Process
Hash
Session ID
Request ID
File path
Account identifier
```

The quality of the investigation depends on whether those relationships are actually supported by the available evidence.

---

## Separate Observation From Interpretation

Suppose a log contains:

```text
02:14:15
user=alice
process=powershell.exe
```

An observation would be:

```text
The log records powershell.exe executing under alice's account
at 02:14:15.
```

An interpretation might be:

```text
The activity may require investigation because of the events
surrounding that execution.
```

A conclusion such as:

```text
Alice ran malware.
```

would require much more evidence.

Perhaps Alice intentionally used PowerShell. Perhaps another process launched it under her session. Perhaps the account had been compromised. Perhaps the log itself has been misunderstood.

Keeping observation separate from interpretation prevents an investigation from turning an early suspicion into an unsupported conclusion.

A strong analysis makes clear which statements come directly from recorded evidence and which are explanations being tested against that evidence.

---

## Ask What the Log Source Could Actually Know

Every log source has limits.

A firewall can record that it allowed a connection, but it may not know which user initiated it.

A web server can record a request, but it may not know which local process on the client generated that request.

An endpoint tool can record a process, but it may not see traffic after it passes through another device.

A DNS resolver can record a query, but that does not prove the client later connected to the returned address.

Before interpreting an event, ask:

```text
Where was this log generated?

What could this component observe?

Which fields came directly from the activity?

Which values were added or transformed later?

What important information is outside this source's visibility?
```

Those questions help establish the boundary of what the event can support.

---

## A Practical Log Investigation

Imagine an alert says that the account `alice` may have experienced suspicious authentication activity.

Instead of immediately searching every log source, begin with the account and time range.

Authentication logs reveal:

```text
02:13:41  alice  login failed   192.168.10.25
02:13:48  alice  login failed   192.168.10.25
02:13:56  alice  login failed   192.168.10.25
02:14:07  alice  login success  192.168.10.25
```

The successful login gives us a point from which to continue.

Suppose the target host is `WS-17`.

Endpoint records show:

```text
02:14:15  WS-17  alice  powershell.exe  process created
```

A network-related endpoint event then shows:

```text
02:14:28  WS-17  powershell.exe  connection to 203.0.113.25:443
```

A firewall record confirms:

```text
02:14:28
src=WS-17
dst=203.0.113.25
dpt=443
action=allow
```

Now we can construct a timeline:

```text
02:13:41  Authentication failure
02:13:48  Authentication failure
02:13:56  Authentication failure
02:14:07  Authentication success
02:14:15  PowerShell process created
02:14:28  Outbound connection initiated
02:14:28  Firewall allowed connection
```

This still does not tell us everything.

We would want to investigate the PowerShell command, determine whether the source address was expected, examine the destination, identify the parent process, look for related activity, and understand whether the account owner legitimately performed these actions.

But we have moved from an isolated alert to a sequence supported by several log sources.

That is the foundation of log-based investigation.

---

## A Compact Log Analysis Reference

| Element | What to Ask |
|---|---|
| Timestamp | When did the event occur, and in which time zone? |
| Host | Which system generated or experienced the event? |
| User | Which identity was associated with the activity? |
| Source | Where did the activity originate as observed by this system? |
| Destination | What system, service, or resource was targeted? |
| Action | What activity was attempted or performed? |
| Result | Did it succeed, fail, get denied, or produce another outcome? |
| Process | Which executable or service was involved? |
| Parent process | What caused the process to start? |
| Event ID/type | What category of event does the source assign to this record? |
| Session/request ID | Which other events belong to the same activity? |
| Log source | Which system observed and recorded this event? |

When several records appear related, a useful process is:

```text
Start with a question
        |
        v
Identify the relevant time range
        |
        v
Find the first useful event
        |
        v
Extract identifiers
        |
        v
Pivot to related events
        |
        v
Build a timeline
        |
        v
Compare with expected behavior
        |
        v
Test possible explanations
        |
        v
Document what the evidence supports
```

The objective is not to collect the largest number of logs. It is to find the records that help answer the question being investigated.

---

## When the Logs No Longer Fit on One Machine

Reading logs directly on a system works well when the question is narrow and the environment is small. We can inspect an authentication file, query the systemd journal, open Windows Event Viewer, or search an application's local records.

The problem changes when an organization has hundreds or thousands of systems.

An authentication event may exist on one server, a process event on an endpoint, a DNS query on a resolver, a firewall decision on a network device, and an application event somewhere else. Each system can use a different format, retain its logs for a different amount of time, and provide a different way to search them.

An investigation that requires manually signing into every system and searching each source independently becomes slow and difficult to repeat. It also makes relationships across the environment harder to see.

The records become much more useful when important sources can be collected, parsed, normalized, searched, and compared from one place. Once that happens, the same evidence that was scattered across individual machines can begin supporting broader searches, correlations, and security detections.

**SIEM From Zero: From Raw Logs to Security Alerts**
