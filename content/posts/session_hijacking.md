---
author: ["Anass"]
title: "Session Hijacking 2.0: Emerging Threats and Defenses"
date: "2024-10-07"
description: "The new Frontier in Cyber Attacks."
tags: ["Session Hijacking", "Infostealer", "MitM", "AitM", "BitM", "MFA bypass"]
categories: ["Security", "Technology"]
ShowToc: true
---

### Introduction

In today's rapidly evolving cybersecurity landscape, attackers have found new ways to bypass multi-factor authentication (MFA) and compromise user sessions through tactics such as session hijacking and infostealer malware. These techniques pose significant risks to organizations and users, even those who have deployed MFA solutions to secure their systems.

<!--more-->

### Session Hijacking 2.0: The Evolution of a Classic Attack

Session hijacking, traditionally known as a man-in-the-middle (MitM) attack, has undergone a modern evolution. Now labeled **"Session Hijacking 2.0,"** this attack method has become even more sophisticated, leveraging phishing and malicious software to steal session cookies and bypass MFA controls. The attacker intercepts and steals a user's session ID or cookie, which grants them access to the victim's online services without requiring any re-authentication.

This bypasses MFA since the attacker doesn't need the victim's password or second factor to access a secured account. The key point is that once a session is authenticated, MFA is not applied again. Attackers exploit this gap to resume sessions on their own devices using stolen cookies, making it appear as if the session is legitimate from the service's perspective.

### Infostealers: Opportunistic Session Hijacking

Infostealer malware is another potent threat used for session hijacking. Unlike targeted attacks, infostealers are more opportunistic, often distributed via phishing campaigns, malicious downloads, and infected websites. They indiscriminately steal all session cookies and login credentials stored in browsers. This is particularly dangerous as it affects a wide range of sessions across various apps, including critical business systems.

Once an attacker has these stolen session cookies, they can import them into their browser and resume the victim’s session without needing to authenticate again. Even advanced authentication systems like passkeys or MFA can't protect against this, as no new authentication occurs during the hijacked session.

### Bypassing MFA through Phishing and Browser Exploits

The most common vector for bypassing MFA involves phishing attacks such as **adversary-in-the-middle (AitM)** and **browser-in-the-middle (BitM)** techniques. In these cases, the attacker replicates the authentication process through phishing websites that mimic legitimate login portals. When a user enters their credentials and MFA token, the attacker captures this information and uses it to hijack the session in real-time.

This method allows attackers to gain access to services protected by MFA without needing to crack or bypass MFA itself, as they are effectively stealing the authenticated session rather than attacking the MFA mechanism directly.

### Session hijacking in action

Watch the demo video below to see the full attack process unfold, starting from the initial infostealer compromise. It covers how session cookies are stolen, reimported into the attacker's browser, and how policy-based controls in Microsoft 365 (M365) are bypassed. The video also highlights how downstream apps, typically accessed via single sign-on (SSO), are compromised in attacks targeting both Microsoft Entra and Okta environments.

[![IMAGE ALT TEXT HERE](https://img.youtube.com/vi/RlSweA5UfYw/0.jpg)](https://www.youtube.com/watch?v=RlSweA5UfYw)


### Detection and Mitigation Strategies

Defending against session hijacking and MFA bypass requires a multi-layered approach, which includes:

- **Endpoint Detection and Response (EDR)**: While EDR systems can detect and block many commercial infostealers, more advanced attackers may use custom malware designed to evade detection. Organizations need to stay vigilant and ensure their EDR systems are up-to-date.
- **Application-Level Security**: Even after session cookies are stolen, robust app-level controls, such as IP-based restrictions and device fingerprints, can help identify and block unauthorized access. Strict access policies can limit the attacker’s ability to hijack sessions, especially if the stolen cookies are from less-protected apps downstream from the main Identity Provider (IdP).
- **Training and Awareness**: Educating users about phishing techniques and ensuring they recognize malicious links or files can help prevent the initial compromise that leads to session hijacking.

### Conclusion

While MFA has significantly improved security, it is not foolproof. Modern session hijacking techniques, combined with the use of infostealer malware, present a serious threat to organizations. Defenders must adopt a proactive stance, including educating users, deploying advanced EDR, and implementing strict session management policies, to mitigate the risks of session hijacking and MFA bypass attacks.

---

**Sources:**

1. [The Hacker News - Session Hijacking 2.0 — The Latest Way That Attackers are Bypassing MFA](https://thehackernews.com/2024/09/session-hijacking-20-latest-way-that.html)
2. [Push Security - What the rise of infostealers says about identity attacks](https://pushsecurity.com/blog/what-the-rise-of-infostealers-says-about-identity-attacks/)
2. [CERT-FR - COOKIE THEFT THREATS AND COUNTERMEASURES](https://www.cert.ssi.gouv.fr/uploads/CERTFR-2022-CTI-005.pdf)
