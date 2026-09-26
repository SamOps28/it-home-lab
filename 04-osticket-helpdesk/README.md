# osTicket Help Desk Lab

A self-hosted IT service management (ITSM) environment built on Windows Server 2022, joined to an Active Directory domain, with email intake from a Microsoft 365 mailbox and tickets worked end to end by multiple agents.

This is lab 04 in my [IT home lab](../) series. It builds on the Active Directory domain and hybrid identity work from labs 01 and 02.

---

## What this lab demonstrates

- Standing up a full web stack and a production-style ticketing application
- Designing a help desk structure: departments, tiers, categories, and SLA targets
- Automated ticket routing based on email content
- Email intake and outbound replies through Microsoft 365 using OAuth2
- Agent authentication against Active Directory using LDAP
- The full ticket lifecycle: log, triage, assign, escalate, resolve, close
- Building a file server with layered NTFS and share permissions
- Group-based access control using the AGDLP model
- Delivering drive mappings by Group Policy with item-level targeting
- Documenting fixes in a knowledge base so repeat issues get faster

---

## Environment

| Machine | Role | IP | Notes |
|---|---|---|---|
| DC01 | Domain controller | 192.168.10.10 | AD DS, DNS, DHCP for lab.local |
| TICKET01 | Help desk server | 192.168.10.30 | Windows Server 2022, XAMPP, osTicket |
| FS01 | File server | 192.168.10.40 | Windows Server 2022, hosts the Accounting share |
| SYNC01 | Identity sync | 192.168.10.20 | Entra Connect to the Microsoft 365 tenant |
| PC01 | User workstation | DHCP | jdoe and Kevin Smith |
| PC02 | Agent workstation | DHCP | Where the help desk agents work |

**Software stack on TICKET01**

- Windows Server 2022 Standard (Evaluation), joined to lab.local
- XAMPP 8.2.12 (Apache 2.4.58, MariaDB, PHP 8.2.12, phpMyAdmin)
- osTicket v1.18.4

TICKET01 and FS01 each have two network adapters. One sits on the internal lab network so the other machines can reach them. The second is NAT, so TICKET01 can reach Microsoft 365 out on the internet to collect mail.

---

## Architecture

```
                    Microsoft 365 tenant
                    helpdesk mailbox
                              |
                              | OAuth2 over IMAP and SMTP
                              | (cron polls every 60 seconds)
                              |
   PC01 (users)  ------>  TICKET01  <------  PC02 (agents)
   jdoe                   Apache + PHP        browser only
   Kevin Smith            MariaDB
        |                     |
        |                     | LDAP bind
        |                     |
        |                   DC01  ----------- FS01
        |                   lab.local         C:\Shares\Accounting
        |                                          ^
        +------------------------------------------+
                 S: drive mapped by Group Policy
```

Nobody logs into TICKET01 or FS01 to do daily work. They are servers. Agents open a browser from PC02 and work through the web interface. That separation between server and desk is deliberate.

---

## Build phases

### Phase A: Server foundation

1. Created the VM, renamed it to TICKET01, and rebooted before joining the domain.
2. Set a static IP of 192.168.10.30 on the lab adapter, with DNS pointed at DC01.
3. Joined TICKET01 to lab.local.
4. Added an inbound firewall rule for HTTP.

Renaming before the domain join matters. If you join first and rename after, the domain still holds the old computer name and you have to leave and rejoin to clean it up.

![TICKET01 renamed and joined to lab.local](screenshots/Picture_1.png)
*TICKET01 renamed and joined to lab.local.*

![Static IP 192.168.10.30, DNS pointed at DC01](screenshots/Picture_2.png)
*Static IP 192.168.10.30, DNS pointed at DC01.*


### Phase B: Web stack and database

1. Installed XAMPP 8.2.12 and set Apache and MySQL to run as Windows services so they survive a reboot.
2. Enabled the PHP extensions osTicket needs: `imap`, `intl`, `gd`, `mbstring`, and later `ldap`.
3. Set a MySQL root password.
4. Created the `osticket` database and a dedicated user with privileges on that database only.

