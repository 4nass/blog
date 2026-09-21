---
author: ["Anass"]
title: "Bypassing Intune Conditional Access: The TokenSmith Case Study"
date: "2025-01-02"
description: "A technical analysis of the TokenSmith research, the boundary between Intune compliance and Entra Conditional Access, and the controls that reduce token replay risk."
tags: ["Microsoft Intune", "Microsoft Entra ID", "IAM", "Conditional Access", "OAuth", "Token Security"]
categories: ["Identity & Access Systems"]
topics: ["Identity & Access"]
content_type: "security-analysis"
ShowToc: true
---

Requiring an **Intune-compliant device** in a Microsoft Entra Conditional Access policy sounds like a simple rule: access is allowed only when the endpoint satisfies the organisation's security requirements.

The reality is more nuanced. **Intune evaluates device state. Microsoft Entra evaluates access policy. OAuth clients obtain tokens. Resource providers decide how those tokens are accepted.** The security boundary is therefore distributed across several services, clients and protocols.

The TokenSmith research published by JUMPSEC in December 2024 demonstrated how this boundary could be abused. A specially trusted first-party client used for Intune device enrollment could complete an authentication path from a device that did not satisfy the same device-compliance requirement intended for ordinary cloud access. The resulting token material could then become useful beyond the narrow enrollment scenario, depending on the tenant policy and resource involved.

This is not a universal bypass of every Intune or Conditional Access deployment. It is a case study in **bootstrap exceptions, client trust and token portability**. The hard question is not only whether the initial sign-in was evaluated, but also whether the resulting token remains useful when it moves to another device, client or resource.

<!--more-->

## The short version

- **Intune is not the policy engine:** it reports device compliance to Microsoft Entra.
- **Conditional Access is contextual:** it evaluates a request, its client, resource, identity and available claims.
- **Enrollment needs a bootstrap path:** a new device cannot be compliant before it has completed enrollment and configuration.
- **The research exposed a trust-boundary problem:** an enrollment-oriented first-party OAuth client could be used from outside the expected compliant-device context.
- **The dangerous assumption:** a token issued through a legitimate client is automatically bound to the device that should have received it.
- **The defensive response:** restrict enrollment, separate bootstrap access from resource access, protect tokens where supported, and monitor the complete sign-in chain.

> **Lab safety:** this article does not include the TokenSmith tool, Microsoft client identifiers, redirect URIs, token-redemption requests or commands against Microsoft 365. The demonstration below is a local policy model with no network I/O. Validate controls only in a tenant, test account and device estate that you own.

## At a glance

| Field | Details |
| --- | --- |
| Systems involved | Microsoft Intune, Microsoft Entra ID and Conditional Access |
| Research case | TokenSmith and Intune-compliant-device policy bypass |
| Core issue | Bootstrap client and token trust boundary |
| Preconditions | Valid account authentication and a policy path that accepts the enrollment client |
| Main abuse | Reuse of legitimate token material outside the intended device context |
| Primary impact | Access to resources permitted by the resulting token and tenant policy |
| Not a claim of | A universal bypass of Intune, MFA or every Conditional Access policy |
| Key controls | Enrollment restrictions, token protection, CAE, phishing-resistant MFA and monitoring |

