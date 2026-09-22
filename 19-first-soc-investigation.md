# Your First SOC Investigation: Alert → Evidence → Conclusion

An alert has appeared in the security queue.

It contains a user, a source address, a target system, a time range, and the name of the detection that generated it. The detection has already identified activity that matched a defined pattern, but the alert does not tell us what actually happened or why that activity occurred.

That is where the investigation begins. A SOC investigation takes the initial signal and works backward into the evidence, outward into related activity, and eventually toward a conclusion that can be supported by what the environment actually recorded.

The investigation is not a fixed checklist where every alert requires exactly the same searches. It is a structured process of answering increasingly specific questions until the activity can be explained well enough to make a decision.

```text
              Alert
                |
                v
    Understand the Detection
                |
                v
      Validate the Activity
                |
                v
        Establish Scope
                |
                v
    Collect Related Evidence
                |
                v
       Build the Timeline
                |
                v
      Test Explanations
                |
                v
      Reach a Conclusion
                |
                v
  Document / Escalate / Close
```

---

## Start With What Triggered the Alert

Consider a fictional alert:

```text
Alert:
Repeated Authentication Failures Followed by Success

User:
alice

Source:
192.168.10.25

Target:
WS-17

Failed attempts:
12

Successful login:
02:15:09

Severity:
Medium
```

The first task is not to decide whether Alice's account has been compromised. The first task is to understand exactly why this alert exists and what behavior the detection identified.

The detection might be defined as:

```text
For the same account:

10 or more failed authentications
within 5 minutes

followed by

a successful authentication
within 2 minutes
```

Now the alert has a precise meaning. The SIEM observed events that satisfied those conditions, giving us a starting account, source, target, behavior, and approximate period to examine.

```text
Account:
alice

Source:
192.168.10.25

Target:
WS-17

Behavior:
Repeated authentication failures followed by success

Approximate time:
02:10 - 02:16
```

At this stage, the alert tells us that the pattern occurred. It does not yet tell us who caused it, whether the activity was authorized, or what happened after the successful authentication.

---

## Read the Rule Before Investigating the Story

An alert title is often a simplified description of the underlying detection. Two rules with similar names may operate very differently, and those differences can completely change what the alert establishes.

One rule may count failures from the same source address, another may group by username, and another may require the successful authentication to come from the same source as the failures.

Suppose the detection groups only by username:

```text
Failed login
user=alice
source=192.168.10.25

Failed login
user=alice
source=192.168.10.25

Successful login
user=alice
source=10.20.5.14
```

The rule may still generate an alert even though the successful authentication came from another system. Reading only the alert title could lead to the incorrect assumption that one source repeatedly attempted passwords and eventually authenticated successfully.

Before following the activity further, determine which events matched, how they were grouped, what time window was used, and which conditions caused the rule to fire. Those details establish the boundary of what the alert actually tells us.

---

## Validate the Alert Against the Underlying Events

Once the detection logic is understood, inspect the events that produced the alert. This confirms whether the summarized information accurately represents the activity that matched the rule.

Suppose the underlying records are:

```text
02:13:41  alice  failure  192.168.10.25  WS-17
02:13:48  alice  failure  192.168.10.25  WS-17
02:13:55  alice  failure  192.168.10.25  WS-17
02:14:02  alice  failure  192.168.10.25  WS-17
02:14:09  alice  failure  192.168.10.25  WS-17
02:14:16  alice  failure  192.168.10.25  WS-17
02:14:23  alice  failure  192.168.10.25  WS-17
02:14:30  alice  failure  192.168.10.25  WS-17
02:14:37  alice  failure  192.168.10.25  WS-17
02:14:44  alice  failure  192.168.10.25  WS-17
02:14:51  alice  failure  192.168.10.25  WS-17
02:14:58  alice  failure  192.168.10.25  WS-17
02:15:09  alice  success  192.168.10.25  WS-17
```

The records support the pattern described by the alert. The same account, source, and target appear repeatedly, and the successful authentication follows the failures.