That last point is a small thing that matters. The application never connects as root. If the app is ever compromised, the damage is limited to one database.

![XAMPP Control Panel, Apache and MySQL as services](screenshots/Picture_3.png)
*XAMPP Control Panel, Apache and MySQL as services.*

![XAMPP dashboard reached from PC01 across the network](screenshots/Picture_4.png)
*XAMPP dashboard reached from PC01 across the network.*

![phpinfo confirming the imap extension enabled](screenshots/Picture_5.png)
*phpinfo confirming the imap extension enabled.*


### Phase C: Installing osTicket

1. Extracted osTicket v1.18.4 and copied the contents of the `upload` folder into the web root.
2. Renamed `include\ost-sampleconfig.php` to `ost-config.php`. Nothing runs until this is done.
3. Ran the setup wizard and created the admin agent account.
4. Locked it down afterwards: deleted the `setup` folder and set `ost-config.php` to read-only.
5. Enabled HTTPS using the self-signed certificate that ships with XAMPP.

![osTicket installer prerequisite check passing](screenshots/Picture_6.png)
*osTicket installer prerequisite check passing.*

![Install complete, with post-install cleanup steps](screenshots/Picture_7.png)
*Install complete, with post-install cleanup steps.*


### Phase D: Building the help desk structure

**Departments**

| Department | Purpose |
|---|---|
| Help Desk | General front door for most issues |
| Accounts | Password resets, lockouts, access requests |
| Network | Connectivity and VPN |

Unused default departments (Support, Sales, Maintenance) were set to Private rather than deleted, so existing references do not break.

![Departments: Help Desk, Network, Accounts](screenshots/Picture_8.png)
*Departments: Help Desk, Network, Accounts.*


**Agents**

All agents exist as real Active Directory accounts in `OU=LabUsers,DC=lab,DC=local` and sign in with their domain passwords.

| Agent | Username | Department |
|---|---|---|
| Rico Alvarez | ralvarez | Help Desk |
| Mark Smith | msmith | Network |
| Alex Reyes | areyes | Network |
| Dana Cruz | dcruz | Accounts |
| Sam Penix | spenix | Help Desk (admin) |

![Agents mapped to departments](screenshots/Picture_9.png)
*Agents mapped to departments.*


**Teams**

Tier 1, Tier 2, and Tier 3. Tier 1 is the front door. Tier 2 takes escalations. Tier 3 is reserved for deeper infrastructure work.

![Teams: Tier 1, Tier 2, Tier 3](screenshots/Picture_10.png)
*Teams: Tier 1, Tier 2, Tier 3.*


**SLA plans**

| Plan | Target |
|---|---|
| P1 Critical | 4 hours |
| P2 High | 8 hours |
| P3 Normal | 24 hours |
| P4 Low | 72 hours |

![SLA plans and response targets](screenshots/Picture_11.png)
*SLA plans and response targets.*


**Help topics**

Help topics are the categories users pick from. Each one carries a department and a priority, so choosing a topic routes the ticket and sets its urgency at the same time.

| Topic | Department | Priority |
|---|---|---|
| Access Requests | Accounts | Normal |
| Account Lockout | Accounts | Emergency |
| Email Issue | Help Desk | High |
| General Inquiry | Help Desk | Normal |
| Hardware Failure | Help Desk | High |
| Network or VPN | Network | Emergency |
| New Hire Setup | Help Desk | Normal |
| Password Reset | Accounts | High |
| Printer Issue | Help Desk | Normal |
| Software Install | Help Desk | Low |

Access Requests was added later, after a real ticket arrived that none of the original nine topics fitted. That story is in the problems section below.

![Ten help topics with department and priority](screenshots/Picture_12.png)
*Ten help topics with department and priority.*


### Phase E: Email intake and automation

The help desk mailbox lives in a Microsoft 365 tenant. osTicket collects mail from it and sends replies back through it.

