---
author: ["Anass"]
title: "Evilginx2 & Session Hijacking: The AitM Relay Problem"
date: "2026-02-18"
description: "A defensive analysis of Evilginx2-style adversary-in-the-middle attacks, authenticated-session theft and the controls that detect and contain token replay."
tags: ["Evilginx2", "Session Hijacking", "AitM", "Phishing", "FIDO2", "Token Security"]
categories: ["Security"]
topics: ["Security"]
content_type: "field-lab"
ShowToc: true
---

Multi-factor authentication protects an authentication transaction. It does not automatically protect the authenticated session that follows.

An **adversary-in-the-middle (AitM)** proxy such as Evilginx2 places an attacker-controlled relay between a victim and the legitimate identity provider. The victim may see the real authentication flow and complete the password and MFA steps, while the proxy observes the conversation and attempts to capture the resulting session state.

The result is often described as an “MFA bypass”. That description is imprecise. In the common case, the identity provider did authenticate the user and did evaluate MFA. The attacker stole or relayed the authenticated state after that decision, then attempted to reuse it from another context.

This article explains the attack boundary and the defensive controls without reproducing a phishing kit. It does not contain Evilginx2 installation instructions, phishlets, lure URLs, credential capture, cookie export or token replay.

<!--more-->

## The short version

- **Evilginx2-style attack:** a reverse proxy relays a real sign-in flow through an attacker-controlled origin.
- **What is captured:** credentials may be exposed, but the most valuable artifact is often the authenticated session or token.
- **Why ordinary MFA may not stop it:** a completed session can be reused without starting authentication again.
- **What does stop many AitM flows:** origin-bound, phishing-resistant methods such as FIDO2/WebAuthn and passkeys.
- **What a valid certificate proves:** only that the browser trusts the certificate for that domain; it does not prove that the domain is legitimate.
- **Main defensive objective:** detect context mismatch, revoke sessions quickly and prevent bearer-token replay where possible.

> **Lab safety:** this article does not provide a phishing infrastructure, target-specific proxy configuration, credentials, cookies, tokens or instructions to test a real identity provider. The demonstration below uses synthetic session metadata and has no network I/O. Run any validation only against an application, account and domain that you own.

## At a glance

| Field | Details |
| --- | --- |
| Threat class | Adversary-in-the-middle and authenticated-session theft |
| Common tool reference | Evilginx2-style reverse proxy |
| Primary target | Credentials and post-authentication session state |
| MFA limitation | MFA may be completed before the session is captured |
| Typical impact | Impersonation within the session's scope and lifetime |
| Strongest preventive control | Phishing-resistant, origin-bound authentication |
| Detection focus | Origin, device, network and session-context mismatch |
| Containment | Revoke sessions and tokens, reset credentials and investigate endpoint |