The investigation can now move beyond confirming that the detection worked. The next question is whether the activity has an expected explanation and whether anything important happened around it.

---

## Establish the Immediate Context

The alert gives us three important entities:

```text
alice
192.168.10.25
WS-17
```

Before expanding into every available security source, we need to understand what those entities represent inside the environment.

For the account, useful questions include whether Alice is an employee, administrator, service account, contractor, or another type of identity. Recent password changes, account status, expected working patterns, and normal systems may also become relevant depending on what the organization records.

For the source address, determine which device was using it during the relevant period. If the address belongs to a managed workstation, identify its hostname and owner. If it belongs to a VPN pool, proxy, NAT gateway, or shared infrastructure, the address may not uniquely identify the originating device.

For the target, determine what `WS-17` actually is and why Alice would authenticate to it. A user's assigned workstation, a domain controller, a database server, and a shared administrative host create very different investigative paths.

This stage turns fields from the alert into real identities, devices, and systems. Once those relationships are understood, later searches become much more purposeful.

---

## Expand the Time Window

An alert usually contains only the period necessary for its detection logic. The activity that explains the alert may have started earlier or continued after the detection conditions were satisfied.

If the alert covers:

```text
02:13 - 02:15
```

an analyst might initially examine:

```text
01:45 - 02:45
```

The exact expansion depends on the behavior being investigated. The purpose is to see what happened immediately before the alert and what followed it without jumping directly into an unnecessarily large time range.

The earlier period may reveal activity involving other accounts or systems. The later period may show what happened after authentication, whether the session continued, and whether additional security-relevant behavior followed.

The alert provides the focal point. The surrounding evidence determines how far the useful time range needs to extend.

---

## Look Backward Before Moving Forward

Once the alert has been validated, it can be tempting to immediately investigate everything that happened after the successful login. Looking backward can reveal activity that changes the meaning and scope of the original alert.

Suppose the expanded search reveals:

```text
02:04:12  bob      failure  192.168.10.25
02:04:19  bob      failure  192.168.10.25
02:06:03  charlie  failure  192.168.10.25
02:06:10  charlie  failure  192.168.10.25
02:13:41  alice    failure  192.168.10.25
...
02:15:09  alice    success  192.168.10.25
```

The alert originally appeared to concern Alice. The broader activity now shows that the same source attempted authentication against several accounts before the successful login.

The investigation is therefore no longer limited to one user's failed authentication attempts. The source address and the additional accounts become part of the scope because the surrounding evidence connects them to the activity being examined.

---

## Then Follow What Happened After Success

The successful authentication at `02:15:09` gives us another direction to investigate. We now want to determine what happened during the session that followed.

Suppose endpoint telemetry from `WS-17` shows:

```text
02:15:09  alice  authentication success

02:15:31  alice  powershell.exe started
            parent=explorer.exe

02:16:02  alice  whoami.exe started
            parent=powershell.exe

02:16:08  alice  hostname.exe started
            parent=powershell.exe

02:16:27  alice  network connection
            destination=203.0.113.25
            port=443
```

The investigation now contains more than authentication activity. Process execution and outbound communication occurred shortly after the session began, giving us new evidence and new entities to examine.

The timing makes those events relevant to the session, but timing alone is not enough to explain their purpose. The next stage is to determine how the processes relate to one another and what they were actually doing.

---

## Follow the Process Tree

Process relationships can show how execution developed on an endpoint.

From the events above, we have:

```text
        explorer.exe
             |
             v
       powershell.exe
             |
        +----+----+
        |         |
        v         v
   whoami.exe  hostname.exe
```

This sequence shows that PowerShell was started from Explorer and subsequently launched commands that requested identity and hostname information.

The process tree still needs interpretation. If Alice is an administrator who opened PowerShell manually and ran those commands, the sequence may be expected. If the session came from an unexpected source and the commands do not match Alice's normal activity, the same process tree deserves further examination.