The [JUMPSEC TokenSmith research](https://labs.jumpsec.com/tokensmith-bypassing-intune-compliant-device-conditional-access/) documents the original case study. Microsoft's documentation explains that Intune compliance and Entra Conditional Access work together, while also documenting the enrollment bootstrap behaviour that administrators must account for.

## The four layers people often combine

### 1. Intune device compliance

Intune evaluates whether a managed device satisfies declared requirements such as:

- encryption enabled;
- minimum operating-system version;
- password or PIN requirements;
- endpoint protection settings;
- jailbreak or root detection;
- threat level reported by an integrated security product.

The result is a device-compliance signal. It is not, by itself, an access token and it is not a permanent guarantee about the device's future state.

### 2. Microsoft Entra device identity

Microsoft Entra can distinguish registered, joined and hybrid-joined devices. Depending on the platform and client, the sign-in flow can also use a device certificate or a device-bound authentication artifact.

Device registration, device join and Intune management are related but different states. A device can be known to Entra without being compliant in Intune, and a device can be compliant for a period of time while its security state later changes.

### 3. Conditional Access

Conditional Access evaluates conditions such as:

- user or workload identity;
- target resource;
- client application;
- device platform and device state;
- location and network risk;
- authentication strength;
- user or sign-in risk;
- session controls.

The result is a grant, a block or an additional requirement. It is evaluated in the context of the request and the claims available at that point.

### 4. OAuth tokens and resource providers

The identity provider issues tokens for a client and a resource. The resource provider then validates the token and applies its own supported controls. A token that was issued legitimately is not necessarily proof that every later request comes from the original device.

That last distinction is the heart of the TokenSmith case. A policy may correctly evaluate one authentication flow while still leaving a token-replay or client-trust gap after issuance.

## The enrollment bootstrap problem

A newly provisioned device cannot be marked compliant before it has enrolled, received its management configuration and reported its state. If Conditional Access blocks every request from every non-compliant device—including enrollment itself—users cannot reach the state required to satisfy the policy.

Microsoft documents this bootstrap behaviour explicitly: requiring a compliant device for all users and resources does not block Intune enrollment, and it does not block the limited access needed by Microsoft Authenticator registration.

That exception is operationally necessary. It is also a high-value trust boundary:

1. the enrollment client must be allowed to authenticate;
2. it must receive enough access to complete enrollment;
3. the access must not silently become a general-purpose path to unrelated resources;
4. the resulting credentials and tokens must remain tied to the intended device and client context;
5. the activity must be visible to administrators.

The security question is therefore not “does enrollment have an exception?” It is “how narrowly is that exception scoped, and what can be done with the artifacts it produces?”

## What the TokenSmith research showed

The JUMPSEC case study followed this general path:

1. A tenant required access to cloud resources to come from an Intune-compliant and Microsoft Entra joined device.
2. A test user on a non-compliant system was correctly blocked from ordinary resources.
3. The researchers examined the first-party client used by Company Portal and device enrollment.
4. The enrollment-oriented authentication path was accepted even though the originating environment did not satisfy the ordinary compliant-device requirement.
5. An authorization response was returned to a browser-compatible flow, despite the client normally expecting an application-specific redirect.
6. The resulting token material could be used to request access to resources beyond the narrow enrollment purpose, subject to the tenant's policies and the token's claims.

The research did not demonstrate that every tenant, every resource or every client is exposed in the same way. It demonstrated that the following assumptions are unsafe:

- a successful sign-in log always means every Conditional Access grant control was satisfied;
- a first-party client is automatically limited to the business function suggested by its name;
- a refresh token is harmless because the original device was compliant;
- a device-compliance claim is equivalent to cryptographic token binding;
- enabling one policy for “all cloud apps” removes every alternate authentication path.

## What is actually being bypassed?

The title says “bypassing Intune Conditional Access”, but several different controls can be involved:

### Device compliance

The attacker may obtain a token without presenting the expected compliant-device context. This is the most direct relationship with the policy.

### Device join

The research also discussed cases where a requirement for a Microsoft Entra joined or hybrid-joined device did not stop the alternate client path. Device join and Intune compliance must therefore be validated independently in sign-in records.

### Client restriction

Conditional Access may be scoped to a resource and a client type, but first-party clients and enrollment services can have special behaviour. A policy that only considers browser traffic may not cover native, brokered or device-enrollment flows.

### Token replay

Even when the original authentication is legitimate, a bearer token can be valuable if it is copied to another environment. Device compliance at issuance does not automatically make a bearer token device-bound.

These are separate control planes. Treating them as one “compliant device” checkbox makes testing and incident analysis less reliable.

## The attack scenario

Consider a tenant with the following design:

- all users must use MFA;
- access to Microsoft 365 and administrative resources requires a compliant device;
- new Windows devices use Company Portal to enroll into Intune;
- no explicit control protects the enrollment client from being used outside the intended bootstrap context;
- issued tokens are not protected by a device-bound token control for the target resource.

An abuse path could look like this:

1. Credentials for a valid user are obtained through a separate compromise or the user authenticates voluntarily.
2. The actor starts the legitimate enrollment-oriented sign-in flow from an unmanaged environment.
3. The bootstrap path returns OAuth artifacts because the platform must support new-device enrollment.
4. The actor attempts to use those artifacts for a resource that the tenant believed required a compliant device.
5. The resource accepts the token if its claims and Conditional Access evaluation allow the request.
6. The final impact is limited by the user's permissions, the resource's support for token controls and the tenant's remaining policies.

This is not a password bypass and not an MFA bypass in the same sense as AuthQuake. The actor still needs to complete whatever authentication requirements apply to the path. The risk is that a legitimate bootstrap path can produce a token context that is broader or more portable than the administrator intended.

## Why “shorter token lifetime” is not a complete fix

Short-lived access tokens reduce the time available for replay, but they do not solve an incorrect issuance decision. Refresh tokens can create new access tokens, and an attacker may only need a short session to access sensitive data or establish persistence.

Similarly:

- revoking a session helps after detection but does not prevent the initial issuance;
- Continuous Access Evaluation can react to supported events and resources but does not make every client and resource continuously evaluate device compliance;
- device compliance checks prove a state at a point in the flow but do not cryptographically bind every bearer token to the endpoint;
- MFA protects the account authentication step but does not define the scope of a token afterwards.

These controls are complementary. They should be designed as layers rather than substitutes.

## Safe local demonstration of the trust boundary

The following Python model shows the policy gap without contacting Microsoft, generating OAuth tokens or making any API request. It compares:

- an unsafe design where enrollment is allowed and the resulting token is treated as portable;
- a hardened design where enrollment has limited scope and resource access requires a device-bound token signal.

Save it as **intune-ca-trust-boundary-model.py** and run:

~~~bash
python intune-ca-trust-boundary-model.py
~~~

The command name makes the purpose explicit: it models a trust boundary; it does not reproduce the research exploit.

~~~python
from dataclasses import dataclass


COMMAND_NAME = "intune-ca-trust-boundary-model"


@dataclass(frozen=True)
class Request:
    flow: str
    device_compliant: bool
    mfa_satisfied: bool
    token_device_bound: bool
    bootstrap_token: bool
    resource: str


def unsafe_policy(request: Request) -> str:
    if not request.mfa_satisfied:
        return "DENY: MFA is required"

    if request.flow == "enrollment":
        return "ALLOW: bootstrap token issued"

    if request.bootstrap_token:
        return "ALLOW: portable bootstrap token replayed"

    if request.device_compliant:
        return "ALLOW: compliant-device grant satisfied"

    return "DENY: device is not compliant"


def hardened_policy(request: Request) -> str:
    if not request.mfa_satisfied:
        return "DENY: MFA is required"

    if request.flow == "enrollment":
        return "ALLOW: limited enrollment scope only"

    if not request.device_compliant:
        return "DENY: device is not compliant"

    if not request.token_device_bound:
        return "DENY: device-bound token signal is required"

    return f"ALLOW: protected access to {request.resource}"


bootstrap = Request(
    flow="enrollment",
    device_compliant=False,
    mfa_satisfied=True,
    token_device_bound=False,
    bootstrap_token=False,
    resource="enrollment-service",
)

replayed = Request(
    flow="resource",
    device_compliant=False,
    mfa_satisfied=True,
    token_device_bound=False,
    bootstrap_token=True,
    resource="mail-and-files",
)

compliant_device = Request(
    flow="resource",
    device_compliant=True,
    mfa_satisfied=True,
    token_device_bound=True,
    bootstrap_token=False,
    resource="mail-and-files",
)

print(COMMAND_NAME)
print("unsafe bootstrap:", unsafe_policy(bootstrap))
print("unsafe replay   :", unsafe_policy(replayed))
print("hardened replay :", hardened_policy(replayed))
print("hardened access :", hardened_policy(compliant_device))
~~~

Expected output:

~~~text
intune-ca-trust-boundary-model
unsafe bootstrap: ALLOW: bootstrap token issued
unsafe replay   : ALLOW: portable bootstrap token replayed
hardened replay : DENY: device is not compliant
hardened access : ALLOW: protected access to mail-and-files
~~~

The model intentionally keeps the result simple. Its purpose is to show that a bootstrap allowance and resource access must be evaluated as different decisions. In a real deployment, the identity provider, client, device state, token claims and resource provider all contribute to that decision.

## Validation plan for an authorised tenant

Use a dedicated test user, test device and non-production resource. Do not use TokenSmith, copied tokens or production accounts as a first validation step.

### Map the intended policy

Document:

1. which users and groups are covered;
2. which resources are covered;
3. which client applications and authentication flows are possible;
4. where enrollment and registration are intentionally excepted;
5. what device claim is expected at issuance;
6. what controls the resource provider enforces after issuance;
7. how tokens and sessions are revoked after a test compromise.

### Test the bootstrap boundary

- Start with a clean test device that is not yet compliant.
- Verify that enrollment succeeds only through the intended Company Portal and broker path.
- Verify that enrollment does not grant access to unrelated applications or administrative APIs.
- Verify that a new browser or native-client flow does not inherit the enrollment exception.
- Confirm that the sign-in record identifies the expected client and resource.

### Test token portability safely

Do not copy live tokens. Instead, validate the policy through Microsoft Entra sign-in logs, Conditional Access report-only results and a harmless test resource:

- a non-compliant test device must remain blocked from the protected resource;
- a compliant device must be allowed only when the expected claims are present;
- a change from compliant to non-compliant must produce the expected resource behaviour;
- session revocation must invalidate access within the documented service window;
- a client or resource that does not support a token control must be explicitly recorded as an exception.

The objective is to verify the decision path and its observability, not to obtain an unauthorized token.

## Detection and investigation

The strongest signal is a mismatch between identity, client, device and resource:

- a successful sign-in from a device that has no corresponding compliant record;
- enrollment-client activity followed by access to unrelated resources;
- a client or redirect pattern inconsistent with the user's normal platform;
- an access token issued for an enrollment-oriented flow followed by Graph, mail, files or administrative activity;
- the same user appearing on a managed endpoint and an unmanaged environment within a short interval;
- token issuance or refresh activity after the device has been retired, wiped or marked non-compliant;
- new application consent, registered device, mailbox rule or role assignment after the event.

Review, together:

1. Microsoft Entra interactive and non-interactive sign-in logs;
2. the Conditional Access and report-only result for each event;
3. Intune device inventory, compliance history and enrollment events;
4. Microsoft 365 audit activity for the affected resource;
5. endpoint telemetry for credential or token access;
6. network and proxy records where the sign-in originated.

Do not conclude “Conditional Access was bypassed” from a successful sign-in alone. Inspect which policies applied, which were not applied, which claims were present and which resource ultimately accepted the token.

## Hardening priorities

### 1. Treat enrollment as a constrained bootstrap zone

Allow the minimum access required to enroll and register a device. Keep enrollment separate from general resource access:

- restrict who may enroll devices;
- restrict supported platforms and ownership types;
- limit the number of devices per user;
- require MFA or an appropriate authentication strength for enrollment;
- use dedicated pilot groups before broad rollout;
- keep administrative and high-impact resources outside the bootstrap exception;
- review the exception whenever the enrollment architecture changes.

Microsoft's requirement for a compliant device should be tested in report-only mode first, then enabled progressively. A policy that is technically correct but blocks legitimate enrollment creates pressure for unsafe exclusions.

### 2. Use token protection where it is supported

Microsoft Entra Token Protection attempts to reduce token replay by accepting device-bound sign-in session tokens for supported applications and platforms. It is not universal: support depends on the operating system, client and resource.

Deploy it in a pilot, capture interactive and non-interactive sign-in logs, and document unsupported resources. A policy that protects Exchange Online or SharePoint does not automatically protect every custom application or browser flow.

### 3. Enable and understand Continuous Access Evaluation

Continuous Access Evaluation can help resource providers react to events such as password changes, account disablement, MFA changes, explicit token revocation and elevated risk. It is valuable for reducing the lifetime of a compromised session, but support varies by client and resource.

Use CAE as a revocation and policy-refresh layer, not as proof that every request continuously carries a fresh Intune compliance decision.

### 4. Make Conditional Access outcomes observable

Use report-only policies, policy impact views and sign-in logs to answer:

- which policy applied?
- which grant control was satisfied?
- which policy was not applied?
- which client and resource were involved?
- was the event interactive or non-interactive?
- was a compliant-device or device-bound claim present?

Keep a baseline for enrollment-client activity. A first-party client should not become invisible simply because it is trusted.

### 5. Reduce token theft at the endpoint

Protect the device where tokens are created and used:

- keep Windows, browsers, Company Portal and security agents current;
- use phishing-resistant authentication for privileged users;
- apply endpoint protection and attack-surface-reduction controls;
- restrict local administrator access;
- monitor suspicious browser, broker and token-store access;
- revoke sessions and refresh tokens quickly after suspected theft.

Token controls cannot compensate for an endpoint that is already under an attacker's control, but they can reduce the value of copied artifacts.

### 6. Separate users, workloads and emergency access

User-scoped Conditional Access policies do not automatically protect service principals and workload identities. Apply workload-identity controls separately, prefer managed identities where appropriate, and protect emergency accounts with an explicit tested design.

An emergency account should bypass only the minimum policies necessary for recovery, be strongly protected, and generate high-priority alerts.

## Incident response

If a device-compliance or token-trust boundary may have been abused:

1. revoke the user's sessions and refresh tokens;
2. disable or isolate the suspected device and preserve endpoint evidence;
3. reset credentials and require phishing-resistant reauthentication where possible;
4. review registered devices, authentication methods, application consents and role changes;
5. inspect mail, files, Teams, Graph and administrative activity;
6. check whether other accounts used the same client or enrollment path;
7. tighten the bootstrap policy before restoring access;
8. record which resources do and do not support token protection or CAE.

The investigation should establish whether the event involved a stolen password, a stolen session, a legitimate enrollment flow, a mis-scoped policy or a combination of these conditions.

## What this case study teaches

The lesson is not that Intune compliance is useless or that Conditional Access can always be bypassed. The lesson is that **device compliance is a claim in a larger authentication system**.

A robust design asks five separate questions:

- How does a device become trusted in the first place?
- Which client is allowed to perform that bootstrap?
- What resource scope does the resulting token carry?
- Is the token cryptographically tied to the device?
- Can the organisation detect and revoke the session when context changes?

Conditional Access is strongest when bootstrap exceptions are narrow, token use is bound to the intended context, resource coverage is explicit and every decision is visible in logs. The checkbox is only the beginning of the design.

## Sources

1. [JUMPSEC Labs — TokenSmith: Bypassing Intune Compliant Device Conditional Access](https://labs.jumpsec.com/tokensmith-bypassing-intune-compliant-device-conditional-access/)
2. [Microsoft Learn — Require device compliance with Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/policy-all-users-device-compliance)
3. [Microsoft Learn — Token Protection in Microsoft Entra Conditional Access](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-token-protection)
4. [Microsoft Learn — Continuous access evaluation in Microsoft Entra](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-continuous-access-evaluation)
5. [Microsoft Learn — Conditional Access policy insights and report-only mode](https://learn.microsoft.com/en-us/entra/identity/conditional-access/concept-conditional-access-report-only)
6. [RFC 9700 — Best Current Practice for OAuth 2.0 Security](https://datatracker.ietf.org/doc/html/rfc9700)
