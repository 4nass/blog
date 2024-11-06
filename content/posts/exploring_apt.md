---
author: [ "Anass" ]
title: "Understanding Advanced Packaging Tool (APT), Sources and Keyrings"
date: "2024-11-06"
description: "The topic of repositories is always a large one, and comes up frequently. It is an item which people often get wrong and confused with. Please take the time to read the information below and any references which is linked to before acting on anything."
tags: [ "APT", "package management", "Linux repositories", "key management", "PGP keys" ]
categories: [ "Security",  "Linux" ]
ShowToc: true
---

The Advanced Packaging Tool (APT) is a powerful command-line utility in Debian-based Linux distributions, like Debian
and Ubuntu, for managing software packages. APT simplifies package management by handling dependencies, downloading,
installing, updating, and removing packages efficiently. Through APT, users can install software directly from trusted
repositories on the internet, keeping their systems secure and up-to-date.

### Why Use APT?

APT is essential for maintaining a stable and secure Linux environment. It allows for seamless updates to system
software and dependencies, ensuring compatibility across packages and reducing manual configuration. Its package
verification mechanisms protect users from corrupted or malicious software, which is critical for system integrity.

### How to Use APT

APT commands are straightforward and highly versatile:

1. Update Package Lists:

Refresh the local package index with:

```bash
sudo apt update
```

This command checks for the latest versions available in configured repositories.

2. Install Packages:

Install a package by specifying its name:

```bash
sudo apt install package-name
```

APT resolves dependencies automatically, downloading any required additional packages.

3. Upgrade Packages:

Upgrade installed packages to their latest versions:

```bash
sudo apt upgrade
```

4. Remove Packages:

Uninstall packages when no longer needed:

```bash
sudo apt remove package-name
```

5. Search for Packages:

Find packages related to keywords:

```bash
apt search keyword
```

APT is an indispensable tool for Linux users, enabling efficient and secure package management with minimal hassle.
Whether you are adding new software, updating existing packages, or securing your repositories, APT provides a robust
foundation for a smooth Linux experience, but with great power comes responsibility—especially when it comes to
repository keys.

The traditional `apt-key add` method for managing repository keys has been deprecated due to significant
security risks. This article explores the reasons for this deprecation, introduces safer key management practices, and
provides step-by-step guidance to ensure your APT configuration remains secure.

---

### Why `apt-key add` Is Deprecated

The APT package manager in Debian and Ubuntu allows users to add third-party repositories to install additional
software. Previously, `apt-key add` was a commonly used command to add repository signing keys, but it is now
deprecated.
The primary reason for this deprecation is security. Using `apt-key add` adds keys to a system-wide, highly trusted
keyring (`/etc/apt/trusted.gpg` or `/etc/apt/trusted.gpg.d/`).

Any key added to this keyring is fully trusted across all APT repositories, not just the repository it was intended for.
This configuration weakens the verification process and opens the system to potential risks. For instance, if a key from
a less trusted repository is compromised, it could allow an attacker to replace or inject malicious packages into any
repository, even official ones, if they are signed with that key.

### The Importance of Repository-Specific Trust

APT’s design was intended to verify package signatures, ensuring only authorized code could be installed. However,
blindly trusting keys from any repository undermines this security. Instead, APT should accept signatures from a
third-party repository only for packages explicitly downloaded from that repository—preventing “cross-signing.” This
approach reinforces APT’s default priority rules, where official repositories maintain higher priority than third-party
ones.

### Safe Key Management with Repository-Specific Keys

To manage repository keys safely, use dedicated keyrings and configure APT to only trust each key for its specific
repository. Below are the steps to implement this security practice.

---

### Step-by-Step Guide to Secure APT Key Management

1. Download the Repository Key

First, download the desired repository key. For example:

```bash
wget https://archive.kali.org/archive-key.asc
```

The key should be in a "PGP public key block" format. Verify this format with:

```bash
file archive-key.asc
```

2. Convert the Key Format if Necessary

If the key format is not compatible, convert it using GPG:

```bash
gpg --no-default-keyring --keyring ./temp-keyring.gpg --import archive-key.asc
gpg --no-default-keyring --keyring ./temp-keyring.gpg --export --output your-keyfile-name.gpg
rm temp-keyring.gpg
```

3. Place the Key in a Secure Location

Move the key to a dedicated directory, such as `/etc/apt/keyrings/`, which may need to be created first:

```bash
sudo mkdir -p /etc/apt/keyrings/
sudo mv your-keyfile-name.gpg /etc/apt/keyrings/
```

4. Modify the Repository’s `.list` File

Edit the repository’s .list file in `/etc/apt/sources.list.d/` to specify the signed-by option. This restricts the key
to only sign packages from its intended repository:

```plaintext
deb [signed-by=/etc/apt/keyrings/your-keyfile-name.gpg]  http://http.kali.org/kali kali-rolling main contrib non-free non-free-firmware
```

Note that in the command above, we don’t modify the file `/etc/apt/sources.list`, but instead we create a new file
`/etc/apt/sources.list.d/kali-experimental.list`. This is a convention: the file `/etc/apt/sources.list` should only
contain the main branch, while additional branches should go in `/etc/apt/sources.list.d/`, one branch per file.

**Sources.list Format**

```plaintext
deb   http://http.kali.org/kali   kali-rolling   main contrib non-free non-free-firmware
<Archive>   <Mirror>                <Branch>                <Components>
```

* **Archive** is going to be `deb` (Regular Binary) or `deb-src` (Source), depending on if you want a package or the
  source of the package.
* **Mirror** should be `http://http.kali.org/kali` as this is our load balancer, which will direct you to best mirror.
* **Branch** is what version of Kali you wish to use.
* **Components** are what packages you wish to use, based on the Debian Free Software Guidelines (DFSG). Kali defaults
  to everything.

5. Manage Existing Keys

For keys previously stored in `/etc/apt/trusted.gpg.d/`, move them to `/etc/apt/keyrings/`, then update the respective
.list files with signed-by fields. If there are keys in `/etc/apt/trusted.gpg`, manually locate and remove any
unnecessary third-party keys.

6. Use APT Pinning to Control Repository Priority

APT allows pinning, which prioritizes packages from specific sources. To verify or adjust pinning, use:

```bash
apt-cache policy
```

You can customize pinning preferences in `/etc/apt/preferences.d/`.

7. (Optional) Switch to Deb822 Format

For better readability and manageability, consider using `.sources` files instead of `.list` files in the Deb822 format,
which allows more explicit repository configurations.

---

### Security Considerations and Future Directions

Although securing your APT sources and keys reduces attack vectors, it is essential to remember that APT itself remains
somewhat permissive. For example, packages with root privileges may still execute arbitrary shell commands. However,
improvements in package security practices are continually developing.

---

By following these guidelines, you can better secure your Linux system against potential package-based attacks and
reinforce trust boundaries between official and third-party repositories.

---

**Sources:**

1. [Debian wiki](https://wiki.debian.org/DebianRepository/UseThirdParty)
2. [Kali Network Repositories](https://www.kali.org/docs/general-use/kali-linux-sources-list-repositories/)
