---
author: [ "Anass" ]
title: "Bypassing Intune Compliant Device Conditional Access: A Security Perspective"
date: "2025-01-02"
description: "How attackers can exploit loopholes in Microsoft's Conditional Access policies and how to mitigate the risks."
tags: [ "Microsoft Intune", "Azure Entra ID",  "IAM", "Conditional Access", "SecurityThreats", "Cybersecurity" ]
categories: [ "Security",  "Technology" ]
ShowToc: true
---

Microsoft Intune's Conditional Access policies are designed to enforce compliance and protect enterprise environments by ensuring only managed and compliant devices can access corporate resources. However, security researchers at Jumpsec Labs have demonstrated a technique to bypass these restrictions, raising concerns about the effectiveness of Intune’s enforcement mechanisms.

This article explores how the TokenSmith method enables adversaries to sidestep device compliance checks and what security teams can do to mitigate such risks.

### Understanding Conditional Access and Intune Compliance

Azure AD Conditional Access policies enforce security requirements based on:

- User identity (who is signing in)

- Device compliance (whether the device meets security requirements)

- Application & location (where and how the sign-in attempt occurs)

When properly configured, these policies ensure that only trusted users with compliant devices can access corporate data. However, the recently disclosed bypass technique highlights potential gaps.

---

### The "TokenSmith" Attack: Bypassing Intune Device Compliance

## How the Attack Works

Jumpsec Labs identified a way to obtain an OAuth refresh token from a device that was initially compliant, then reuse that token on a non-compliant system without triggering a re-evaluation of device compliance.

1. **Initial Authentication on a Compliant Device**
The attacker first logs into a managed, compliant device where the user is authorized to access corporate resources.

2. **Stealing or Extracting the Refresh Token**
Using various methods (e.g., malware, phishing, token extraction tools), the attacker retrieves the user's OAuth refresh token.

3. **Token Reuse on a Non-Compliant Device**
Instead of re-authenticating, the attacker uses the stolen refresh token on an unmanaged device, bypassing Conditional Access checks.

## Why Does This Work?

- **Lack of Continuous Device Validation:** 
Conditional Access policies validate compliance only at login and do not continuously revalidate device status.

- **Token Lifetime and Reuse:** 
OAuth refresh tokens are long-lived and allow re-authentication without requiring a fresh login or device compliance check.

---

### Mitigating the Risk

1. **Enforce Conditional Access with Device ID Matching:**
To prevent token reuse on non-compliant devices, organizations should enforce Device ID binding:

- Ensure device-bound tokens are used (e.g., by leveraging Azure AD’s device authentication claims).

- Require a valid device certificate during authentication.

2. **Reduce Token Lifetime and Enforce Reauthentication:**
By shortening the refresh token lifetime, security teams can force frequent reauthentication:

- Use Azure AD Token Lifetime Policies to set a short expiration for refresh tokens.

- Require interactive reauthentication at shorter intervals.

3. **Enable Continuous Access Evaluation (CAE):**

Microsoft's Continuous Access Evaluation (CAE) feature helps detect changes in device compliance dynamically:

- CAE can trigger re-evaluations when security conditions change.

- Ensure CAE is enabled for critical applications like Microsoft 365 and Azure services.

4. **Implement Endpoint Detection & Response (EDR):**

Use Microsoft Defender for Endpoint or similar EDR solutions to:

- Detect token exfiltration attempts.

- Monitor suspicious authentication patterns across devices.

5. **Strengthen Phishing & Credential Theft Protections:**

Many attacks start with phishing or malware-based credential theft. To mitigate these risks:

- Enforce phishing-resistant MFA (e.g., FIDO2 keys or certificate-based authentication).

- Monitor for unusual token usage using Azure AD Sign-in logs and Identity Protection.

---

### Conclusion

The "TokenSmith" bypass method highlights a critical weakness in how Intune's device compliance enforcement works. While Conditional Access policies provide strong security controls, organizations must adopt continuous monitoring, device-bound authentication, and token management strategies to close security gaps.

By implementing device identity binding, Continuous Access Evaluation, and proactive token lifecycle management, organizations can significantly reduce the risk of unauthorized access due to stolen or misused OAuth tokens.

---

**Sources:**

1. [Jumpsec Labs: TokenSmith Research](https://labs.jumpsec.com/tokensmith-bypassing-intune-compliant-device-conditional-access/)
2. [Microsoft Conditional Access Documentation](https://learn.microsoft.com/en-us/entra/identity/conditional-access/)
3. [OAuth Token Security Best Practices](https://datatracker.ietf.org/doc/html/rfc9700)