1. Registered an app in Entra ID called osTicket Mail and granted admin consent for `IMAP.AccessAsUser.All`, `SMTP.Send`, `offline_access` and `User.Read`.
2. Connected the mailbox over IMAP on `outlook.office365.com:993` using OAuth2, fetching every minute.
3. Configured outgoing mail over SMTP on `smtp.office365.com:587`, also using OAuth2.
4. Created a Windows scheduled task running `cron.php` every minute so new mail becomes a ticket without anyone clicking Fetch.

![Entra app registration with granted mail permissions](screenshots/Picture_13.png)
*Entra app registration with granted mail permissions.*

![Incoming mail: IMAP over OAuth2, fetching every minute](screenshots/Picture_14.png)
*Incoming mail: IMAP over OAuth2, fetching every minute.*

![Outgoing mail: SMTP over OAuth2](screenshots/Picture_15.png)
*Outgoing mail: SMTP over OAuth2.*


**Automatic routing filters**

Five filters read the incoming subject line and set the help topic, priority, department and team before a human sees the ticket. They run in order, lowest number first.

| Order | Filter | Matches subjects containing |
|---|---|---|
| 10 | Account Lockout Auto-Route | locked, lockout, cannot log in |
| 20 | Password Reset Auto-Route | password, reset my password |
| 30 | Printer Auto-Route | paper jam, printer, printing, toner |
| 50 | Hardware Auto-Route | battery, broken, laptop, not turning on, screen |
| 60 | Access Request Auto-Route | access request, permission, shared folder |

Numbering in tens leaves room to insert a filter later without renumbering everything. Order 40 is currently unused. The Access Request filter sits last on purpose, because its keywords are broad and would otherwise catch tickets the more specific filters should handle.

![Account Lockout filter rules](screenshots/Picture_16-1.png)
*Account Lockout filter rules.*

![Account Lockout filter actions](screenshots/Picture_16-2.png)
*Account Lockout filter actions.*

![Hardware filter rules](screenshots/Picture_17-1.png)
*Hardware filter rules.*

![Hardware filter actions](screenshots/Picture_17-2.png)
*Hardware filter actions.*

![Password Reset filter rules](screenshots/Picture_18-1.png)
*Password Reset filter rules.*

![Password Reset filter actions](screenshots/Picture_18-2.png)
*Password Reset filter actions.*

![Printer filter rules](screenshots/Picture_19-1.png)
*Printer filter rules.*

![Printer filter actions](screenshots/Picture_19-2.png)
*Printer filter actions.*


### Phase F: Active Directory authentication

Rather than osTicket keeping its own separate list of passwords, agents authenticate against DC01 over LDAP.

1. Created a dedicated `svc-osticket` service account in AD for the bind.
2. Installed and configured the osTicket LDAP plugin against lab.local.
3. Agents now sign in with their normal Windows password, which osTicket never stores.

This is the same pattern used at work. One account, one password, disabled in one place when someone leaves.

### Phase G: The file server and the Accounting share

FS01 was added later, to give the help desk something real to grant access to.

1. Built FS01 as a separate server rather than putting a share on the domain controller.
2. Created `C:\Shares\Accounting` with a test file inside.
3. Shared it with Everyone Full Control at the share level.
4. Broke NTFS inheritance, removed the Users entries, and granted Modify to a resource group.

**Why the share is wide open**

A user gets the more restrictive of the two permission sets. Leaving share permissions open means NTFS is the only thing deciding access, so there is one place to look when someone reports a problem instead of two.

![FS01 joined to lab.local on 192.168.10.40](screenshots/Picture_36.png)
*FS01 joined to lab.local on 192.168.10.40.*

![NTFS permissions on the Accounting folder](screenshots/Picture_37.png)
*NTFS permissions on the Accounting folder.*

![Share permissions: Everyone Full Control](screenshots/Picture_38.png)
*Share permissions: Everyone Full Control.*


