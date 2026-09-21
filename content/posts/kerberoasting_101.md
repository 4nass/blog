---
author: ["Anass"]
title: "Kerberoasting 101: Service Tickets, Detection and Hardening"
date: "2026-01-15"
description: "A defensive introduction to Kerberoasting in Active Directory, covering SPNs, service tickets, offline password guessing, detection and safer service-account design."
tags: ["Kerberoasting", "Active Directory", "Kerberos", "Windows Security", "Service Accounts"]
categories: ["Identity & Access Systems"]
topics: ["Security"]
content_type: "field-lab"
ShowToc: true
---

Kerberoasting is an Active Directory technique that turns a normal Kerberos service-ticket request into an offline password-guessing opportunity.

The attacker does not need to compromise a domain controller, send repeated passwords to a login form or exploit a software vulnerability. A domain principal that can authenticate may be able to request a service ticket for an account associated with a Service Principal Name (SPN). Part of that ticket is encrypted with a key derived from the service account's secret. The attacker can then take the ticket away from the domain and attempt password guesses offline.

The risk is determined less by the existence of Kerberos than by the identity behind the SPN:

- a human-managed service account with a weak, old password is a high-risk target;
- a privileged service account can turn one recovered password into a major compromise;
- a gMSA with a long, randomly managed password is much harder to attack with password guessing;
- an RC4 ticket deserves urgent attention, but AES does not make a weak service-account password safe.

<!--more-->

## The short version

- **Target:** a service account or other principal with an SPN.
- **Precondition:** an authenticated domain identity able to request a service ticket.
- **Technique:** request a TGS, then perform password guessing offline.
- **Why it is dangerous:** the domain controller does not see the offline guesses.
- **Important telemetry:** Event ID 4769 on domain controllers.
- **Priority signal:** unusual TGS volume, RC4 usage, unusual requester or privileged service account.
- **Best fix:** use gMSAs where possible, remove unnecessary privileges and rotate strong secrets.
- **Common mistake:** relying on account lockout or ticket lifetime to stop offline cracking.

> **Lab safety:** this article does not include ticket extraction, password-cracking commands, wordlists or a procedure against a real domain. The demonstration uses synthetic events only. Test Active Directory controls in a disposable domain that you own.

## At a glance

| Field | Details |
| --- | --- |
| Technique | Kerberoasting |
| MITRE ATT&CK | T1558.003 |
| Required access | Authenticated domain identity |
| Directory object | Principal with a Service Principal Name |
| Requested artifact | Kerberos Ticket-Granting Service ticket |
| Password testing | Offline, away from the domain controller |
| High-risk conditions | Weak secret, RC4 ticket, privileged service account |
| Main detection | Event ID 4769 with behavioural baselining |
| Strongest mitigation | gMSA, least privilege and strong key rotation |

