# Incident 3: Domain-Join Blocked by Account Lockout Cascade

**Project:** Enterprise Active Directory Infrastructure — Deployment, Administration & Incident Response
**Phase:** Windows Client Domain-Join (CLIENT01 → enterprise.local)

## Summary

While joining CLIENT01 to the `enterprise.local` domain, a chain of cascading issues surfaced — a DNS misconfiguration, an incomplete security group rule set, and an account lockout that temporarily severed RDP access to the domain controller itself.

## Timeline

### 1. DNS Resolution Failure

The initial domain-join attempt failed with:

> "An Active Directory Domain Controller (AD DC) for the domain 'enterprise.local' could not be contacted."

**Diagnosis:** Ran `nslookup enterprise.local` from CLIENT01 — request timed out. CLIENT01's network adapter was using default DHCP-assigned DNS rather than DC01, so it had no way to resolve AD-integrated DNS records.

**Fix:** Set CLIENT01's IPv4 Preferred DNS server to DC01's private IP (`10.0.1.45`), while leaving the IP address itself on DHCP.

### 2. Security Group Gap

After the DNS fix, `nslookup` succeeded, but `ping 10.0.1.45` returned 100% packet loss — confirming a network-layer block rather than a DNS problem.

**Diagnosis:** Reviewed the `Enterprise-Server-SG` inbound rules and found only RDP, SSH, HTTP, and HTTPS permitted, each scoped to specific external IPs or `0.0.0.0/0`. No rule allowed internal VPC traffic — ICMP, DNS, Kerberos, LDAP, SMB — between CLIENT01 and DC01.

**Fix:** Added an inbound rule permitting traffic from the VPC's internal CIDR range. Re-verified with `nslookup` and `ping` — both succeeded (0% loss), confirming DC01 was now reachable at the network layer.

### 3. Account Lockout Cascade

With connectivity confirmed, the domain-join was retried. A credential mismatch — including one attempt where `enterprise.local` was mistakenly typed into the Computer Name field instead of the Domain field — led to repeated failed authentication attempts. This tripped the Account Lockout Policy configured earlier in the project (Phase 5.6), locking the domain `Administrator` account.

**Side effect:** Because DC01's own RDP login also relies on that same domain Administrator account, the lockout severed remote access to the domain controller.

**SSM Session Manager was not a viable fallback.** Attempting to connect returned:

> "Unable to start command: Failed to create user ssm-user: Instance is running active directory domain controller service. Disable the service to continue to use session manager."

This is consistent with the same platform limitation identified in Incident 2 — SSM cannot create its managed local user account on a host running the AD DC role.

**Resolution path:** Regained access to DC01 via the AWS EC2 Console's **Connect → Get Windows Password** feature, decrypting the instance's original local Administrator password using its key pair. This account authenticates independently of the domain, bypassing the lockout entirely. Successfully logged into DC01.

**Status:** A subsequent domain-join retry from CLIENT01 re-triggered the account lockout before the join could complete. The fix path — unlocking the account via Active Directory Users and Computers on DC01 — was identified but not yet executed at time of writing.

## Key Takeaways

- **Domain-join failures cascade through layers.** DNS, network reachability, and authentication are independent failure points — isolate each one before assuming the final error message is the root cause.
- **Security groups built access-first can silently block internal AD traffic.** Scoping RDP/SSH to specific external IPs is good practice, but internal service-to-service traffic (DNS, Kerberos, LDAP, SMB) needs its own explicit allowance within the VPC's CIDR range.
- **Shared credentials between AD and RDP access create a single point of failure.** A lockout event on a domain account doesn't just block directory operations — it can cut off remote access entirely. This reinforces the value of maintaining a local admin credential path (or a dedicated break-glass account) as a fallback that doesn't depend on domain authentication.
- **SSM Session Manager cannot be used on Active Directory Domain Controllers** in its default configuration, since it depends on creating a local `ssm-user` account, which conflicts with a DC's lack of a local SAM database. RDP (or a break-glass local account) remains the reliable access path for DCs.

## Tools & Commands Used

```powershell
nslookup enterprise.local
ping 10.0.1.45
```

- AWS EC2 Console → Security Groups → Inbound Rules
- AWS EC2 Console → Connect → Get Windows Password
- Active Directory Users and Computers (`dsa.msc`) → Account unlock