Process names become much more informative when combined with their parent processes, command lines, account, host, and time. Those relationships help determine whether the execution fits a legitimate workflow or requires a broader investigation.

---

## Command Lines Can Explain What a Process Was Doing

A process name identifies the executable, while a command line can reveal how that executable was invoked.

Compare:

```text
powershell.exe
```

with:

```text
powershell.exe -File C:\Scripts\inventory.ps1
```

and:

```text
powershell.exe -EncodedCommand <data>
```

All three involve the same executable, but they provide very different information about the activity being performed.

If command-line auditing is available, the analyst can examine the invocation and determine whether it corresponds to known administration, automation, software behavior, or something that requires additional investigation.

The parent process also remains important. A PowerShell process opened manually from a desktop session represents a different execution path from one launched unexpectedly by a document reader, scheduled task, script interpreter, or another process.

The investigation should therefore follow the execution chain rather than making a decision from the executable name alone.

---

## Investigate the Network Activity

The endpoint evidence showed:

```text
02:16:27
process=powershell.exe
destination=203.0.113.25
port=443
```

The destination now becomes another investigative lead. Useful questions include whether it belongs to an internal or external network, whether the host normally communicates with it, whether other systems contacted the same destination, and whether additional network security data exists for the connection.

Suppose firewall records confirm:

```text
02:16:27
src=WS-17
dst=203.0.113.25
dst_port=443
action=allow
```

The firewall record supports the existence of the connection, but it does not necessarily explain what was transmitted. If proxy information, endpoint network telemetry, DNS records, or another relevant source is available, those sources may provide additional detail.

The investigation follows whatever evidence the environment actually retains. It should not assume that every network connection can be reconstructed down to its application content.

---

## Search for the Destination Elsewhere

Once a destination becomes relevant, it can be searched across the environment to determine whether the activity is isolated.

Suppose a search for:

```text
203.0.113.25
```

returns connections from three hosts:

```text
WS-17  -> 203.0.113.25
WS-22  -> 203.0.113.25
WS-41  -> 203.0.113.25
```

The scope may now extend beyond Alice and `WS-17`. Before doing that, however, we need to determine whether those connections actually represent related behavior.

The destination may belong to a legitimate shared application, update service, security product, or another expected dependency. It may also represent infrastructure associated with the activity being investigated.

The search has therefore produced a new question rather than an automatic answer. Each additional entity needs to be connected back to the investigation through evidence.

---

## Scope Is More Than a Time Range

When analysts talk about **scoping an incident**, they are trying to determine how much of the environment may be involved.

Scope can include:

```text
Users
Hosts
IP addresses
Domains
Applications
Processes
Files
Cloud resources
Sessions
Time periods
```

An investigation that begins with:

```text
User:
alice

Host:
WS-17
```

may eventually expand into:

```text
Users:
alice
bob
charlie

Hosts:
WS-17
WS-22
WS-41

External destination:
203.0.113.25

Time:
02:04 - 02:42
```

The scope should expand when evidence supports that expansion. Searching the entire environment for every possible suspicious event immediately can introduce large amounts of unrelated information and make the investigation harder to follow.

A controlled scope keeps the investigation tied to the activity already discovered while still allowing new evidence to broaden it when necessary.

---

## Determine Whether an Indicator Is Unique or Common

Finding the same address, process, domain, or file on several systems can look significant until its prevalence is understood.

Suppose `203.0.113.25` appears on 400 endpoints every day. That pattern may indicate commonly used infrastructure rather than activity unique to this investigation.

If the destination appears only on three endpoints during the same thirty-minute period and those endpoints also share related process behavior, the relationship deserves closer examination.

The same reasoning can be applied to:

```text
Process names
File hashes
Domains
Command lines
Accounts
Scheduled tasks
Services
Registry paths
```

Prevalence provides another dimension to the evidence. It helps distinguish something rare within the environment from something that initially looked unusual only because the analyst had not yet seen its broader usage.

---

## Use Threat Intelligence as Supporting Information

