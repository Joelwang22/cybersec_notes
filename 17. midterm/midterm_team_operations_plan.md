# Midterm team operations plan

## Purpose

This plan is for a four-person team, with a possible fifth member, attacking the same authorized exam network from separate computers over four days.

The team must:

- Discover and map at least seven possible machines and their network relationships.
- Obtain every required flag.
- Preserve the commands, options, results, and evidence for each step.
- Explain the full attack path and methodology during the final presentation and Q&A.

The main rule is that another teammate must be able to continue any machine using the written record alone.

## Operating model

Use hybrid ownership.

Each machine has one current owner. That person keeps its status, notes, and evidence current. Other members can help with a specialist task, but the owner remains responsible for assembling the complete story.

This avoids two common failures:

- Pure machine ownership creates knowledge silos.
- Pure specialist roles create queues whenever work must pass between people.

The hybrid model keeps responsibility clear while allowing the team to move people toward difficult machines.

## Source-of-truth rules

Use each tool for one purpose.

### Miro shows the current state

Miro answers these questions quickly:

- What machines and networks have we found?
- Which systems can reach each other?
- Which host is the current pivot?
- Who owns each machine?
- What is complete, active, blocked, or unclaimed?
- Where are flags still missing?

Do not put full command histories, long output, or raw credentials on Miro. Store those in Google Docs and link the relevant document from the Miro node.

### Google Docs preserves the complete record

Google Docs answers these questions:

- What did we run?
- Why did we run it?
- What does every command option mean?
- What happened?
- How did the result lead to the next action?
- Where is the supporting evidence?
- How did we obtain each flag?

If Miro and Google Docs disagree, verify the machine and correct both. Do not guess which entry is newer.

### Shared evidence folder holds original proof

Keep screenshots, scans, exported output, and other evidence in a shared folder. Google Docs should link to these files.

Do not rely on screenshots as the only record. Important output should also be copied into the relevant machine document as text.

## Shared workspace structure

Create one shared Drive folder:

```text
Midterm Network PT/
|-- 00 Command Center
|-- 01 Machine Records/
|   |-- H01 - hostname-or-ip
|   |-- H02 - hostname-or-ip
|   `-- ...
|-- 02 Evidence/
|   |-- H01/
|   |-- H02/
|   `-- ...
|-- 03 Presentation
`-- 04 Reference Material
```

Use separate Google Docs for each machine. Seven or more machines in one document will become slow to navigate and difficult to edit concurrently.

Store the existing pivoting guide in `04 Reference Material` or link it from the Command Center:

[`pivoting_exam_guide.md`](pivoting_exam_guide.md)

## Stable identifiers

Assign identifiers as soon as something is discovered. Do not rename identifiers later.

| Item | Format | Example |
|---|---|---|
| Network | `N##` | `N02 - 10.20.30.0/24` |
| Host | `H##` | `H04 - 10.20.30.45` |
| Credential | `C##` | `C07 - local Administrator` |
| Flag | `F-H##-#` | `F-H04-2` |
| Evidence | `E-H##-###` | `E-H04-006` |

Use these identifiers in Miro, Google Docs, evidence filenames, and the presentation. They prevent confusion when hostnames or IP addresses change.

## Miro board design

Divide the board into five areas.

### 1. Network map

Place Kali and known entry points on the left. Add internal networks and deeper machines toward the right.

Every network should have a labelled container such as:

```text
N01 - External/NAT - 192.168.254.0/24
N02 - Internal - 10.20.30.0/24
```

Every host node should contain only:

```text
H04 - HOSTNAME
IPs: 10.20.30.45
OS: Windows Server 2019
Owner: JW
State: ACTIVE
Access: Weak user shell
Flags: 1/2
Doc: link
```

Label connections with what makes them possible:

```text
TCP/445 SMB
TCP/80 HFS
Meterpreter route through H02
Ligolo tunnel through H03
Credential C07 valid
```