[MITRE ATT&CK T1558.003](https://attack.mitre.org/techniques/T1558/003/) describes Kerberoasting as the abuse of TGS tickets for offline brute-force attacks. Microsoft's documentation for [Event 4769](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4769) explains that a record is generated when a domain controller receives a Kerberos service-ticket request.

## Kerberos in one minute

Kerberos separates the proof of a user's identity from access to a particular service:

1. The client authenticates to the Key Distribution Center (KDC).
2. The KDC issues a Ticket-Granting Ticket (TGT).
3. The client presents the TGT when requesting access to a service.
4. The KDC returns a Ticket-Granting Service (TGS) ticket for the requested SPN.
5. The client presents the service ticket to the target service.

The SPN identifies a service instance, for example an HTTP application, SQL Server service or file service. Active Directory stores the SPN on the account that runs that service.

The TGS is useful to the attacker because its encrypted service portion can be tested against password guesses without asking the KDC to validate each guess. The exact cryptographic construction depends on the ticket encryption type:

- RC4-HMAC commonly uses the service account's NT hash as the key;
- AES uses keys derived from the account secret and realm information;
- a weak password remains weak even when the ticket uses AES.

Kerberoasting is therefore an interaction between directory configuration, service-account hygiene and Kerberos encryption policy.

## Why an SPN creates a target

An SPN is not a vulnerability. It is required for Kerberos to locate a service identity. The exposure appears when an SPN is attached to a manually managed user account with a password that is:

- short or predictable;
- shared between environments;
- rarely rotated;
- known by too many administrators;
- reused for several services;
- attached to privileged groups;
- retained after the service has been retired.

The directory may contain legitimate reasons for a user account to own an SPN. The correct response is not to delete every SPN. It is to understand the service identity, reduce its privileges and use an automatically managed secret where the service supports it.

## The attack scenario

Consider an internal web service running under a conventional domain user:

1. The account has an SPN for the service.
2. The account password has not been rotated for several years.
3. The account has more permissions than the service requires.
4. An already authenticated user requests a service ticket for that SPN.
5. The ticket is collected and tested offline against candidate passwords.
6. A recovered password is used as a valid account credential.
7. The attacker gains whatever access the service account has, potentially including lateral movement or privilege escalation.

The domain controller sees a normal-looking TGS request. It does not observe the subsequent offline guesses. The high-value detection opportunity is therefore the ticket request and the account context, not a stream of failed password events.

Kerberoasting also does not automatically produce Domain Admin access. The final impact depends on the recovered account's group memberships, local rights, service permissions, delegation settings and reachable systems.

## What makes a ticket request suspicious?

Event 4769 is generated frequently in a functioning domain. A single request is not evidence of an attack, and a high count can be normal for application servers, file servers, monitoring systems or service meshes.

Useful signals become stronger when combined:

- a user or workstation requests tickets for many distinct SPNs in a short interval;
- the requester has never used those services before;
- the requester is an interactive desktop account rather than an expected application host;
- tickets use RC4 where the service is capable of AES;
- a privileged service account has a long password age;
- an SPN points to a user account rather than a managed service account;
- requests originate from an unusual host, subnet or logon type;
- the target services span unrelated applications or servers;
- the same source requests tickets across several domains or trust paths.

Do not alert on RC4 alone. Legacy applications can still require it, and disabling it without compatibility analysis can break authentication. Treat RC4 as an investigation and migration priority.

Machine accounts ending in a dollar sign also request service tickets routinely. They should not be treated as malicious solely because they appear in Event 4769.

## RC4 versus AES

The ticket encryption type changes the cost of offline password guessing, but neither choice repairs poor identity design.

Microsoft identifies the common Kerberos encryption values as:

| Value | Encryption type | Practical interpretation |
| --- | --- | --- |
| 0x11 | AES128-CTS-HMAC-SHA1-96 | Modern Kerberos encryption |
| 0x12 | AES256-CTS-HMAC-SHA1-96 | Modern Kerberos encryption |
| 0x17 | RC4-HMAC | Legacy encryption; prioritise migration |

Moving a service from RC4 to AES is worthwhile, but it is not a standalone Kerberoasting fix. An attacker can still request an AES-encrypted ticket and test guesses offline. The durable fix is a high-entropy secret that is rotated automatically or frequently.

Before disabling RC4:

1. inventory Event 4769 encryption types;
2. identify applications and operating systems that still request RC4;
3. update or replace incompatible services;
4. test authentication and delegation flows;
5. disable legacy encryption in a staged policy;
6. monitor failures and roll out progressively.

See Microsoft's [RC4 detection and remediation guidance](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos) for the current Windows Server approach.

## Why gMSAs change the economics

A group Managed Service Account (gMSA) is a domain identity whose password is generated and managed by Windows. Microsoft describes gMSA passwords as long, randomly generated values that rotate automatically, commonly every 30 days.

gMSAs reduce the main assumptions that make Kerberoasting effective:

- administrators do not need to invent a password;
- the password does not need to be copied manually to every service host;
- regular rotation does not depend on a calendar reminder;
- the account can be restricted to approved computers;
- the account can be removed from privileged groups and scoped to the service.

gMSA is not a magic security boundary. It still needs:

- a controlled list of hosts allowed to retrieve the managed password;
- least-privilege directory and application permissions;
- correct SPN ownership;
- monitoring of service-account use;
- testing for every service instance and failover path.

Use a standalone managed service account where a service runs on one host and a gMSA where multiple supported hosts share the identity.

## Safe local demonstration of Kerberoasting detection

The following Python program models a detector using invented Event 4769-like records. It does not contact a domain controller, request a ticket or crack a password.

The detector combines volume, distinct service count, RC4 use and requester type. The thresholds are illustrative; each environment needs its own baseline.

Save it as **kerberos-service-ticket-anomaly-model.py** and run:

~~~bash
python kerberos-service-ticket-anomaly-model.py
~~~

~~~python
from collections import Counter


COMMAND_NAME = "kerberos-service-ticket-anomaly-model"


events = [
    {
        "requester": "alice",
        "source": "WORKSTATION-07",
        "service": "HTTP/intranet.example.test",
        "encryption": "AES256",
        "interactive": True,
    },
    {
        "requester": "alice",
        "source": "WORKSTATION-07",
        "service": "MSSQLSvc/sql.example.test",
        "encryption": "RC4",
        "interactive": True,
    },
    {
        "requester": "alice",
        "source": "WORKSTATION-07",
        "service": "CIFS/fileserver.example.test",
        "encryption": "RC4",
        "interactive": True,
    },
    {
        "requester": "alice",
        "source": "WORKSTATION-07",
        "service": "HTTP/legacy.example.test",
        "encryption": "RC4",
        "interactive": True,
    },
]


by_requester = {}
for event in events:
    by_requester.setdefault(event["requester"], []).append(event)


print(COMMAND_NAME)

for requester, tickets in by_requester.items():
    services = {ticket["service"] for ticket in tickets}
    rc4_count = sum(ticket["encryption"] == "RC4" for ticket in tickets)
    sources = Counter(ticket["source"] for ticket in tickets)
    interactive = any(ticket["interactive"] for ticket in tickets)

    suspicious = (
        len(services) >= 3
        and rc4_count >= 2
        and interactive
    )

    print(
        f"{requester}: services={len(services)} "
        f"rc4={rc4_count} source={sources.most_common(1)[0][0]} "
        f"alert={suspicious}"
    )
~~~

Expected output:

~~~text
kerberos-service-ticket-anomaly-model
alice: services=4 rc4=3 source=WORKSTATION-07 alert=True
~~~

This model is intentionally small. A production detector should add time windows, account type, service-account privilege, expected application hosts, encryption capability and historical baselines. It should also suppress known service-to-service patterns rather than alerting on every burst.

## Read-only inventory of SPNs

Start with an inventory, not an attack simulation. On a domain-joined administrative system with the Active Directory module:

~~~powershell
Get-ADUser -LDAPFilter "(servicePrincipalName=*)" -Properties servicePrincipalName,msDS-SupportedEncryptionTypes,PasswordLastSet,MemberOf | Select-Object SamAccountName,PasswordLastSet,msDS-SupportedEncryptionTypes,MemberOf,servicePrincipalName
~~~

For managed service accounts:

~~~powershell
Get-ADServiceAccount -Filter * -Properties ServicePrincipalNames,PrincipalsAllowedToRetrieveManagedPassword,msDS-SupportedEncryptionTypes | Select-Object Name,ObjectClass,ServicePrincipalNames,PrincipalsAllowedToRetrieveManagedPassword,msDS-SupportedEncryptionTypes
~~~

For a specific service name, the read-only setspn query is:

~~~powershell
setspn -Q HTTP/app.example.test
~~~

The inventory should answer:

- which account owns each SPN;
- which service and hosts use it;
- whether the account is human-managed or managed by Windows;
- when its password last changed;
- which groups and resources it can access;
- which encryption types it uses;
- whether the service is still active.

Do not export password material, ticket data or sensitive directory attributes into a shared spreadsheet.

## Monitoring Event 4769

Enable success and failure auditing for Kerberos Service Ticket Operations on domain controllers, then forward the events to a SIEM. Microsoft notes that the event volume is high, so central collection and filtering are more useful than ad hoc searches on one controller.

A safe first inspection on a domain controller is:

~~~powershell
Get-WinEvent -FilterHashtable @{LogName="Security"; Id=4769} -MaxEvents 100 | Select-Object TimeCreated,Id,ProviderName,Message
~~~

For a production analytics rule, extract structured fields from the event XML instead of relying on a fixed message line or property index. Retain at least:

- requester account and domain;
- service name and service account;
- client address;
- ticket encryption type;
- advertised encryption types;
- timestamp and domain controller;
- failure code;
- logon and device context where available.

Build a baseline by requester type:

- user workstations;
- application servers;
- domain controllers;
- service accounts;
- scheduled tasks;
- monitoring and backup platforms.

Then alert on deviations from the expected service graph. A server requesting its known database and file-service tickets is normal. A desktop user suddenly requesting tickets for many unrelated SPNs is not automatically malicious, but it deserves context.

## Hardening priorities

### 1. Move supported services to gMSA

Use gMSAs for Windows services, IIS application pools, scheduled tasks and supported server farms. Restrict which computer accounts may retrieve the password and remove unnecessary group memberships.

Microsoft's [gMSA guidance](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts) provides the current prerequisites and management workflow.

### 2. Remove privilege from service identities

A service account should not be Domain Admin merely because an application installer requested it. Replace broad group membership with the minimum rights required by the application:

- database roles instead of domain-wide administration;
- filesystem ACLs on specific paths;
- delegated directory permissions where needed;
- local service rights only on approved hosts;
- separate identities for separate trust zones.

Review nested group memberships, local administrator rights, delegation settings and access to secrets.

### 3. Rotate human-managed service passwords

For services that cannot use gMSA:

- use a long random secret from an approved vault;
- rotate it on a defined schedule;
- update every service instance atomically;
- restrict interactive logon;
- deny use from unapproved hosts;
- monitor password age and ownership;
- retire the account when the service is removed.

Do not use seasonal passwords, company names, product names or passwords shared across environments.

### 4. Prefer AES and retire RC4

Enable AES support for compatible services, confirm that the account has current AES keys and stage the removal of RC4. Changing the encryption type requires compatibility testing; it does not replace password hygiene.

### 5. Reduce SPN sprawl

Remove stale SPNs and correct duplicate ownership. An SPN for a decommissioned service is still directory configuration that needs an owner and a review decision.

### 6. Protect the directory and service hosts

Use tiered administration, protected administrative workstations, endpoint detection, constrained delegation where required and strong controls around directory replication and service deployment. Kerberoasting is often one step in a larger identity attack, not the complete intrusion.

## What does not fix Kerberoasting

Several commonly proposed controls address the wrong part of the attack:

- **Account lockout:** offline password guesses never reach the domain controller.
- **Shorter ticket lifetime:** it may reduce online session duration, but it does not invalidate a ticket already collected for offline analysis.
- **MFA on an administrator's interactive login:** useful, but it does not automatically protect a service account or every protocol that uses it.
- **AES alone:** stronger ticket encryption raises cost, but a weak service password remains a target.
- **Deleting an account from one group:** verify all nested groups, local rights, service permissions and SPNs.

The durable control is a managed, high-entropy service identity with narrowly scoped privileges.

## Validation plan for an authorised lab

Use a disposable forest with one test service and synthetic data:

1. inventory the SPN and the account that owns it;
2. record its group memberships and password-management model;
3. verify the expected Event 4769 baseline for normal service use;
4. migrate the service to a gMSA if supported;
5. confirm the service continues to authenticate after password rotation;
6. confirm only approved hosts can use the gMSA;
7. verify that the SIEM identifies unexpected ticket requests;
8. confirm that removing the service removes its SPN and access rights.

The lab should validate inventory, rotation, permissions and detection. There is no need to extract a ticket or attempt password recovery to demonstrate that the controls work.

## Incident response

If a service-account password may have been recovered:

1. identify every SPN and service using the account;
2. reset or rotate the account secret immediately;
3. update all dependent services and hosts;
4. remove unnecessary group memberships and local rights;
5. revoke active sessions and review Kerberos and application logs;
6. search for new logons, remote-service use and directory changes;
7. inspect the source hosts for credential-access activity;
8. replace the account with a gMSA where possible;
9. document whether the account was used for delegation, scheduled tasks or automation.

Resetting the password is necessary, but it is not the end of the investigation. A recovered service credential may have been used to access file shares, databases, APIs or other systems before the rotation.

## What Kerberoasting teaches

Kerberoasting is a good example of an attack that exploits normal protocol behaviour and weak identity lifecycle management:

- the TGS request is legitimate;
- the password testing happens outside the domain;
- the service identity often outlives its original owner;
- excessive privileges turn one secret into lateral movement;
- the strongest mitigation is operational rather than exotic.

The practical questions are:

- who owns this SPN?
- how is the password generated and rotated?
- what can the service account access?
- which encryption types are actually used?
- does the request come from an expected host?
- can the account be replaced with a gMSA?

Secure service identities, remove unnecessary privilege, retire RC4 carefully and baseline Event 4769. That combination addresses the attack without weakening Kerberos or treating every service-ticket request as an incident.

## Sources

1. [MITRE ATT&CK — Kerberoasting (T1558.003)](https://attack.mitre.org/techniques/T1558/003/)
2. [Microsoft Learn — Event 4769: A Kerberos service ticket was requested](https://learn.microsoft.com/en-us/previous-versions/windows/it-pro/windows-10/security/threat-protection/auditing/event-4769)
3. [Microsoft Learn — Detect and remediate RC4 usage in Kerberos](https://learn.microsoft.com/en-us/windows-server/security/kerberos/detect-remediate-rc4-kerberos)
4. [Microsoft Learn — Manage group Managed Service Accounts](https://learn.microsoft.com/en-us/windows-server/identity/ad-ds/manage/group-managed-service-accounts/group-managed-service-accounts/manage-group-managed-service-accounts)
5. [Microsoft Learn — Secure group Managed Service Accounts](https://learn.microsoft.com/en-us/entra/architecture/service-accounts-group-managed)
