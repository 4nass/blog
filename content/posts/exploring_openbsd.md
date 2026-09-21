---
author: ["Anass"]
title: "Exploring OpenBSD: Secure by Default, Not by Magic"
date: "2024-07-11"
description: "A practical look at OpenBSD’s security model, from PF and secure defaults to privilege reduction, local verification and operational limits."
tags: ["OpenBSD", "Operating Systems", "PF", "System Hardening", "Unix"]
categories: ["Systems Infrastructure"]
topics: ["Security"]
content_type: "systems-note"
ShowToc: true
---

OpenBSD is often introduced with a slogan about security. The more interesting story is the set of engineering decisions behind it: a small and reviewable base system, conservative defaults, continuous auditing, privilege separation, memory protections and security primitives that applications can use explicitly.

That does not make an OpenBSD host invulnerable. A service can still be exposed by a bad firewall rule, an administrator can still grant excessive privileges, and an application can still contain a bug. OpenBSD changes the cost and the potential blast radius of those failures; it does not remove the need for architecture, patching and operational discipline.

This article explores that boundary through a realistic misconfiguration scenario and safe local checks. The commands are intended for an OpenBSD lab or a host you administer.

<!--more-->

## The short version

- OpenBSD is a free, multi-platform 4.4BSD-based operating system focused on correctness, security, portability and standardization.
- Its security model is layered: secure-by-default services, continuous code review, memory protections, privilege separation, `pledge(2)`, `unveil(2)`, `doas(1)` and PF.
- PF is a powerful packet filter, but the default presence of PF does not mean that a deployment has a correct ruleset.
- `pledge` limits the system operations a process may perform; `unveil` limits the filesystem view it can access.
- The strongest benefit appears when the application, operating system and network policy are designed together.
- “Secure by default” is a starting position. Updates, service inventory, least privilege and ruleset review remain mandatory.

> **Lab safety:** run the demonstrations only on an OpenBSD VM or another system you own. The PF example is parsed with `-n` and is not loaded. Do not use packet-generation, port-scanning or exploitation tools against networks that you do not administer.

## At a glance

| Field | Details |
| --- | --- |
| System | OpenBSD, a free 4.4BSD-based Unix-like operating system |
| Primary design goals | Correctness, security, portability and standardization |
| Network control | PF (`pf`, `pfctl`, `pf.conf`) |
| Process confinement | `pledge(2)` and `unveil(2)` |
| Privilege delegation | `doas(1)` and separated service accounts |
| Default posture | Non-essential services disabled |
| Update path | `syspatch(8)` for supported base-system security and reliability fixes |
| Main operational risk | A correct primitive used with an incorrect policy |
| Key lesson | Reduce privileges and make every exposed path intentional |

## What makes OpenBSD different?