Use solid lines for verified reachability and dashed lines for suspected reachability.

### 2. Work queue

Use these columns:

```text
UNCLAIMED | CLAIMED | BLOCKED | VERIFY | COMPLETE
```

Each task card contains:

```text
Task:
Host ID:
Owner:
Started:
Goal:
Last result:
Next action:
Machine Doc link:
```

Examples of good tasks:

- `H03: identify web service on TCP/8080`
- `H05: enumerate local privilege escalation paths`
- `N03: verify route through H04`

Avoid broad cards such as `hack H03`.

### 3. Pivot and session panel

Keep a small table on the board:

| Pivot | Owner | Tool/session | Reachable subnet | Status | Last checked |
|---|---|---|---|---|---|
| `H02` | `AB` | Meterpreter 3 | `N02` | Stable | 14:20 |
| `H04` | `CD` | Ligolo `pivot2` | `N03` | Stable | 14:25 |

This is the fastest way to prevent teammates from building conflicting tunnels or depending on a dead session.

### 4. Blockers and requests

Use this area for short requests:

- Need a second person to verify a flag.
- Need help interpreting a service.
- Pivot session is unstable.
- Target may require a risky exploit.
- Credential needs validation on another host.

Every blocker must name an owner and the next action.

### 5. Presentation queue

Move completed attack paths here when they have enough evidence for the final presentation.

A presentation-ready card must link to:

- The machine document.
- The best topology screenshot.
- The important command output.
- The captured flag evidence.
- A short explanation of why the method worked.

## Miro status system

Use a text code as well as a color. Color alone is easy to misread.

| Code | Meaning | Suggested color |
|---|---|---|
| `NEW` | Discovered but not assessed | Grey |
| `CLAIMED` | Assigned but work has not started | Blue |
| `ACTIVE` | Someone is working on it | Yellow |
| `BLOCKED` | Cannot continue without help or a dependency | Orange |
| `VERIFY` | Believed complete but needs a second check | Purple |
| `COMPLETE` | Required flags and documentation verified | Green |
| `UNSTABLE` | Crashes, dead sessions, or risky state | Red |

## Google Docs design

### Command Center document

The Command Center is the team's index. Keep it short.

It contains:

1. Team roster and current roles.
2. Scope and known network ranges.
3. Links to the Miro board and every machine document.
4. Host status table.
5. Credential index.
6. Flag ledger.
7. Pivot and active-session table.
8. Current blockers.
9. Decisions and major changes.
10. Daily checkpoint summaries.

Suggested host status table:

| Host | Owner | Access | Privilege | Flags | State | Next action |
|---|---|---|---|---|---|---|
| `H04` | `JW` | PowerShell bind shell | Weak | 1/2 | ACTIVE | Validate admin credential |

Suggested credential index:

| ID | Username/domain | Secret type | Found on | Validated on | Evidence | Notes |
|---|---|---|---|---|---|---|
| `C07` | `H04\Administrator` | Password | `H04` | `H04` | `E-H04-006` | Local account |

Never store personal credentials in the exam workspace. Only record credentials obtained from or created for the authorized lab.

Suggested flag ledger:

| Flag ID | Host | Required context | Location/source | Captured by | Verified by | Evidence |
|---|---|---|---|---|---|---|
| `F-H04-1` | `H04` | Administrator | Desktop | `JW` | `AB` | `E-H04-009` |

### Machine document

Create a new document from this template for every host.