An investigation may involve an external address, domain, URL, or file hash that can be checked against available threat intelligence.

Suppose an external source reports that:

```text
203.0.113.25
```

has previously been associated with malicious infrastructure.

That information increases the relevance of the destination, but it does not prove what happened on `WS-17`. Threat intelligence can be outdated, incomplete, incorrectly attributed, or associated with activity unrelated to the current case.

Shared hosting and cloud infrastructure create another complication because a single address can serve many unrelated customers.

The investigation should therefore distinguish between what was observed locally and what an external source reports. The local telemetry establishes the connection, while the intelligence provides additional information that may help determine where to investigate next.

---

## Build the Timeline as the Investigation Develops

As relevant evidence accumulates, place it into chronological order so that the sequence can be examined as a whole.

For this investigation:

```text
02:04:12  Authentication failures begin for bob
    |
    v
02:06:03  Authentication failures begin for charlie
    |
    v
02:13:41  Authentication failures begin for alice
    |
    v
02:15:09  Successful authentication for alice on WS-17
    |
    v
02:15:31  powershell.exe starts
    |
    v
02:16:02  whoami.exe executes
    |
    v
02:16:08  hostname.exe executes
    |
    v
02:16:27  PowerShell creates outbound connection
    |
    v
02:16:27  Firewall allows connection to 203.0.113.25
```

The timeline should contain events that help explain the activity rather than every record generated by the system during the period.

A useful timeline allows another analyst to see how the investigation moved from authentication activity into endpoint execution and network communication without having to reconstruct the order from separate searches.

---

## Form Explanations That Can Be Tested

At this stage, several explanations may still fit the evidence.

One possibility is that Alice repeatedly entered an incorrect password, eventually authenticated successfully, opened PowerShell, ran system-information commands, and contacted a legitimate external service.

Another possibility is that someone attempted credentials against several accounts, successfully accessed Alice's account, executed commands on `WS-17`, and established external communication.

There may be additional explanations, and the investigation does not need to choose one before enough evidence exists.

Instead, each explanation can produce questions that are testable against available data.

```text
Question:
Was the source device expected for Alice?

Evidence to examine:
Asset information, DHCP records, VPN information


Question:
Was the PowerShell activity expected?

Evidence to examine:
Command line, process parent, user role, historical activity


Question:
Was the destination expected?

Evidence to examine:
Historical connections, DNS information,
proxy data, application information


Question:
Did the same behavior occur elsewhere?

Evidence to examine:
Environment-wide searches for the process,
command, account, and destination
```

A useful explanation leads to evidence that can support or weaken it. This prevents the investigation from becoming a search for anything that merely looks suspicious.

---

## Search for Contradicting Evidence

Once one explanation begins to look convincing, it becomes easy to search only for information that supports it.

A stronger investigation also looks for evidence that could weaken the current explanation.

If the working explanation is that Alice's account was used by someone else, relevant questions include whether the source device is Alice's normal workstation, whether she regularly works at that time, whether the PowerShell commands match an approved script, and whether the external destination belongs to a known company service.

If those facts are confirmed, the interpretation may need to change. If they are contradicted and additional unusual activity is discovered, the case for escalation becomes stronger.

The purpose is not to force every investigation into two equally likely explanations. It is to make sure the conclusion follows the evidence rather than the analyst's first impression.

---

## Know What Is Confirmed and What Is Still Unknown

As the investigation develops, separate findings that are directly supported from questions that remain unresolved.

For example:

```text
Confirmed

- 12 failed authentication attempts for alice
  originated from 192.168.10.25.

- A successful authentication followed from
  the same source.

- PowerShell executed on WS-17 under
  alice's session.

- WS-17 connected to 203.0.113.25
  shortly afterward.
```

Other questions may still be unresolved:

```text
Unknown

- Who controlled the source device at the time?

- Was the successful authentication authorized?

- What data was exchanged with 203.0.113.25?

- Did the activity affect additional systems?
```

There may also be an assessment supported by several observations:

```text
Assessment

The authentication sequence and subsequent endpoint
activity do not yet have a confirmed expected explanation
and require additional investigation.
```

Keeping these categories separate prevents uncertainty from slowly turning into assumed fact as the case moves between analysts.

---

## Not Every Question Can Be Answered

Sometimes the evidence required to answer a question simply does not exist.

Suppose the analyst wants to determine exactly what data was transmitted during the connection to `203.0.113.25`, but the organization retained only firewall metadata and the communication was encrypted.

The available evidence may establish:

```text
Source
Destination
Port
Time
Connection decision
```

but not:

```text
Exact application content
Files transferred
Commands sent inside the encrypted session
```

The investigation should not fill that gap with an assumption.

A defensible note might state:

```text
The available network telemetry confirms an outbound
connection from WS-17 to 203.0.113.25 at 02:16:27.

The retained data does not provide application content,
so the contents of the communication could not be
determined from the available evidence.
```

A conclusion can still be useful when some questions remain unanswered. Recording the limitation is more valuable than pretending the available telemetry provides certainty that it does not.

---

## Know When the Scope Has Outgrown the Alert

Suppose searches reveal that the same authentication source targeted fifteen accounts and that three systems later communicated with the same external destination.

The investigation is no longer limited to one authentication alert.

At that point, the analyst may need to involve additional responders, preserve relevant evidence, contain affected accounts or systems, or activate the organization's incident-response process.

The exact escalation path depends on the organization. A Tier 1 SOC analyst may not be responsible for performing every containment or forensic action, but they need to recognize when routine alert triage has uncovered activity that requires a broader response.

The quality of the escalation depends heavily on what has already been established. Clearly identified users, systems, timestamps, processes, destinations, and unresolved questions allow the next responder to continue from the current investigation instead of starting over.

---

## Triage and Investigation Are Related but Not Identical

**Triage** is the initial process of determining what an alert represents and what level of attention it requires. The analyst may verify the supporting events, gather immediate information about the entities involved, identify obvious expected activity, and decide whether deeper analysis is necessary.

A deeper investigation expands beyond the initial alert. It may involve additional systems, longer periods, process relationships, account history, network activity, external information, endpoint evidence, and other sources required to determine scope and explanation.

The boundary between the two is not always sharp. A simple alert may be resolved during triage, while another may reveal enough unusual activity within minutes that it needs to become a broader investigation.

The depth of the work should grow with the evidence. Treating every alert as trivial can miss meaningful activity, while treating every alert as a confirmed incident creates unnecessary escalation.

---

## Alert Disposition Records the Outcome

When an investigation is complete enough to make a decision, the alert needs a **disposition**.

Organizations use different terminology, but common categories may include:

```text
True Positive
False Positive
Benign Positive
Expected Activity
Escalated
Confirmed Incident
```

The exact definitions should come from the organization's own process because the same label can be used differently between teams.

A disposition should represent the result of the investigation rather than the analyst's first impression.

An authentication alert might be closed as expected activity after confirming that a password change caused an application to repeatedly attempt authentication using old stored credentials. Another alert might be escalated because the account activity, process execution, and network behavior cannot be explained by legitimate use.

The label records the outcome, while the investigation notes preserve the evidence and reasoning that led to it.

---

## A Good Investigation Note Should Stand on Its Own

Investigation notes should allow another analyst to understand the case without repeating the entire investigation from the beginning.

A useful structure might look like this:

```text
Alert
Repeated Authentication Failures Followed by Success

Initial entities
alice
192.168.10.25
WS-17

Time investigated
02:00 - 03:00

Key findings
- Authentication failures targeted multiple accounts.
- alice successfully authenticated from the same source.
- PowerShell executed shortly after authentication.
- System-information commands followed.
- WS-17 connected to 203.0.113.25.
- Related activity was identified on additional hosts.

Assessment
The activity could not be explained by expected
user behavior and requires further investigation.

Action
Escalated with relevant users, hosts, timestamps,
process information, and network indicators.
```