The [OpenBSD project describes](https://www.openbsd.org/faq/faq1.html) its goals in terms of correctness, security, standardization and portability. Those goals influence both the code and the way the system is operated.

### Secure by default

OpenBSD does not enable every network daemon simply because a package is installed. The project’s [security documentation](https://www.openbsd.org/security.html) describes a default posture in which non-essential services are disabled. This reduces the number of listening sockets and the amount of configuration an administrator must understand before the first boot.

The principle is useful, but it has a limit: installing and enabling a service creates a new policy decision. A secure baseline is not a substitute for an explicit service inventory.

### Continuous code review

OpenBSD’s security reputation is closely associated with long-running source-code auditing. The benefit is not that every bug is found before release. The benefit is that code quality, attack surface and defensive design are treated as first-class engineering concerns rather than emergency additions after an incident.

### Privilege separation

Many OpenBSD services split network-facing parsing from privileged operations. A compromised or malformed-input path may then reach a constrained process instead of the full authority of the daemon. The quality of the boundary depends on the specific service, but the pattern is broadly useful: keep the code that handles hostile input away from the code that owns sensitive state.

### Memory and process mitigations

The project has developed and integrated mitigations such as W^X, address-space randomization, guard pages, randomized allocation and read-only relocation data. These mechanisms raise the cost of turning a memory-safety bug into reliable code execution. They are layers, not guarantees, and they do not fix an application’s authorization logic or exposed secrets.

## The security primitives in practice

### PF: the network boundary

[PF](https://www.openbsd.org/faq/pf/) filters TCP/IP traffic, performs NAT and can normalize traffic, control bandwidth and prioritize packets. Rules are evaluated in order, and a later matching rule can determine the final result unless `quick` stops evaluation.

The operational lesson is simple: read the complete ruleset, not the one rule you just added. A broad `pass`, an unintended interface, a missing `quick` or an overlooked IPv6 path can invalidate an otherwise careful design.

### `pledge(2)`: reduce system operations

The [`pledge(2)` system call](https://man.openbsd.org/pledge.2) lets a process declare the classes of system operations it expects to use. Operations outside the declared promises are refused or, in the normal violation mode, cause the process to be terminated.

An application that only parses a file and writes a result does not need the same operating-system authority as a general-purpose shell. `pledge` makes that difference enforceable after initialization.

### `unveil(2)`: reduce filesystem visibility

The [`unveil(2)` system call](https://man.openbsd.org/unveil.2) restricts which paths a process can see and which operations it may perform on those paths. A process may be allowed to read one configuration directory without being able to inspect the rest of the filesystem.

`unveil` is particularly valuable when combined with privilege separation. One process can receive network data, another can access a narrow set of files, and neither needs the authority of the whole service.

### `doas(1)`: make elevation explicit

[`doas(1)`](https://man.openbsd.org/doas) provides controlled execution as another user. Compared with allowing routine work from a root shell, a small `doas.conf` makes elevated commands and their intended users visible and reviewable.

### `syspatch(8)`: keep the base system current

The [OpenBSD security-update guidance](https://www.openbsd.org/faq/faq10.html) recommends `syspatch` for binary fixes on supported releases. It is a maintenance control, not a vulnerability scanner. A host still needs an inventory of packages, services, local changes and exposure.

## The attack scenario: a secure host with an unsafe exposure

Consider an OpenBSD host used as an internet-facing reverse proxy or small edge gateway.

### 1. A service is enabled for convenience

An administrator installs a web service, monitoring endpoint or management interface and binds it to all interfaces. The application is patched, but its intended audience is not documented.

### 2. PF contains a broad allow rule

To “make it work”, a rule permits inbound traffic to the service from `any`. A management endpoint that should have been reachable only from an administration network is now exposed to the internet.

### 3. The application receives hostile input

An attacker probes the public service and finds an application flaw, weak authentication or an exposed debugging feature. OpenBSD’s mitigations may make exploitation harder, but they do not turn a dangerous application configuration into a safe one.

### 4. The blast radius depends on the process boundary

If the service runs with broad filesystem and operating-system authority, an application compromise can become a host compromise. If the service is separated, uses a dedicated account, applies `pledge` and `unveil`, and has narrow PF egress rules, the same initial failure has fewer paths to sensitive data or administrative control.

The scenario is intentionally ordinary. It shows why “OpenBSD is secure” is not a sufficient threat model. The relevant questions are:

- Which interfaces listen?
- Which source networks may reach them?
- Which user owns the process?
- Which files can it read or write?
- Which system operations can it invoke?
- Where can it connect after receiving hostile input?
- How will an operator detect and revoke the exposure?

## Safe PF ruleset review

The following command is named `openbsd-pf-ruleset-review`. It validates a candidate ruleset without loading it, then displays the active rules and state when run on the host. Use a temporary file in a lab first:

```sh
#!/bin/ksh

COMMAND_NAME="openbsd-pf-ruleset-review"
RULESET="${1:-/tmp/pf-lab.conf}"

echo "== ${COMMAND_NAME} =="
echo "ruleset=${RULESET}"

if [[ ! -r "${RULESET}" ]]; then
    echo "error: ruleset is not readable"
    exit 2
fi

echo "-- syntax check only --"
doas pfctl -nf "${RULESET}"

echo "-- active ruleset --"
doas pfctl -sr

echo "-- state summary --"
doas pfctl -si
```

A small lab ruleset can make the intended policy explicit:

```pf
ext_if = "em0"
mgmt_net = "192.0.2.0/24"

set block-policy drop
set skip on lo

block all
pass out on $ext_if inet from ($ext_if) to any keep state
pass in on $ext_if inet proto tcp from $mgmt_net to ($ext_if) port ssh keep state
```

The example is deliberately narrow and uses documentation address space. Replace the interface and management network only in a lab or in a reviewed production change. The important test is not whether the file parses; it is whether the resulting policy matches the intended exposure, including IPv6, return traffic, logging and emergency access.

Useful read-only PF checks include:

```sh
doas pfctl -sr    # show filter rules
doas pfctl -ss    # show state table
doas pfctl -si    # show counters and statistics
doas pfctl -sa    # show all available information
```

## A local `pledge` and `unveil` demonstration

The following small C program runs locally and demonstrates the direction of the restriction. It allows the process to read `/etc/hostname`, locks the filesystem view, then attempts to read `/etc/passwd`. It does not open a network socket or modify the system.

Save it as `openbsd-sandbox-boundary.c` on an OpenBSD lab host:

```c
#include <err.h>
#include <fcntl.h>
#include <stdio.h>
#include <unistd.h>

static void
report_read(const char *path)
{
    int fd = open(path, O_RDONLY);

    if (fd == -1) {
        warn("open %s", path);
        return;
    }

    printf("allowed: %s\\n", path);
    close(fd);
}

int
main(void)
{
    const char *command_name = "openbsd-sandbox-boundary";

    printf("== %s ==\\n", command_name);

    if (pledge("stdio rpath unveil", NULL) == -1)
        err(1, "pledge initialization");

    if (unveil("/etc/hostname", "r") == -1)
        err(1, "unveil hostname");

    if (unveil(NULL, NULL) == -1)
        err(1, "lock unveil rules");

    if (pledge("stdio rpath", NULL) == -1)
        err(1, "lock pledge promises");

    report_read("/etc/hostname");
    report_read("/etc/passwd");
    return 0;
}
```

Compile and run it:

```sh
cc -Wall -Wextra -o openbsd-sandbox-boundary openbsd-sandbox-boundary.c
./openbsd-sandbox-boundary
```

The first read should succeed and the second should fail because the path was not unveiled. The exact error text can differ between versions and libc paths. The demonstration is intentionally modest: it shows a post-initialization boundary, not a complete sandbox for an arbitrary application.

## Verifying the service surface

Before hardening a host, establish what is actually exposed. On the host itself, use local inspection rather than scanning from an unrelated network:

```sh
COMMAND_NAME="openbsd-service-surface-check"
echo "== ${COMMAND_NAME} =="

echo "-- listening sockets --"
netstat -an -f inet | grep LISTEN || true

echo "-- enabled daemons --"
rcctl ls on

echo "-- PF status --"
doas pfctl -si
```

Review the output against the intended service inventory. A daemon enabled in `rc.conf.local`, a package-managed service and a process started by a custom script may be governed by different operational owners.

Do not interpret an empty `netstat` result as proof that the host is safe. Unix sockets, outbound connections, scheduled jobs, package hooks and management paths can still matter.

## Hardening workflow

### 1. Start from a minimal install

Install only the roles and packages required for the host. Keep non-essential daemons disabled and document every exception. The goal is not an artificially empty server; it is a host whose exposed surface has an owner and a reason.

### 2. Patch before exposure

Apply the current base-system patches with `syspatch` and update installed packages with the supported package workflow. Read the release errata and keep a record of the last successful update.

### 3. Design PF from the service contract

Write down the allowed flows before writing rules: source, destination, protocol, port, direction and expected state. Deny by default, permit only the documented paths, and review `pass` rules for both IPv4 and IPv6.

### 4. Separate administration from application traffic

Put management access on a dedicated network or VPN. Do not expose SSH, dashboards or control APIs to the public internet merely because PF can restrict them later.

### 5. Run services with the least authority

Use dedicated users, chroots or privilege separation where appropriate. Add `pledge` and `unveil` after understanding the application’s required files and system operations; an overbroad policy gives a false sense of confinement, while an over-tight policy can cause operators to disable the control during an incident.

### 6. Protect elevation paths

Keep `/etc/doas.conf` small and explicit. Prefer command-specific permissions to unrestricted root shells, require authentication for sensitive operations and review membership of administrative groups.

### 7. Observe the controls

Enable appropriate PF logging, monitor service state, retain authentication records and alert on unexpected daemon or ruleset changes. A control that cannot produce evidence is difficult to validate during an incident.

## Detection and incident response

When a service is suspected of compromise, collect evidence before changing everything at once:

- record the active PF ruleset and state table;
- identify the process, account and parent service that handled the connection;
- review authentication, daemon and PF logs around the event;
- compare enabled services and packages with the known-good inventory;
- inspect recent changes to `pf.conf`, `rc.conf.local`, `doas.conf` and service configuration;
- check whether the process had access to sensitive paths or unexpected outbound destinations;
- preserve the affected VM or disk image when forensic analysis is required.

Containment may involve blocking the service in PF, isolating the host, revoking credentials and rotating keys. Do not assume that `pledge` or `unveil` eliminates the need to investigate data access and credential exposure.

## What OpenBSD does not solve

OpenBSD’s design is strong, but it does not answer every security question:

- it does not decide whether a user should have access to an application;
- it does not validate business authorization rules;
- it does not remove vulnerable third-party packages;
- it does not automatically create a resilient backup or recovery plan;
- it does not prevent an administrator from writing an overbroad PF rule;
- it does not make a leaked credential harmless;
- it does not guarantee that a service is correctly confined without application-specific work.

This distinction is important when comparing operating systems. A hardened base can reduce the impact of a failure, but the final security posture still depends on the software, configuration, identity model and operational process around it.

## Why OpenBSD remains valuable

OpenBSD is valuable less because it promises a magic shield than because it makes defensive engineering visible. PF turns network policy into reviewable text. `doas` makes privileged actions explicit. `pledge` and `unveil` let developers narrow process authority. The base system’s defaults reduce unnecessary exposure, while the project’s source and release practices provide material to study and audit.

For security engineers, OpenBSD is also a useful reference implementation of a principle that applies far beyond BSD systems:

> **Make the secure path the easiest path, then make every remaining authority explicit.**

That principle scales from an edge firewall to an identity platform. It does not replace threat modeling, but it gives threat modeling concrete boundaries to inspect.

## Sources

1. [OpenBSD — About OpenBSD](https://www.openbsd.org/faq/faq1.html), project goals and system characteristics.
2. [OpenBSD — Security](https://www.openbsd.org/security.html), secure defaults and security technologies.
3. [OpenBSD — PF User’s Guide](https://www.openbsd.org/faq/pf/), packet filtering and NAT.
4. [OpenBSD — PF Getting Started](https://www.openbsd.org/faq/pf/config.html), ruleset validation and control commands.
5. [`pledge(2)` manual page](https://man.openbsd.org/pledge.2), system-operation restrictions.
6. [`unveil(2)` manual page](https://man.openbsd.org/unveil.2), filesystem visibility restrictions.
7. [`doas(1)` manual page](https://man.openbsd.org/doas), controlled privilege delegation.
8. [OpenBSD FAQ — Security Updates](https://www.openbsd.org/faq/faq10.html), `syspatch` and supported update paths.