```markdown
# H## - hostname or IP

## Current status

- Owner:
- State:
- Hostname:
- IP addresses:
- Operating system:
- Current access:
- Current privilege:
- Reachable from:
- Networks reachable through this host:
- Required flags:
- Captured flags:
- Next action:

## Attack path summary

Write five to ten lines explaining how the team reached this machine,
obtained access, escalated privileges, captured flags, and used it as a
pivot. Update this summary after major progress.

## Reconnaissance

Record discovered ports, services, versions, shares, users, applications,
and useful files. Link the original scan output.

## Initial access

Record the vulnerability, credential, or configuration that provided the
first shell. Explain why the target was vulnerable.

## Privilege escalation

Record the starting identity, method, commands, result, and final identity.

## Credentials and lateral movement

Reference credential IDs. Record where each credential came from and where
it was validated. Do not claim password reuse without a successful test.

## Pivoting

Record interfaces, routes, tunnel commands, payload direction, session IDs,
local ports, bind ports, and the subnets made reachable.

## Flags

For every flag, record its flag ID, location or source, privilege context,
capture method, evidence, and second verifier.

## Command record

Use the command-entry template below.

## Evidence index

List every evidence ID, filename, description, and Drive link.

## Failed paths and limitations

Record only failures that affected the methodology or explain why the team
changed direction. Do not fill this section with every typo.

## Presentation notes

List the few commands, screenshots, and findings needed to explain this host.
```

### Command-entry template

Create one entry for each meaningful command. A pipeline or short sequence may remain together when it performs one task.

```markdown
### CMD-H##-### - Short purpose

Operator:
Time:
Starting host and user:

Command:

```text
exact command here
```

Options:

- `-x`: what this option does in this command.
- `VALUE`: why this value or address was chosen.

Result:

Copy the important output or summarize it precisely.

Interpretation:

Explain what the result proves and why it led to the next action.

Evidence:

- `E-H##-###`: filename and link.

Next action:

State the next test or mark the objective complete.
```

Do not paste a command without explaining its options. That creates trouble during Q&A because the team may remember that it worked but not why.

## Evidence rules

Name evidence consistently:

```text
E-H04-001_nmap-service-scan.txt
E-H04-002_hfs-version.png
E-H04-003_initial-shell.png
E-H04-004_whoami-admin.txt
E-H04-005_administrator-flag.png
```

Every screenshot should show enough context to identify the host and result. Include the terminal prompt, command, relevant output, and hostname or IP where practical.

For important commands, save both:

- A text copy that can be searched and quoted.
- A screenshot that can be placed in the presentation.

Capture evidence when the result occurs. Recreating evidence on the last day wastes time and may be impossible if a session dies.

## Team roles

Roles describe each person's default responsibility. They do not stop people from helping elsewhere.

### Four-person team

| Role | Main responsibility |
|---|---|
| Operations lead and mapper | Maintains the Miro topology, resolves task conflicts, approves risky actions, and runs team checkpoints. This person still performs technical work. |
| Discovery and pivot specialist | Finds hosts and services, validates reachability, establishes routes and tunnels, and documents network paths. |
| Exploitation and lateral-movement specialist | Validates likely vulnerabilities, obtains initial access, tests authorized credentials, and helps owners select payload direction. |
| Post-exploitation and evidence specialist | Performs local enumeration and privilege escalation, tracks flags, checks documentation, and prepares evidence for the presentation. |

### Optional fifth member

The fifth member becomes the verifier and presentation lead. This person:

- Independently verifies completed flags and attack paths.
- Watches for missing command explanations and evidence.
- Maintains the presentation draft.
- Covers blocked tasks or an absent member.
- Leads Q&A practice.

If there are only four members, rotate verification and presentation work. The machine owner cannot be the only person who verifies that machine.

## Machine ownership and task claiming

Follow this sequence before starting work:

1. Check Miro and the Command Center.
2. Create or claim one specific task.
3. Put your initials and start time on it.
4. Confirm that nobody owns a conflicting session, scan, or exploit.
5. Run the command.
6. Record the command and result immediately.
7. Update the host's state and next action.

Only one person owns a host at a time. Helpers claim smaller tasks linked to the host.

If a person stops work for more than 30 minutes, they leave a handoff note:

```text
Host/task:
Current access:
Last successful command:
Important result:
Active session or tunnel:
Next recommended action:
Risks or things not to disturb:
Documentation/evidence links:
```

## Shared-session and pivot rules

Sessions and tunnels are shared dependencies. Treat them carefully.

1. Give every active session or tunnel an owner.
2. Record its source host, destination, payload direction, ports, and reachable subnet.
3. Mark stale or dead sessions immediately.
4. Do not migrate, kill, replace, or background another person's critical session without contacting them.
5. Prefer focused scans through pivots. Broad scans can stall shells and tunnels.
6. Reserve unique bind ports when multiple operators target the same host.
7. Test reachability from the pivot before troubleshooting the tunnel.
8. Keep at least one tested recovery method for every important pivot.
9. Update Miro as soon as a new subnet becomes reachable.

When possible, one stable tunnel should serve the team. Several unrecorded tunnels to the same network create contradictory results and make failures hard to diagnose.

## Credentials and flag handling

### Credentials

- Assign a credential ID before using a new credential elsewhere.
- Record the source and privilege context.
- Test credentials slowly enough to avoid account lockout.
- Record successful and meaningful failed validation attempts.
- Do not change passwords unless the task requires it.
- If a password must change, tell the team first and update the Command Center immediately.

### Flags

- Copy the flag exactly.
- Record the host, source, privilege context, and method used to obtain it.
- Capture text and screenshot evidence.
- Have a second person compare the flag with the original evidence.
- Mark it complete only after verification.
- Do not place flags only in chat messages or Miro cards.

## Risk controls

The operations lead and one other member must approve actions likely to disrupt shared progress, including:

- Rebooting or shutting down a target.
- Running denial-of-service or crash-prone modules.
- Launching a wide password attack.
- Changing a password or network setting.
- Killing a process that owns a shared session.
- Replacing a stable pivot.
- Running a broad scan through a fragile tunnel.
- Reverting a VM snapshot.

Record the decision, owner, expected effect, and recovery plan before running the action.

## Communication cadence

Use one group voice or text channel for short coordination. Keep technical records in the shared documents.

### Start-of-day briefing, 15 minutes

- Review the topology and flag count.
- Confirm all active sessions and tunnels.
- Assign machine owners and priority tasks.
- Identify blockers and risky planned actions.
- Confirm the day's presentation goals.

### Work blocks

Work in blocks of about 90 to 120 minutes. At the end of each block, run a ten-minute sync:

- What changed?
- What new host, credential, route, or flag was found?
- What is blocked?
- Does ownership need to change?
- Is the documentation current?

Do not wait for a checkpoint to announce a new credential, subnet, unstable pivot, or captured flag.

### End-of-day checkpoint, 30 minutes

- Verify the current flag ledger.
- Confirm that every active host has an owner and next action.
- Update the Miro topology.
- Check that important evidence is uploaded.
- Add completed attack paths to the presentation draft.
- Write the next day's first tasks before stopping.

## Four-day schedule

### Before the exam

- Create the Drive folders, documents, Miro areas, and templates.
- Confirm all members can edit every shared file.
- Agree on initials and role assignments.
- Test screenshot capture and evidence uploads.
- Prepare and test approved tools on each operator computer.
- Check tool compatibility with older targets, especially Windows 7.
- Review the pivoting guide together.

### Day 1: discovery and initial access

Goals:

- Establish the first foothold.
- Identify visible hosts, services, and network ranges.
- Create a Miro node and machine document for every discovered host.
- Capture easy flags before moving deeper.
- Identify possible dual-homed pivot machines.

Suggested split:

- One person manages the shared map and performs network discovery.
- Two people validate different high-value services or hosts.
- One person begins local enumeration on the first compromised machine and checks documentation quality.

End Day 1 with a verified topology of everything currently reachable and a prioritized Day 2 queue.

### Day 2: privilege escalation and expansion

Goals:

- Complete local privilege escalation on foothold machines.
- Collect and validate credentials.
- Establish the first internal route or tunnel.
- Discover and attack the next network segment.
- Recover immediately useful flags.

Keep one person responsible for pivot stability. Other members should not change routes or tunnels without coordinating with that person.

End Day 2 with documented attack paths into every discovered network and recovery instructions for important pivots.

### Day 3: close gaps and build the presentation

Goals:

- Reach remaining machines and networks.
- Resolve missing flags.
- Recheck machines marked blocked or incomplete.
- Have a second person verify every completed host.
- Finish command explanations and evidence indexes.
- Build most of the presentation.

By the end of Day 3, the presentation should already contain the network diagram, methodology, and completed machine stories. Day 4 should not begin with an empty slide deck.

### Day 4: verification, presentation, and Q&A

Goals:

- Perform a final host and flag audit.
- Freeze the Miro topology used for the presentation.
- Correct inconsistent IP addresses, hostnames, credentials, and route descriptions.
- Finish the slides and speaker notes.
- Rehearse the complete progression and Q&A.

Avoid risky technical work after the final verification point unless a required flag is still missing.

## Machine definition of done

A host may move to `COMPLETE` only when:

- Its identity, addresses, operating system, and network position are recorded.
- The route or access path to it is clear.
- Initial access is explained.
- Privilege escalation is explained, or the document states why it was unnecessary.
- Every required flag is captured with evidence.
- Exact important commands and their options are documented.
- Credentials found or used are referenced by ID.
- Pivot routes and sessions are documented if applicable.
- Failed paths that explain a methodology change are recorded.
- A second person has checked the flags and attack-path summary.
- The machine has enough evidence for its presentation section.

## Whole-network definition of done

The assessment is ready for presentation when:

- Every discovered network and machine appears on Miro.
- Miro matches the Command Center and machine documents.
- Every required flag appears once in the verified flag ledger.
- Every pivot path is reproducible from the written commands.
- The presentation follows the real attack order.
- Every displayed command has an explanation of its important options.
- Every claim has supporting output or evidence.
- Each person can explain the overall topology.
- Each machine owner can answer detailed questions about their machines.
- At least one other person can explain each machine if its owner is unavailable.

## Presentation structure

Use one consistent story.

1. Scope, goal, team, and rules of engagement.
2. Final network topology.
3. Methodology and team workflow.
4. Initial discovery and first foothold.
5. Machine-by-machine progression in attack order.
6. Privilege escalation and credential discoveries.
7. Pivoting and newly reached network segments.
8. Flag summary tied to hosts and privilege contexts.
9. Problems, limitations, and how the team changed methods.
10. Final attack-path recap.

For each machine, answer the same questions:

- How did we discover it?
- What services mattered?
- How did we obtain access?
- How did we escalate privileges?
- What credentials or pivots came from it?
- How did we obtain its flags?
- What evidence proves each step?

## Q&A preparation

Create a question list while building the presentation. Likely questions include:

- Why did you choose that scan or exploit?
- What does each command option do?
- How did you know the vulnerability applied?
- Why did you use a bind payload instead of a reverse payload?
- How did traffic reach the hidden network?
- What privilege did you have before and after escalation?
- How did you verify that a credential or flag belonged to that machine?
- What failed, and why did you change methods?
- How would you recover if the pivot died?

During rehearsal, the presenter answers first. Another member then checks the answer against the machine document. Fix vague explanations before the presentation.

## Short version for the team

```text
1. Check Miro and the Command Center.
2. Claim a specific task before running anything.
3. Keep one owner per machine and one owner per shared pivot.
4. Record meaningful commands, options, results, and evidence immediately.
5. Put full details in the machine Doc, not on Miro.
6. Announce new credentials, routes, flags, and unstable sessions at once.
7. Require approval before disruptive actions.
8. Get a second person to verify every completed machine and flag.
9. Add presentation material throughout the exam, not only on Day 4.
10. A machine is complete only when another teammate can explain and continue it.
```