[MITRE ATT&CK T1557](https://attack.mitre.org/techniques/T1557/) describes adversary-in-the-middle positioning, while [T1539](https://attack.mitre.org/techniques/T1539/) covers the theft and reuse of web session cookies. [Okta's threat research](https://sec.okta.com/articles/phishingasaservice/) describes the operational pattern of real-time phishing proxies targeting enterprise identity flows.

## What Evilginx2 is—and is not

Evilginx2 is an open-source reverse-proxy framework that can be configured to relay selected web authentication flows. In an AitM campaign, the attacker controls the public-facing origin while the legitimate identity provider remains the backend.

The proxy can potentially:

- present a look-alike URL to the victim;
- relay requests and responses in real time;
- observe credentials submitted through the relay;
- observe the result of an MFA interaction;
- capture an application session when it is established;
- forward real content so the browser experience appears convincing.

The proxy does not magically break the cryptography of FIDO2 or decrypt a properly protected token. It abuses the fact that a browser can be persuaded to authenticate through the wrong origin and that some authentication methods issue a bearer-like session after completion.

It is also important not to confuse the tool with the attack. The same AitM pattern can be implemented with other frameworks, commercial phishing services or custom code. Detection should focus on the behaviour and the resulting session anomalies, not only on a process name.

## The AitM flow

The attack path can be represented without exposing any operational details:

~~~text
Victim browser  --->  Attacker-controlled relay  --->  Legitimate IdP
      |                       |                         |
      |   credentials + MFA  |   forwards traffic      |
      |---------------------->|------------------------>|
      |                       |<------------------------|
      |   authenticated page  |   observes session      |
      |<----------------------|                         |
      |                                                 |
      |                 Attacker reuses session elsewhere
~~~

The security boundary changes during the flow:

1. The victim begins on an attacker-controlled origin.
2. The relay forwards the browser's requests to the genuine identity provider.
3. The victim completes the identity provider's authentication requirements.
4. The identity provider issues an authenticated result.
5. The relay attempts to retain the resulting state.
6. The attacker uses the state against the relying party while it remains valid.

The important event is not only the initial MFA success. It is the later use of the session from a context that does not match the context in which it was created.

## Why ordinary MFA can arrive too late

MFA is evaluated when the identity provider receives an authentication request. Many applications then create a session that is accepted for subsequent requests until it expires, is revoked or fails another policy check.

If an attacker obtains that session after the MFA step:

- the attacker may not need the password again;
- the attacker may not need the MFA device again;
- the attacker may not trigger another Conditional Access decision;
- the relying party may see only a valid session identifier or bearer token.

This is not a criticism of MFA. It is a reminder that authentication, session management and authorization are separate controls.

The risk is lower when:

- the session is short-lived and server-side revocable;
- sensitive actions require reauthentication;
- tokens are sender-constrained or device-bound;
- the application participates in continuous evaluation;
- the relying party checks device and risk context;
- phishing-resistant authentication prevents the relay from completing the original ceremony.

## What the attacker needs

An AitM session-theft attack usually requires a combination of:

- a convincing delivery channel;
- a domain or link the victim will open;
- a relay that can handle the target authentication flow;
- a user willing to continue the sign-in;
- an authentication method whose result can be relayed or captured;
- a relying party that accepts the resulting session without a fresh context check.

This is why the attack is not equivalent to “Evilginx bypasses everything”. The outcome depends on the identity provider, authentication method, browser, relying party, session format, device controls and user behaviour.

## Certificates are not an authenticity signal

The previous version of this article suggested that Evilginx uses self-signed certificates. That is not a reliable detection rule.

An attacker-controlled domain can obtain a valid certificate from a public certificate authority. The browser may display a normal lock icon even though the hostname is fraudulent. Certificate validity proves domain control for TLS purposes; it does not prove that the domain belongs to Microsoft, Okta, Google or the organisation being impersonated.

Useful URL checks are about the complete origin:

- scheme;
- registrable domain;
- subdomain;
- path;
- tenant or organisation context;
- spelling and look-alike characters;
- whether the link was expected.

Certificate issuer, age or encryption status can enrich a detection model, but they should not be treated as proof of legitimacy.

## The role of phishing-resistant authentication

FIDO2/WebAuthn and passkeys use origin and relying-party information when creating and verifying the authentication response. A credential registered for one legitimate origin should not normally produce a valid response for an attacker-controlled origin.

This makes phishing-resistant authentication a strong preventive control against many AitM login flows. It does not solve every problem:

- a compromised endpoint can still interact with an active session;
- malware can abuse the browser after the user authenticates;
- users can be tricked into registering an unwanted credential;
- downstream applications may create weaker independent sessions;
- account recovery can provide an alternate path.

Use phishing-resistant authentication for privileged and high-impact accounts, then protect the resulting sessions and endpoints as well.

## Safe local demonstration of session-context mismatch

The following Python model simulates a legitimate session followed by a request from a different device, region and browser family. It uses an opaque identifier and invented metadata only. It does not create a cookie, contact an IdP or replay a token.

Save it as **aitm-session-context-model.py** and run:

~~~bash
python aitm-session-context-model.py
~~~

~~~python
from dataclasses import dataclass


COMMAND_NAME = "aitm-session-context-model"


@dataclass
class Session:
    opaque_id: str
    user: str
    device: str
    region: str
    browser_family: str
    revoked: bool = False


@dataclass(frozen=True)
class Request:
    device: str
    region: str
    browser_family: str
    sensitive_action: bool = False


def context_changes(session: Session, request: Request) -> int:
    return sum(
        (
            session.device != request.device,
            session.region != request.region,
            session.browser_family != request.browser_family,
        )
    )


def evaluate(session: Session, request: Request) -> str:
    if session.revoked:
        return "DENY: session revoked"

    changes = context_changes(session, request)

    if request.sensitive_action and changes >= 2:
        session.revoked = True
        return "DENY: suspicious context; session revoked"

    if changes >= 2:
        return "CHALLENGE: require reauthentication"

    return "ALLOW: context is consistent"


session = Session(
    opaque_id="opaque-session-1",
    user="alice",
    device="managed-laptop",
    region="FR",
    browser_family="browser-family-a",
)

normal_request = Request(
    device="managed-laptop",
    region="FR",
    browser_family="browser-family-a",
)

replay_request = Request(
    device="unknown-host",
    region="US",
    browser_family="browser-family-b",
    sensitive_action=True,
)

print(COMMAND_NAME)
print("normal request:", evaluate(session, normal_request))
print("replay request:", evaluate(session, replay_request))
print("after revoke  :", evaluate(session, normal_request))
~~~

Expected output:

~~~text
aitm-session-context-model
normal request: ALLOW: context is consistent
replay request: DENY: suspicious context; session revoked
after revoke  : DENY: session revoked
~~~

This is a detection and response model, not a recommendation to bind every session to one IP address. Real deployments must account for travel, mobile networks, corporate proxies, VPNs and browser updates.

## Validation plan for an authorised lab

Use a disposable account, a local or isolated test application and synthetic data. Do not configure a public phishing domain or place a real identity provider behind a relay.

### Authentication policy

- verify that privileged users are required to use phishing-resistant authentication;
- verify that legacy authentication paths are disabled;
- verify that recovery and factor-registration paths require strong authentication;
- verify that sensitive actions trigger reauthentication or step-up controls;
- document which applications create their own independent sessions.

### Session lifecycle

- confirm that sessions rotate after authentication and privilege changes;
- verify idle and absolute expiration;
- verify server-side logout and revocation;
- verify that password and factor changes invalidate the expected sessions;
- verify that access is denied after account disablement;
- verify that a resource does not accept an expired or revoked session.

### Detection and response

- create synthetic sign-in events with a changed device and region;
- confirm that the identity platform records interactive and non-interactive use;
- verify that alerts distinguish a context change from a normal network transition;
- confirm that a suspicious session can be revoked without waiting for expiry;
- measure false positives before enabling automatic blocking;
- document the response time from alert to revocation.

The objective is to prove the control plane—logging, risk evaluation, reauthentication and revocation—not to capture a working session.

## Detection strategy

No single signal identifies every AitM campaign. Correlate several weak signals.

### Identity-provider signals

- a new sign-in context shortly after a normal authentication;
- a successful MFA event followed by activity from an unrelated device;
- interactive and non-interactive events that disagree on platform or location;
- a session or token used from an ASN, region or user agent not associated with the original event;
- access to sensitive resources without a corresponding fresh authentication;
- multiple accounts showing similar new-domain or infrastructure patterns.

Avoid treating “impossible travel” as proof. Proxies, mobile networks and cloud egress can produce false positives. It is most useful when combined with endpoint and session evidence.

### Endpoint signals

- a browser launched from an unusual parent process;
- an unsigned or untrusted extension added shortly before the event;
- suspicious access to browser profile directories;
- malware or scripting activity targeting browser processes;
- a new local proxy or unexpected proxy configuration;
- a browser session continuing after the endpoint has generated an EDR alert.

MITRE's [T1539 detection guidance](https://attack.mitre.org/techniques/T1539/) includes monitoring suspicious access to browser cookie stores and memory, as well as token reuse from unusual user agents or locations. These controls require careful privacy and false-positive management.

### Network and email signals

- links using look-alike registrable domains;
- newly registered or newly observed authentication domains;
- DNS changes inconsistent with a known service;
- mail messages whose visible brand and actual link origin disagree;
- a login journey that begins from a domain not approved for the service;
- TLS metadata inconsistent with the expected domain, used only as an enrichment signal.

The browser's lock icon is not a sufficient control. Enforce approved origins through safe-link rewriting, DNS and web filtering, browser policies and user-facing reporting.

## Containment and incident response

When an AitM session is suspected:

1. revoke active sessions and refresh tokens for the affected account;
2. reset the password from a trusted endpoint;
3. require fresh phishing-resistant authentication;
4. isolate and preserve the suspected endpoint;
5. review registered devices, authentication methods and OAuth consents;
6. inspect mailbox rules, files, API calls and administrative actions;
7. identify downstream applications with independent sessions;
8. search for the same domain, infrastructure and campaign pattern across users;
9. preserve identity, endpoint, email and proxy logs.

Password reset alone may not remove a session that has already been issued. Sign-out from one application may not terminate the IdP session or another application's local session. The revocation plan must cover all layers.

## Hardening priorities

### 1. Prefer origin-bound authentication

Require FIDO2, WebAuthn, passkeys or another phishing-resistant method for privileged users and high-impact applications. Do not downgrade to SMS or software codes because a migration is inconvenient.

### 2. Reduce bearer-token value

Where supported:

- use sender-constrained or device-bound tokens;
- rotate refresh tokens and detect reuse;
- scope tokens to one audience and minimum permissions;
- avoid long-lived tokens in browser local storage;
- use a backend-for-frontend pattern where it reduces browser exposure;
- revoke tokens after password, factor or risk events.

Token protection and continuous evaluation are not universal. Verify support by operating system, client and resource.

### 3. Strengthen session management

For web applications:

- set Secure, HttpOnly and an appropriate SameSite attribute on session cookies;
- prefer host-only cookies and the __Host- prefix where compatible;
- rotate the session identifier after login and privilege changes;
- use idle and absolute timeouts;
- avoid session identifiers in URLs, logs and referrerable content;
- require CSRF protection for state-changing requests;
- enforce reauthentication before high-impact actions;
- provide a reliable server-side revocation path.

Cookie flags protect specific delivery and browser behaviours. They do not make a stolen active session harmless.

### 4. Harden enrollment and recovery

An attacker often targets the weakest alternate path:

- new-device registration;
- factor replacement;
- password recovery;
- application consent;
- help-desk verification;
- legacy protocols.

Protect these flows with strong authentication, approval controls, rate limits, audit events and separate monitoring. A strong primary login is not enough if recovery can silently create a new trusted session.

### 5. Protect the endpoint and browser

- restrict browser extensions;
- keep browsers and operating systems current;
- block untrusted downloads and unsigned tooling;
- deploy EDR with browser and credential-access telemetry;
- remove local administrator rights;
- isolate privileged administration from everyday browsing;
- use managed browser policies and approved authentication origins.

### 6. Make the response fast

Define who can revoke sessions, disable accounts, remove OAuth consents and isolate endpoints. Measure the time from a high-confidence alert to containment. A perfect detection rule that nobody can act on is not a complete control.

## What this teaches

Evilginx2-style attacks expose the gap between authenticating a user and protecting the session that follows. MFA can be correctly enforced while the resulting session is still stolen through a compromised browser, endpoint or relay.

The practical questions are:

- Is the authentication method bound to the legitimate origin?
- Which artifact represents the session after MFA?
- Can that artifact be replayed from another device?
- Does the application detect context changes?
- Can every related session and token be revoked quickly?
- Are recovery and downstream applications protected to the same standard?

The right response is not to abandon MFA. It is to combine phishing-resistant authentication, hardened session management, endpoint protection, behavioural detection and rapid revocation.

## Sources

1. [MITRE ATT&CK — Adversary-in-the-Middle (T1557)](https://attack.mitre.org/techniques/T1557/)
2. [MITRE ATT&CK — Steal Web Session Cookie (T1539)](https://attack.mitre.org/techniques/T1539/)
3. [Okta Security — Keeping Phishing Adversaries Out of the Middle](https://sec.okta.com/articles/phishingasaservice/)
4. [OWASP — Cookie Theft Mitigation](https://cheatsheetseries.owasp.org/cheatsheets/Cookie_Theft_Mitigation_Cheat_Sheet.html)
5. [OWASP — Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
6. [CISA — Identity and Access Management Recommended Best Practices](https://www.cisa.gov/sites/default/files/2023-12/ESF%20IDENTITY%20AND%20ACCESS%20MANAGEMENT%20RECOMMENDED%20BEST%20PRACTICES%20FOR%20ADMINISTRATORS%20PP-23-0248_508C.pdf)
