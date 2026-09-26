# IT Home Lab

Hands-on labs that simulate a small company's IT environment, built to practice real help desk and system administration work. Each lab builds on the one before it, and each folder has its own write-up with screenshots.

Built and documented by **Sam Penix**.

---

## Labs

| Lab | What it covers |
|---|---|
| [01 - Active Directory](01-active-directory) | Windows Server 2022 domain controller, DNS, DHCP, OUs, Group Policy, NTFS file share permissions |
| [02 - Entra ID and Intune](02-entra-intune) | Hybrid identity with Entra Connect Sync, Microsoft 365 licensing and MFA, hybrid join, Intune compliance, configuration profiles and app deployment |
| [03 - Networking](03-networking) | Cisco Packet Tracer network with VLANs, 802.1Q trunking, router-on-a-stick inter-VLAN routing, and DHCP relay from a server in another VLAN |
| [04 - osTicket Help Desk](04-osticket-helpdesk) | Self-hosted ticketing system with Microsoft 365 email intake, automatic routing, SLAs, AD sign-in for agents, tickets worked end to end, a file server using AGDLP, and a knowledge base |

---

## The environment

Everything runs as virtual machines in VMware Workstation Pro on one isolated lab network, 192.168.10.0/24, in a domain called lab.local. Lab 03 is the exception: it runs in Cisco Packet Tracer and mirrors the same addressing.

| Machine | Role |
|---|---|
| DC01 | Domain controller, DNS, DHCP |
| SYNC01 | Entra Connect Sync to the Microsoft 365 tenant |
| TICKET01 | osTicket help desk server |
| FS01 | File server |
| PC01 | User workstation (Windows 11 Pro) |
| PC02 | Agent workstation (Windows 11 Pro) |

---

## How I approach each lab

- Build it the way a real company would, not the quickest way
- Keep extra roles off the domain controller
- Test every change as a normal user, not just as an admin
- Write down what broke and how I fixed it
- Be honest about shortcuts taken for a lab, and what production would do instead

---

## Tools and technologies

Windows Server 2022, Windows 11, Active Directory, DNS, DHCP, Group Policy, NTFS permissions, Microsoft Entra ID, Entra Connect Sync, Microsoft 365, Intune, osTicket, Cisco Packet Tracer, Cisco IOS, XAMPP (Apache, MariaDB, PHP), PowerShell, VMware Workstation Pro