**Group model (AGDLP)**

```
John Doe, Kevin Smith
        |
        v
GG-Accounting                  (global group, the role)
        |
        v
DL-Accounting-Share-Modify     (domain local group, the resource)
        |
        v
Modify NTFS on C:\Shares\Accounting
```

Users never get permissions directly on a folder, and the role group never gets them either. The resource group holds the permission, and the role group is nested inside it. Adding someone to GG-Accounting is the only action needed to grant access.

![DL-Accounting-Share-Modify containing GG-Accounting](screenshots/Picture_39.png)
*DL-Accounting-Share-Modify containing GG-Accounting.*

![GG-Accounting members after the access request](screenshots/Picture_40.png)
*GG-Accounting members after the access request.*


**Drive mapping by Group Policy**

The share is delivered to users as an S: drive by a GPO called Map Accounting Drive, linked to the LabUsers OU under User Configuration, Preferences, Drive Maps.

Item-level targeting limits it to members of `LAB\GG-Accounting`, so the drive appears only for people who have access and disappears automatically when membership is revoked. No configuration on individual machines.

![Drive map preference mapping the share to S:](screenshots/Picture_41.png)
*Drive map preference mapping the share to S:.*

![Item-level targeting on GG-Accounting](screenshots/Picture_42.png)
*Item-level targeting on GG-Accounting.*

![S: drive on PC01 under Network locations](screenshots/Picture_43.png)
*S: drive on PC01 under Network locations.*


---

## Tickets worked end to end

Three tickets were worked by hand, start to finish, by real agents from real workstations. Each one demonstrates something different.

### Ticket 471760: Account lockout (incident)

- jdoe locked himself out after three failed sign-in attempts and emailed the help desk from PC01.
- The email filter matched, and the ticket arrived with the Account Lockout topic.
- Rico Alvarez claimed it, raised the SLA to P1 Critical, and verified the lockout in Active Directory.
- Rico unlocked the account with `Unlock-ADAccount` and replied to jdoe.
- His reply included a heads-up that a saved password on a phone can cause an immediate repeat lockout, which prevents a second ticket.
- Internal note recorded the verification and the fix. Resolved.

![Ticket 471760 auto-routed and claimed](screenshots/Picture_20.png)
*Ticket 471760 auto-routed and claimed.*