The note does not need to contain every search the analyst performed. It should preserve the findings that support the conclusion, the important scope, unresolved questions, and any action already taken.

Good documentation reduces duplicated work and allows an escalation to transfer knowledge instead of merely transferring ownership.

---

## Escalation Should Transfer Evidence, Not Just Suspicion

An escalation that says only:

```text
Looks suspicious. Please investigate.
```

provides almost no useful starting point for the next analyst.

A stronger escalation explains what was observed and why additional investigation is needed:

```text
Authentication activity from 192.168.10.25 targeted
three accounts between 02:04 and 02:15. The source
successfully authenticated as alice to WS-17 at 02:15:09.

Within approximately 80 seconds of authentication,
powershell.exe executed under alice's session, followed
by system-discovery commands and an outbound connection
to 203.0.113.25:443.

The source authentication pattern and subsequent endpoint
activity could not be matched to expected activity.
Related connections were also identified on WS-22 and WS-41.

Escalating for broader endpoint and account investigation.
```

The second version gives the next responder the relevant timeline, affected entities, important evidence, and reason for escalation.

A useful escalation should move the investigation forward. The next responder should be able to continue from the work already completed rather than reconstructing why the alert was escalated in the first place.

---

## Know When to Stop

An investigation can expand indefinitely if every discovered value produces another search.

The analyst therefore needs a stopping condition.

An alert may be ready to close when the activity has a well-supported expected explanation and no unresolved evidence requires further analysis. It may be ready to escalate when the evidence shows enough concern that broader response is required.

An investigation may also end with uncertainty when the available telemetry cannot answer the remaining questions. In that case, the uncertainty and evidence gap should be documented rather than hidden behind a definitive disposition that the available data cannot support.

The objective is not absolute certainty about every event in the environment. The objective is enough supported understanding to make the appropriate decision at the analyst's level of responsibility.

---

## A Practical SOC Investigation Flow

The investigation can be reduced to a progression of questions, but each question should lead naturally into the next rather than becoming an isolated checklist.

```text
                 Alert Received
                       |
                       v
        Why did this detection fire?
                       |
                       v
   Do the underlying events support the alert?
                       |
                       v
    Who and what are the entities involved?
                       |
                       v
 What happened before and after the alert?
                       |
                       v
   Did the evidence reveal additional users,
     hosts, processes, destinations, or sessions?
                       |
                       v
       How far does the activity extend?
                       |
                       v
       Which explanations fit the evidence?
                       |
                       v
 What evidence would confirm or contradict them?
                       |
                       v
 What is established, assessed, and still unknown?
                       |
                       v
       Can the activity be explained?
                       |
              +--------+--------+
              |                 |
              v                 v
      Close / Document   Escalate / Respond
```

The exact searches change from alert to alert, but the reasoning remains consistent. Start with what caused the alert, verify the activity behind it, understand the entities involved, expand only where the evidence leads, and determine what the collected evidence can support.

The final result should explain not only what decision was made, but how the investigation reached that decision.

---

## From Individual Concepts to a Working Security Environment

At this point, the pieces that were previously studied separately can begin operating together. A network gives systems a way to communicate, services expose functionality through that network, DNS helps systems locate one another, and firewalls decide which communication is permitted.

Vulnerability information describes known weaknesses in the software being operated. Endpoints, applications, identity systems, and network controls produce security telemetry as activity occurs, while a SIEM brings selected telemetry together and applies detection logic to surface behavior that deserves attention.

An analyst then takes that signal and follows it back through the environment. An investigation may begin with an account and lead to a process, begin with an endpoint alert and move toward network activity, or begin with a destination and reveal related behavior across several hosts.

The remaining step is to stop treating these as separate concepts and make them operate together in one environment. Building that environment deliberately allows us to create activity, observe what each security control records, centralize the resulting telemetry, detect selected behavior, and investigate it from beginning to end.

**Putting Everything Together: Your First Security Home Lab**
