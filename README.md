<img src="banner.svg" alt="Enterprise Active Directory Infrastructure" width="100%" />

# Enterprise Active Directory Infrastructure — Deployment, Administration & Incident Response

---

## Table of Contents

- [Overview](#overview)
- [Objective](#objective)
- [Architecture Diagram](#architecture-diagram)
- [Technologies Used](#technologies-used)
- [Prerequisites](#prerequisites)
- [Environment Setup](#environment-setup)
- [Implementation Steps](#implementation-steps)
  - [Phase 1: AWS Infrastructure](#phase-1-aws-infrastructure)
  - [Phase 2: AD DS Installation & Initial OUs](#phase-2-ad-ds-installation--initial-ous)
  - [Phase 3: OU Structure, User & Group Creation](#phase-3-ou-structure-user--group-creation)
  - [Phase 4: File Shares & NTFS Permissions](#phase-4-file-shares--ntfs-permissions)
  - [Phase 5: Group Policy Hardening](#phase-5-group-policy-hardening)
  - [Phase 6: Ticket Queue — Simulated Day-2 Operations](#phase-6-ticket-queue--simulated-day-2-operations)
  - [Phase 7: Windows Client Domain-Join](#phase-7-windows-client-domain-join)
- [Results](#results)
- [Challenges & Troubleshooting](#challenges--troubleshooting)
  - [Incident 1: RDP Access Failure](#incident-1-rdp-access-failure)
  - [Incident 2: VPN IP Change & SSM-on-DC Limitation](#incident-2-vpn-ip-change--ssm-on-dc-limitation)
  - [Incident 3: Domain-Join Blocked by Account Lockout Cascade](#incident-3-domain-join-blocked-by-account-lockout-cascade)
- [Lessons Learned](#lessons-learned)
- [Future Improvements](#future-improvements)
- [References](#references)
- [Author](#author)

---

## Overview

This project simulates the full lifecycle of an enterprise Active Directory environment, built and operated on AWS from the ground up. It covers cloud infrastructure provisioning, Active Directory Domain Services (AD DS) deployment, organizational design across four departments, file share and NTFS permission structuring, Group Policy administration, ticket-driven day-to-day operations, and a Windows client domain-join — including three real, unplanned incidents that were diagnosed and resolved (or actively worked through) along the way.

Rather than a clean, linear walkthrough, this repo documents the project as it actually happened: including the misconfigurations, cascading failures, and troubleshooting steps that come with real enterprise IT/SOC work.

> **Security note:** Screenshots throughout this repo have been reviewed and scrubbed of exposed credentials before publishing. If you're documenting your own lab this way, always check screenshots for visible passwords, key material, or personal data before committing them.

---

## Objective

To build a realistic, hands-on enterprise Active Directory environment that demonstrates:

- End-to-end cloud infrastructure provisioning for a Windows Server/AD environment
- Correct AD DS deployment and domain design (OUs, users, groups, delegation-ready structure)
- Departmental access control via NTFS permissions, SMB shares, and security groups
- Security hardening through Group Policy (password policy, lockout policy, drive mapping, USB restriction, patch deferral)
- Realistic day-to-day AD administration via a simulated ticket queue
- Practical incident response and troubleshooting skills across networking, AWS security groups, DNS, and AD account policy — the skills a SOC Analyst, Incident Response Analyst, or Systems/Cloud Administrator is expected to apply on the job

---

## Architecture Diagram

<img src="architecture-diagram.svg" alt="Network and logical architecture diagram" width="100%" />

The environment runs inside a single AWS VPC (`10.0.0.0/16`), with **DC01** (Domain Controller + DNS) and **CLIENT01** (domain-joined member) sharing a subnet, gated by a security group that was tightened over the course of the project's incidents. See [Challenges & Troubleshooting](#challenges--troubleshooting) for how the network and access-control boundaries evolved in response to real failures.

---

## Technologies Used

| Category | Tools / Services |
|---|---|
| Cloud Platform | AWS (EC2, VPC, Subnets, Security Groups, Elastic IP, Systems Manager, IAM) |
| Operating System | Windows Server 2022 Datacenter |
| Directory Services | Active Directory Domain Services (AD DS), DNS |
| Policy Management | Group Policy Management Console (GPMC) |
| Scripting | PowerShell (`New-ADUser`, `Get-ADUser`, `Set-Acl`, `Unlock-ADAccount`, etc.) |
| File Services | NTFS Permissions, SMB File Shares |
| Access | Remote Desktop Protocol (RDP), AWS Systems Manager Session Manager |
| Diagnostics | `nslookup`, `ping`, Event Viewer, Windows Firewall logs |

---

## Prerequisites

- An AWS account with permissions to create VPCs, EC2 instances, security groups, and IAM users
- Basic familiarity with Windows Server administration
- Basic familiarity with Active Directory concepts (domains, OUs, GPOs, trusts)
- An RDP client (Microsoft Remote Desktop / Windows App on macOS, or the built-in Remote Desktop Connection on Windows)
- A generated EC2 key pair for decrypting the initial Windows Administrator password

---

## Environment Setup

1. AWS account configured with an IAM admin user (least-privilege principle applied where practical for a lab context)
2. Custom VPC (`10.0.0.0/16`) with a public subnet (`10.0.1.0/24`), internet gateway, and route table
3. Security group (`Enterprise-Server-SG`) initially scoped to RDP/SSH/HTTP/HTTPS, later expanded to permit internal VPC traffic (see Incident 3)
4. Two EC2 instances:
   - **DC01** — Windows Server 2022, promoted to Domain Controller for `enterprise.local`
   - **CLIENT01** — Windows Server 2022, configured as a domain-joined client stand-in (see note below)
5. Key pair (`Enterprise-Lab-Key`) used to decrypt initial local Administrator passwords for both instances

> **Licensing note:** AWS does not offer a licensed Windows 11 client AMI for EC2. CLIENT01 was provisioned as a **Windows Server 2022 instance configured and used as a client stand-in**, to demonstrate the domain-join and client-side verification workflow within AWS's available licensing constraints.

---

## Implementation Steps

### Phase 1: AWS Infrastructure

Provisioned the foundational AWS networking and compute layer: IAM setup, a custom VPC, subnets, an internet gateway, route tables, security groups, and the initial EC2 instance (DC01) with an associated key pair and Elastic IP.

<p float="left">
  <img src="01-aws-infrastructure/01.png" width="32%" />
  <img src="01-aws-infrastructure/03.png" width="32%" />
  <img src="01-aws-infrastructure/05.png" width="32%" />
</p>
<p float="left">
  <img src="01-aws-infrastructure/07.png" width="32%" />
  <img src="01-aws-infrastructure/09.png" width="32%" />
  <img src="01-aws-infrastructure/11.png" width="32%" />
</p>
<p float="left">
  <img src="01-aws-infrastructure/13.png" width="32%" />
  <img src="01-aws-infrastructure/14.png" width="32%" />
  <img src="01-aws-infrastructure/15.png" width="32%" />
</p>

*Full set: `01-aws-infrastructure/` (15 screenshots)*

### Phase 2: AD DS Installation & Initial OUs

Installed and configured Active Directory Domain Services on DC01, establishing the `enterprise.local` domain, and began the initial Active Directory Users and Computers structure.

<p float="left">
  <img src="02-ad-ds-installation-ous/01.png" width="32%" />
  <img src="02-ad-ds-installation-ous/03.png" width="32%" />
  <img src="02-ad-ds-installation-ous/05.png" width="32%" />
</p>
<p float="left">
  <img src="02-ad-ds-installation-ous/07.png" width="32%" />
  <img src="02-ad-ds-installation-ous/09.png" width="32%" />
  <img src="02-ad-ds-installation-ous/12.png" width="32%" />
</p>

*Full set: `02-ad-ds-installation-ous/` (12 screenshots)*

### Phase 3: OU Structure, User & Group Creation

Designed and created **8 Organizational Units** (Domain Controllers, IT, HR, Finance, Sales, Groups, Servers, Service Accounts, Workstations) to reflect departmental structure, and provisioned the initial set of users and security groups via PowerShell (`New-ADUser`, `New-ADGroup`, `Get-ADUser`).

<p float="left">
  <img src="03-ou-user-group-creation/01.png" width="32%" />
  <img src="03-ou-user-group-creation/03.png" width="32%" />
  <img src="03-ou-user-group-creation/05.png" width="32%" />
</p>
<p float="left">
  <img src="03-ou-user-group-creation/07.png" width="32%" />
  <img src="03-ou-user-group-creation/09.png" width="32%" />
  <img src="03-ou-user-group-creation/12.png" width="32%" />
</p>

*Full set: `03-ou-user-group-creation/` (12 screenshots)*

### Phase 4: File Shares & NTFS Permissions

Expanded departmental structure to cover all **4 departments**, adding **Finance Users** and **Sales Users** security groups and users (Sarah Chen, Mike Torres). Created and verified NTFS permissions and SMB file shares per department via PowerShell (`New-SmbShare`, `icacls`).

<p float="left">
  <img src="04-file-shares-ntfs/01.png" width="32%" />
  <img src="04-file-shares-ntfs/03.png" width="32%" />
  <img src="04-file-shares-ntfs/05.png" width="32%" />
</p>
<p float="left">
  <img src="04-file-shares-ntfs/07.png" width="32%" />
  <img src="04-file-shares-ntfs/08.png" width="32%" />
</p>

*Full set: `04-file-shares-ntfs/` (8 screenshots)*

### Phase 5: Group Policy Hardening

Configured and applied GPOs across all four departments:

- **Password Policy** — complexity and history requirements
- **Account Lockout Policy** — threshold and duration for failed login attempts
- **Screen Saver Timeout** — automatic lock after inactivity
- **USB Restriction** — removable media control
- **Windows Update Deferral** — controlled patch rollout
- **Drive Mapping** — automatic network drive assignment per department (I: IT, H: HR, F: Finance, S: Sales)

<p float="left">
  <img src="05-group-policy-hardening/01.png" width="32%" />
  <img src="05-group-policy-hardening/04.png" width="32%" />
  <img src="05-group-policy-hardening/07.png" width="32%" />
</p>
<p float="left">
  <img src="05-group-policy-hardening/10.png" width="32%" />
  <img src="05-group-policy-hardening/13.png" width="32%" />
  <img src="05-group-policy-hardening/16.png" width="32%" />
</p>
<p float="left">
  <img src="05-group-policy-hardening/19.png" width="32%" />
  <img src="05-group-policy-hardening/21.png" width="32%" />
  <img src="05-group-policy-hardening/22.png" width="32%" />
</p>

*Full set: `05-group-policy-hardening/` (22 screenshots)*

> The Account Lockout Policy configured here later played a direct role in Incident 3.

### Phase 6: Ticket Queue — Simulated Day-2 Operations

Four representative help-desk tickets were worked end-to-end via PowerShell to simulate ongoing AD administration:

| Ticket | Action |
|---|---|
| Account unlock | Unlocked `jsmith`'s account after lockout (`Unlock-ADAccount`) |
| Password reset | Reset `mjohnson`'s password (`Set-ADAccountPassword`) |
| New hire onboarding | Provisioned Priya Nair into the Sales department (OU, groups, drive mapping) |
| Offboarding | Disabled `mtorres`'s account per departure process (`Disable-ADAccount`) |

<p float="left">
  <img src="06-ticket-queue-operations/01.png" width="24%" />
  <img src="06-ticket-queue-operations/02.png" width="24%" />
  <img src="06-ticket-queue-operations/03.png" width="24%" />
  <img src="06-ticket-queue-operations/04.png" width="24%" />
</p>

*Full set: `06-ticket-queue-operations/` (4 screenshots — one per ticket)*

### Phase 7: Windows Client Domain-Join

To demonstrate the client side of the domain — logging in as a standard domain user, verifying group-driven access, and confirming drive mapping — CLIENT01 was provisioned and joined to `enterprise.local`. This phase is where Incident 3 occurred; see [Challenges & Troubleshooting](#challenges--troubleshooting) for the full breakdown.

<p float="left">
  <img src="07-client-domain-join-incident3/01.png" width="32%" />
  <img src="07-client-domain-join-incident3/03.png" width="32%" />
  <img src="07-client-domain-join-incident3/05.png" width="32%" />
</p>
<p float="left">
  <img src="07-client-domain-join-incident3/07.png" width="32%" />
  <img src="07-client-domain-join-incident3/09.png" width="32%" />
  <img src="07-client-domain-join-incident3/11.png" width="32%" />
</p>

*Full set: `07-client-domain-join-incident3/` (11 screenshots)*

---

## Results

| Metric | Count |
|---|---|
| Organizational Units | 8 |
| Users | 9 (including Priya Nair; excludes disabled Mike Torres) |
| Security Groups | 6 |
| SMB Shares | 4 |
| GPO Settings applied | 6 |
| Help-desk tickets resolved | 4 |
| Incidents diagnosed & resolved/documented | 3 |

The environment successfully supports departmental OU-based access control, GPO-enforced security baselines, functioning file shares scoped by NTFS permission and group membership, and a domain-joined client demonstrating the end-to-end trust relationship — with the domain-join process itself surfacing genuine, realistic infrastructure issues that were diagnosed and largely resolved live.

---

## Challenges & Troubleshooting

### Incident 1: RDP Access Failure

**Phase:** AWS Infrastructure (Phase 1)

**Issue:** RDP access to DC01 failed unexpectedly.

**Diagnosis:** The security group's RDP rule was scoped to a specific public IP address that had since changed (common with dynamic home/ISP IPs).

**Resolution:** Updated the security group's inbound RDP rule to the current public IP.

**Takeaway:** IP-scoped security group rules need to be revisited whenever the source network changes — a static rule against a dynamic IP is a recurring failure point worth checking first whenever RDP unexpectedly stops working.

### Incident 2: VPN IP Change & SSM-on-DC Limitation

**Phase:** OU Structure & User Creation (Phase 3)

**Issue:** A VPN IP change disrupted access, prompting a fallback attempt to use AWS Systems Manager (SSM) Session Manager to reach DC01 without RDP.

**Diagnosis:** SSM Session Manager failed to establish a session on DC01.

**Resolution / Finding:** SSM Session Manager depends on creating a local `ssm-user` account on the target instance. A Domain Controller has no local SAM database (all accounts are managed via AD), so this account creation fails — meaning **SSM Session Manager cannot be used on a host running the AD DC role** in its default configuration. RDP (or another access path not dependent on a local account) is required for DCs.

**Takeaway:** Not every AWS management tool is compatible with every server role — Domain Controllers, in particular, break the assumption of a local account store that several AWS management features rely on. This limitation resurfaced directly in Incident 3.

### Incident 3: Domain-Join Blocked by Account Lockout Cascade

**Phase:** Windows Client Domain-Join (Phase 7)

Full detailed write-up: [`INCIDENT_3_domain_join_lockout.md`](INCIDENT_3_domain_join_lockout.md)

**Issue:** While joining CLIENT01 to `enterprise.local`, three cascading failures occurred in sequence.

**1. DNS resolution failure**
`nslookup enterprise.local` timed out from CLIENT01 — it was using DHCP-assigned DNS instead of DC01.
**Fix:** Set CLIENT01's Preferred DNS server to DC01's private IP (`10.0.1.45`).

**2. Security Group gap**
With DNS resolving, `ping 10.0.1.45` still showed 100% packet loss. `Enterprise-Server-SG` had no rule permitting internal VPC traffic (ICMP, DNS, Kerberos, LDAP, SMB) between CLIENT01 and DC01 — only external-facing RDP/SSH/HTTP/HTTPS rules existed.
**Fix:** Added an inbound rule permitting traffic from the VPC's internal CIDR range.

**3. Account lockout cascade**
Repeated failed authentication attempts during the join process (including a formatting mistake — the domain name typed into the Computer Name field) tripped the Account Lockout Policy from Phase 5, locking the domain Administrator account. Because DC01's own RDP login relies on that same account, this also **cut off remote access to the domain controller itself**. SSM Session Manager was not a viable fallback, for the same reason identified in Incident 2.
**Resolution path:** Regained access to DC01 via AWS EC2's **Get Windows Password** feature, decrypting the instance's local Administrator password (independent of the domain account) using its key pair.
**Status:** A subsequent domain-join retry re-triggered the lockout before completion — the fix path (unlocking via Active Directory Users and Computers) is identified and ready to execute.

**Takeaway:** Domain-join failures cascade through independent layers — DNS, network reachability, and authentication — each needs isolating rather than assuming the final error message is the root cause. See the full incident file for the complete timeline and additional key takeaways.

---

## Lessons Learned

- **Security groups built access-first can silently block internal service traffic.** Scoping RDP/SSH to specific external IPs is good practice, but internal AD traffic (DNS, Kerberos, LDAP, SMB) needs its own explicit allowance within the VPC's CIDR range — this was the root cause of the trickiest part of Incident 3.
- **Shared credentials across authentication and remote-access paths create a single point of failure.** Using the same domain Administrator account for both AD authentication and RDP login meant one lockout event cut off both directory operations and remote access simultaneously.
- **Not all AWS management tools are DC-compatible.** SSM Session Manager's dependency on a local account conflicts with a Domain Controller's architecture — worth knowing before relying on it as a fallback access method for DCs specifically.
- **Troubleshooting is rarely single-cause.** Incident 3 alone involved three independent, sequential root causes (DNS → network → auth) — isolating each layer methodically was faster than guessing at the final error message.
- **Documentation discipline matters as much as the technical work.** Screenshotting consistently throughout — including the failures — made it possible to reconstruct an accurate, honest incident timeline after the fact, rather than a sanitized version that skips the real troubleshooting.
- **Always screen screenshots for exposed credentials before publishing.** This project's own documentation process caught two real exposed passwords in "Get Windows Password" screenshots before they were published — a good reminder that this step is easy to forget mid-troubleshooting.

---

## Future Improvements

- Complete the CLIENT01 domain-join once the Administrator account lockout clears, and capture the final domain login + drive-mapping verification screenshots
- Move from a single flat security group to more granular, service-specific security groups (e.g., separate rules for AD traffic vs. RDP vs. web)
- Introduce a dedicated "break-glass" local administrator account on DC01, separate from the domain Administrator, to prevent the RDP/domain-auth single point of failure identified in Incident 3
- Add centralized logging (e.g., Windows Event Forwarding or a lightweight SIEM) to capture and correlate authentication failures across DC01 and CLIENT01
- Evaluate AWS Managed Microsoft AD as a comparison point against a self-managed AD DS deployment
- Extend the ticket queue simulation with more complex, multi-step tickets (e.g., cross-department transfers, temporary access grants with expiry)

---

## References

- [Microsoft Learn — Active Directory Domain Services Overview](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/get-started/virtual-dc/active-directory-domain-services-overview)
- [Microsoft Learn — Group Policy Overview](https://learn.microsoft.com/en-us/windows/win32/srvnodes/group-policy)
- [AWS Documentation — Amazon EC2](https://docs.aws.amazon.com/ec2/)
- [AWS Documentation — VPC Security Groups](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html)
- [AWS Documentation — AWS Systems Manager Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager.html)
- [Microsoft Learn — Account Lockout Policy](https://learn.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/account-lockout-policy)

---

## Author

**Ivan Yamoah Boakye**
Security Operations Analyst | IT Support Specialist
Fortinet NSE 4 Certified | Pursuing NSE 5 and SC-200

[LinkedIn](https://www.linkedin.com/in/ivan-yamoah-boakye-70594523a) · [GitHub](https://github.com/iVanny11-tech) · yivan56@gmail.com