![jdoe's original email in Outlook](screenshots/Picture_21.png)
*jdoe's original email in Outlook.*

![PC01 sign-in screen showing the account locked out](screenshots/Picture_21-1.png)
*PC01 sign-in screen showing the account locked out.*

![jdoe's AD account showing the lockout on DC01](screenshots/Picture_21-2.png)
*jdoe's AD account showing the lockout on DC01.*

![Ticket 471760: claim, topic, SLA change, reply](screenshots/Picture_22.png)
*Ticket 471760: claim, topic, SLA change, reply.*

![Ticket 471760 resolved with the verification note](screenshots/Picture_23.png)
*Ticket 471760 resolved with the verification note.*


### Ticket 353803: Printer jam, escalated (incident)

- Kevin Smith reported a paper jam error with no visible paper.
- The Printer Auto-Route filter set the department, priority, team and topic automatically. The audit trail on the ticket shows each action.
- Rico claimed it from the Tier 1 queue and asked two diagnostic questions: was the error on the printer's own display or only on screen, and could anyone else print to it.
- Kevin confirmed the error showed on the printer panel and that a second user could not print either, ruling out his machine and the print queue.
- Rico escalated to Tier 2 with a documented reason and flagged the month-end deadline.
- Mark Smith attended onsite, opened the rear panel, and found a torn paper fragment behind the feed roller holding the jam sensor closed. Removed it, cleared the error, ran a test print.
- Mark explained the fix to Kevin in plain language. Kevin confirmed his reports printed. Resolved.

The two diagnostic questions are the part worth noticing. They narrowed the fault before anyone was sent out, which is the difference between troubleshooting and guessing.

![Kevin's printer email in Outlook](screenshots/Picture_24.png)
*Kevin's printer email in Outlook.*

![Ticket 353803 on arrival, routed to Tier 1](screenshots/Picture_25.png)
*Ticket 353803 on arrival, routed to Tier 1.*

![Printer Auto-Route audit trail on the ticket](screenshots/Picture_25-1.png)
*Printer Auto-Route audit trail on the ticket.*

![Rico's diagnostic questions to isolate the fault](screenshots/Picture_25-2.png)
*Rico's diagnostic questions to isolate the fault.*

![Kevin confirms the fault is the printer itself](screenshots/Picture_25-3.png)
*Kevin confirms the fault is the printer itself.*

![Escalation to Tier 2 and the onsite fix](screenshots/Picture_25-4.png)
*Escalation to Tier 2 and the onsite fix.*

![Fix explained, user confirms, ticket resolved](screenshots/Picture_25-5.png)
*Fix explained, user confirms, ticket resolved.*


### Ticket 633542: Shared folder access (service request)

This one is a different shape from the other two. Nothing was broken. Someone was asking for something new, and that needs approval rather than a fix.

- Kevin Smith emailed the help desk asking for John Doe to be given access to the Accounting folder.
- Rico verified the approval before acting: the request came from Kevin's own account, and Kevin is a current member of GG-Accounting, making him an appropriate approver for that share. He recorded this in an internal note.
- Rico added jdoe to GG-Accounting on DC01 and documented the full permission chain in the ticket.
- He told Kevin upfront that John would need to restart, rather than waiting for a follow-up complaint.
- After the restart the S: drive appeared. Kevin confirmed. Resolved.

**Why the restart is needed**

Group membership is written into a user's logon token when they sign in. The token does not refresh during an active session. Adding someone to a group changes Active Directory, but the user's computer is still carrying the old token, so nothing changes until they sign in again.

This also applies to the Group Policy drive mapping, because item-level targeting reads the same token.

This is one of the most common false alarms on a help desk. Someone is added to a group, reports it still does not work, and an agent starts pulling apart permissions that were correct all along.

![PC01 as jdoe with no Accounting access](screenshots/Picture_27.png)
*PC01 as jdoe with no Accounting access.*

![Kevin's access request email](screenshots/Picture_27-1.png)
*Kevin's access request email.*

![Rico's reply confirming the approval](screenshots/Picture_27-2.png)
*Rico's reply confirming the approval.*

![Grant note with the permission chain and restart note](screenshots/Picture_27-3.png)
*Grant note with the permission chain and restart note.*

![No S: drive before the restart](screenshots/Picture_28.png)
*No S: drive before the restart.*

![S: drive present after the restart](screenshots/Picture_29.png)
*S: drive present after the restart.*

![Ticket 633542 resolved](screenshots/Picture_30.png)
*Ticket 633542 resolved.*

![Approval verification note](screenshots/Picture_31.png)
*Approval verification note.*

![Grant note and S: drive reply, SLA flagged overdue](screenshots/Picture_32.png)
*Grant note and S: drive reply, SLA flagged overdue.*

![User confirms, ticket closed](screenshots/Picture_33.png)
*User confirms, ticket closed.*

![Closing note recording the GPO delivery](screenshots/Picture_34.png)
*Closing note recording the GPO delivery.*


One accidental detail worth pointing at: while the ticket sat waiting for user confirmation, the system flagged it as overdue against its SLA. That is the SLA clock doing its job, visible in `Picture_32.png`.

---

## Knowledge base

Three articles, each written from one of the tickets above. All are published to the user portal, and each is linked to its matching help topic so it gets suggested when a user picks that category.

| Article | From ticket |
|---|---|
| Why is my account locked and how do I get back in? | 471760 |
| My printer says there is a paper jam, but I cannot see any paper. | 353803 |
| I was given access to a shared folder but I still cannot open it | 633542 |

Each article is split in two. The public answer is written for a non-technical user and tells them what to try and when to stop and raise a ticket. The internal notes hold the technical detail: the PowerShell commands, the event ID to check, the permission chain, the token explanation.

That split matters. The printer article deliberately tells users not to open internal panels, because the actual fix involved reaching behind a hot feed roller. A user following the technician's steps could damage the printer or themselves.

![Knowledge base with all three articles](screenshots/Picture_35.png)
*Knowledge base with all three articles.*


---

## Problems hit and how they were fixed

This is the section I would actually talk through in an interview.

**Apache would not start: port 80 already in use**
IIS was installed and holding port 80. Stopping and disabling the IIS service freed the port. Worth checking with `netstat -ano | findstr :80` before assuming XAMPP is broken.

**osTicket showed a blank page after install**
`ost-sampleconfig.php` had not been renamed to `ost-config.php`. Nothing in the application runs until that file exists under the correct name.

**Email refused to connect, with no useful error**
The PHP `imap` extension was not enabled. This one does not surface until the email phase, long after install, which makes it hard to trace back.

**Microsoft 365 permissions could not be selected in the portal**
Microsoft has deprecated the IMAP and SMTP delegated scopes in the Entra portal permission picker, so they no longer appear in the list. The permissions still work, they just have to be added manually rather than chosen from the menu.

**OAuth token was accepted but mail still failed**
The scopes had been requested against Microsoft Graph instead of `outlook.office.com`. They look almost identical and produce a valid token, but the token is not valid for the mail endpoints.

**Outbound replies failed while incoming mail worked**
SMTP AUTH was disabled at the mailbox level in Exchange. This is a per-mailbox setting and is off by default on newer tenants.

**Cron ran every minute but no tickets appeared**
The "Fetch on auto-cron" option was unticked in the email settings. The scheduled task was firing correctly the whole time, osTicket was just choosing not to poll the mailbox. Nothing in the logs points at this.

**LDAP connection failed with a malformed address**
Using a hostname in the LDAP servers field produced a doubled port in the connection string. Entering an IP address instead bypassed the issue.

**A real ticket arrived that none of the help topics fitted**
Kevin's access request came in with no help topic, no SLA and no team, because none of the original nine topics or four filters matched it. The categories had been designed before any real tickets existed.

The fix was to add an Access Requests topic routed to Accounts, build a custom form to capture who needs access, what to, what level, the business reason and who approved it, and add a fifth filter to catch the keywords.

This is worth more than the tidy version would have been. Categories built in advance always miss something, and noticing the gap from a real ticket is how it actually gets found.

**A reply appeared to send but the user never received it**
The reply was set to All Active Recipients, and the ticket had no collaborators, so it resolved to nobody and sent silently. No error, because nothing failed. Switching to Ticket Owner delivered it.

---

## Screenshots

All screenshots are in [`screenshots/`](screenshots/).

| File | Shows |
|---|---|
| `Picture_1.png` | TICKET01 renamed and joined to lab.local |
| `Picture_2.png` | Static IP 192.168.10.30, DNS pointed at DC01 |
| `Picture_3.png` | XAMPP Control Panel, Apache and MySQL as services |
| `Picture_4.png` | XAMPP dashboard reached from PC01 across the network |
| `Picture_5.png` | phpinfo confirming the imap extension enabled |
| `Picture_6.png` | osTicket installer prerequisite check passing |
| `Picture_7.png` | Install complete, with post-install cleanup steps |
| `Picture_8.png` | Departments: Help Desk, Network, Accounts |
| `Picture_9.png` | Agents mapped to departments |
| `Picture_10.png` | Teams: Tier 1, Tier 2, Tier 3 |
| `Picture_11.png` | SLA plans and response targets |
| `Picture_12.png` | Ten help topics with department and priority |
| `Picture_13.png` | Entra app registration with granted mail permissions |
| `Picture_14.png` | Incoming mail: IMAP over OAuth2, fetching every minute |
| `Picture_15.png` | Outgoing mail: SMTP over OAuth2 |
| `Picture_16-1.png` | Account Lockout filter rules |
| `Picture_16-2.png` | Account Lockout filter actions |
| `Picture_17-1.png` | Hardware filter rules |
| `Picture_17-2.png` | Hardware filter actions |
| `Picture_18-1.png` | Password Reset filter rules |
| `Picture_18-2.png` | Password Reset filter actions |
| `Picture_19-1.png` | Printer filter rules |
| `Picture_19-2.png` | Printer filter actions |
| `Picture_20.png` | Ticket 471760 auto-routed and claimed |
| `Picture_21.png` | jdoe's original email in Outlook |
| `Picture_21-1.png` | PC01 sign-in screen showing the account locked out |
| `Picture_21-2.png` | jdoe's AD account showing the lockout on DC01 |
| `Picture_22.png` | Ticket 471760: claim, topic, SLA change, reply |
| `Picture_23.png` | Ticket 471760 resolved with the verification note |
| `Picture_24.png` | Kevin's printer email in Outlook |
| `Picture_25.png` | Ticket 353803 on arrival, routed to Tier 1 |
| `Picture_25-1.png` | Printer Auto-Route audit trail on the ticket |
| `Picture_25-2.png` | Rico's diagnostic questions to isolate the fault |
| `Picture_25-3.png` | Kevin confirms the fault is the printer itself |
| `Picture_25-4.png` | Escalation to Tier 2 and the onsite fix |
| `Picture_25-5.png` | Fix explained, user confirms, ticket resolved |
| `Picture_27.png` | PC01 as jdoe with no Accounting access |
| `Picture_27-1.png` | Kevin's access request email |
| `Picture_27-2.png` | Rico's reply confirming the approval |
| `Picture_27-3.png` | Grant note with the permission chain and restart note |
| `Picture_28.png` | No S: drive before the restart |
| `Picture_29.png` | S: drive present after the restart |
| `Picture_30.png` | Ticket 633542 resolved |
| `Picture_31.png` | Approval verification note |
| `Picture_32.png` | Grant note and S: drive reply, SLA flagged overdue |
| `Picture_33.png` | User confirms, ticket closed |
| `Picture_34.png` | Closing note recording the GPO delivery |
| `Picture_35.png` | Knowledge base with all three articles |
| `Picture_36.png` | FS01 joined to lab.local on 192.168.10.40 |
| `Picture_37.png` | NTFS permissions on the Accounting folder |
| `Picture_38.png` | Share permissions: Everyone Full Control |
| `Picture_39.png` | DL-Accounting-Share-Modify containing GG-Accounting |
| `Picture_40.png` | GG-Accounting members after the access request |
| `Picture_41.png` | Drive map preference mapping the share to S: |
| `Picture_42.png` | Item-level targeting on GG-Accounting |
| `Picture_43.png` | S: drive on PC01 under Network locations |

One rule followed throughout: no passwords appear in any screenshot. The database user screen, the contents of `ost-config.php`, the email settings and the LDAP plugin page all contain credentials and were either avoided or redacted.

---

## Honest notes

XAMPP is a lab convenience, not a production pattern. A real deployment would be a hardened Linux host running Apache or nginx with a properly issued TLS certificate, a database on a separate host, and the web root locked down. XAMPP was chosen here to get a working desk quickly.

The self-signed certificate on TICKET01 produces a browser warning. That is expected and is the correct behaviour for a certificate no public authority vouches for.

Share permissions set to Everyone Full Control looks alarming out of context. It is deliberate, and NTFS does the real blocking. The reasoning is in the file server section above.

---

## Next steps

- Increase ticket volume, keeping any hand-worked tickets clearly separate from bulk-loaded ones
- Build custom queues so each agent sees only their own open work, plus a queue for anything breaching SLA
- Export SLA and volume reports from the dashboard
- Turn on access-based enumeration on the share, so users only see folders they can open
- Rebuild the same environment on Ubuntu Server to compare the stacks
