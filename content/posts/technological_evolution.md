---
author: ["Anass"]
title: "From Mainframes to AI: How Technology Moves the Trust Boundary"
date: "2024-07-08"
description: "A security-minded history of computing, from centralised mainframes to AI platforms, and the trust boundaries that each era creates."
tags: ["Mainframes", "Computing History", "Data Security", "Cloud Computing", "Artificial Intelligence", "Identity"]
categories: ["Technology Policy"]
topics: ["Technology & Policy"]
content_type: "industry-brief"
ShowToc: true
---

Technology history is often told as a sequence of machines: mainframes, personal computers, the internet, the cloud and now artificial intelligence. A more useful story for security is the movement of the **trust boundary**.

Where is the data? Who can access it? Which component makes the decision? What happens when that component fails? Every new computing model changes the answers. It also creates a new set of assumptions that security teams eventually have to challenge.

<!--more-->

## The 1950s: computing becomes a central service

Mainframes turned computation into a shared organisational capability. Machines such as the UNIVAC I and IBM 7090 were expensive, specialised and physically concentrated. Users worked through operators, scheduled jobs and controlled terminals rather than directly administering the system.

![A NASA computer room with IBM 7090 mainframes and operators in 1962.](/images/technological-evolution/mainframe-room-public-domain.jpg)

<small><em>IBM 7090 computer room, 1962 · NASA / U.S. National Archives · public domain in the United States · <a href="https://commons.wikimedia.org/wiki/File:NASAComputerRoom7090.NARA.jpg">source</a></em></small>

The security model followed the architecture:

- physical access was a major part of the perimeter;
- privileged operations were concentrated in a small group;
- workloads were isolated through procedures, accounts and scheduling;
- availability mattered because one machine could support an entire organisation.

Centralisation simplified governance, but it also created a powerful single point of failure. Protecting the computer room was not enough: the organisation had to control operators, job submission, removable media, backups and recovery procedures.

## The 1960s: punched cards make data tangible

Punched cards provided a standard medium for preparing, transporting and replaying programs and data. They made batch processing practical, but they also made the security boundary physical. A card deck could contain source code, payroll records or instructions for a production job.

![An IBM punched card used as a machine-readable data record.](/images/technological-evolution/ibm-punch-card-cc0.png)

<small><em>IBM punched card · MdeVicente / Wikimedia Commons · <a href="https://creativecommons.org/publicdomain/zero/1.0/">CC0 1.0</a> · <a href="https://commons.wikimedia.org/wiki/File:Ibm_punch_card.png">source</a></em></small>

The lesson is still familiar: a data format is also a control surface. Access badges, sign-out registers and locked storage were part of the security architecture. So were validation, job authorisation and the ability to prove which deck had been executed.

IBM's System/360, introduced in 1964, extended this model by standardising a family of compatible systems. Portability improved, but standardisation also increased the value of shared interfaces and operating-system controls. A weakness in a common layer could affect many workloads at once.

## The 1970s: storage and time-sharing widen access

Hard disks changed the economics of storage. The IBM 3340, introduced in 1973, is remembered for its role in making disk storage more practical and reliable. At the same time, minicomputers and time-sharing systems allowed more people to interact with computing resources directly.

This was a major shift in trust:

1. more users received accounts instead of submitting work through an operator;
2. multiple sessions shared one running system;
3. files and permissions became everyday security controls;
4. administrators had to distinguish a user's identity from the machine's identity;
5. audit trails became necessary to understand actions after the fact.

![An IBM System/7 computer system from the 1970s.](/images/technological-evolution/time-sharing-1970s.jpg)

<small><em>IBM System/7, 1977 · Wikimedia Commons · public domain · <a href="https://commons.wikimedia.org/wiki/File:IBM_System-7_(I197704).jpg">source</a></em></small>

The foundations of modern access control emerged from this pressure. Isolation, least privilege, process separation and authenticated sessions were no longer specialist concerns. They became requirements for a multi-user system.

## The 1980s: the endpoint becomes part of the boundary

The personal computer moved computation closer to the user. The IBM PC, introduced in 1981, and compatible systems put local storage, software installation and processing power on desks that had previously been connected to central services.

That brought flexibility and a larger attack surface. Security teams now had to consider:

- unauthorised software and removable media;
- local administrator access;
- lost or stolen devices;
- malware moving between machines;
- weak or shared passwords;
- the difference between data protected in transit and data left on disk.

![An IBM Personal Computer from 1981.](/images/technological-evolution/personal-computer-1980s.jpg)

<small><em>IBM Personal Computer, 1981 · FA2010 / Wikimedia Commons · public domain worldwide · <a href="https://commons.wikimedia.org/wiki/File:IBM_PC_1981_makffm.jpg">source</a></em></small>

Encryption also became a more visible part of the conversation. DES was widely standardised for protecting electronic data, although the short key size that was acceptable at the time eventually became a limitation. The broader lesson was more durable than the algorithm: cryptography has a lifecycle, and a control that is strong enough today may become an operational liability tomorrow.

## The 1990s: the network turns local trust into a global problem

The internet and the World Wide Web connected systems that had previously been managed as separate islands. Services became reachable from outside the organisation, and the old perimeter was no longer a reliable boundary.

Secure protocols such as SSL and later TLS introduced authenticated and encrypted channels, but they did not solve every problem. The application still had to validate identities, handle sessions, protect keys, patch its dependencies and enforce authorisation.

![The first Web site displayed through the World Wide Web browser.](/images/technological-evolution/world-wide-web-1990s.png)

<small><em>The first Web site in the World Wide Web browser · CERN / Tim Berners-Lee · public domain worldwide · <a href="https://commons.wikimedia.org/wiki/File:W3_First_Website.png">source</a></em></small>

The security model changed from “keep outsiders away” to “assume communication can be hostile”. Firewalls, intrusion detection, public-key infrastructure, certificate management and vulnerability management became part of ordinary operations.

This period also established a pattern that continues today: a security mechanism is only as strong as its surrounding identity and operational processes. A valid certificate does not make an insecure application trustworthy, just as an encrypted connection does not make an over-privileged user legitimate.

## The 2000s: virtualisation and cloud abstract the machine

Cloud computing made infrastructure available as a service. AWS launched its first public cloud services in 2006, while virtualisation allowed many workloads to share physical resources without requiring customers to manage the underlying hardware.

The trust boundary moved again. Customers no longer controlled every server, network switch or storage device. Instead, they relied on a provider's control plane, APIs, support processes and isolation mechanisms.

![Sun rack-mount servers from 2008.](/images/technological-evolution/cloud-platform-2000s.jpg)

<small><em>Sun rack-mount servers, 2008 · Raysonho / Open Grid Scheduler / Grid Engine · public domain worldwide · <a href="https://commons.wikimedia.org/wiki/File:SunRackMountServers.jpg">source</a></em></small>

The shared-responsibility model emerged as a practical response. Providers operated parts of the infrastructure; customers remained responsible for their identities, configurations, data and applications. Most serious cloud incidents are not caused by a lack of encryption alone. They often involve excessive permissions, exposed management interfaces, unmonitored service accounts or an incorrect assumption about who owns a control.

Cloud security therefore depends on more than choosing a provider. It requires:

- strong identity and access management;
- separation of administrative and workload identities;
- short-lived credentials and auditable privilege elevation;
- secure defaults for storage, networks and APIs;
- tested backups and exit procedures;
- clear ownership for every control in the stack.

## The 2010s: data becomes a strategic asset

Smartphones, social platforms, sensors and connected devices produced data at a scale that changed both business and security priorities. Big Data systems made it possible to combine datasets that had previously remained separate, often revealing more about people than any single source did.

The Internet of Things widened the boundary again. Cameras, medical devices, industrial controllers and home appliances became networked computers with long lifecycles and uneven patching capabilities. Their security could not be separated from procurement, support contracts and physical safety.

![The rear of a server rack at the NERSC data center, showing network cabling and status LEDs.](/images/technological-evolution/data-platform-2010s.jpg)

<small><em>Rear of a server rack at NERSC, 2011 · Derrick Coetzee · <a href="https://creativecommons.org/publicdomain/zero/1.0/">CC0 1.0</a> · <a href="https://commons.wikimedia.org/wiki/File:Rear_of_rack_at_NERSC_data_center_-_closeup.jpg">source</a></em></small>

Regulation also became a system requirement. The GDPR made governance, purpose limitation, access rights, breach response and data protection by design operational concerns rather than purely legal questions. Data security was no longer only about preventing theft; it was also about limiting collection, controlling reuse and proving accountability.

## The 2020s: AI turns data and decisions into a platform

Artificial intelligence is not one technology or one risk. It is a platform pattern combining data pipelines, models, accelerators, APIs, applications, human review and increasingly autonomous agents.

![An abstract representation of a human brain under the control of artificial intelligence.](/images/technological-evolution/ai-agent-2020s.jpg)

<small><em>Artificial Intelligence · Pixabay / Wikimedia Commons · <a href="https://creativecommons.org/publicdomain/zero/1.0/">CC0 1.0</a> · <a href="https://commons.wikimedia.org/wiki/File:Artificial-Intelligence.jpg">source</a></em></small>

AI changes the trust questions in several ways:

- **Data provenance:** can the organisation explain where training and retrieval data came from?
- **Model supply chain:** which model, weights, libraries and plugins are running in production?
- **Prompt and input handling:** can untrusted content influence instructions or tool calls?
- **Output integrity:** who validates a recommendation before it changes a real-world decision?
- **Agent identity:** which user, service or workload is authorised to call a tool?
- **Confidentiality:** can sensitive prompts, context or outputs reach a provider or another tenant?
- **Availability and cost:** what happens when the model, API, accelerator or quota is unavailable?

The key difference from a traditional application is not that AI is mysterious. It is that the decision path is probabilistic, data-dependent and often distributed across services. Security has to cover the model and the surrounding system, not only the endpoint that exposes the chat interface.

## The trust boundary keeps moving

The eras differ, but the questions recur:

| Era | Dominant boundary | Main security pressure |
| --- | --- | --- |
| Mainframe | Computer room and operator process | Concentrated privilege and availability |
| Batch and punched cards | Physical media and job submission | Unauthorised data or code entering the run |
| Time-sharing | Account, process and file permissions | Multi-user isolation and auditability |
| Personal computing | Endpoint and local storage | Malware, loss and unmanaged software |
| Internet services | Network, application and certificate | Remote exposure and identity at scale |
| Cloud platforms | Provider control plane and customer configuration | Shared responsibility and privilege sprawl |
| AI platforms | Data, model, tools and human decision | Provenance, indirect input and autonomous action |

The boundary does not disappear when infrastructure becomes abstract. It becomes harder to see. That is why architecture diagrams should show identities, keys, control planes, data flows and failure modes—not only servers and network zones.

## Identity is the thread through the whole history

Every era needed an answer to the same question: **who is allowed to do what, under which conditions, and how can we prove it later?**

The implementation changed from operator procedures to account databases, certificates, federated identity, cloud IAM and workload identities. The principle did not.

For modern platforms, identity is often the most durable control plane available. A resilient design should make clear:

1. which human or workload initiated an action;
2. which policy evaluated it;
3. which credentials and keys were used;
4. which downstream service trusted the result;
5. which evidence remains after the action;
6. how access is revoked when the context changes.

This is especially important when AI agents can call APIs or perform actions on behalf of a user. “The model decided” is not an identity, an authorisation decision or an audit trail. The system still needs explicit principals, scoped permissions, approval boundaries and recoverable logs.

## What the next decade should make explicit

The next generation of platforms will likely combine cloud infrastructure, edge devices, AI models and autonomous workflows. Security architecture should prepare for that convergence rather than treating each component as a separate product.

The most useful design questions are practical:

- Can the organisation operate if a provider, model or identity service is unavailable?
- Can it revoke a compromised identity without taking down the whole platform?
- Can it trace a decision across data, model, tool and human approval?
- Can it replace a dependency without losing its users, policies or audit history?
- Can it prove which version of a model or policy produced an outcome?
- Can it detect when a trusted input has become an untrusted instruction?

These are not futuristic concerns. They are the current versions of access control, change management, incident response and business continuity.

## Conclusion

From mainframes to AI, computing has repeatedly traded physical control for reach, flexibility and scale. Each step has moved the trust boundary and created a new concentration of power: the operator, the operating system, the endpoint, the network, the cloud control plane and now the data-and-model pipeline.

The strongest security posture comes from making that boundary visible. Name the principals, isolate the privileged paths, protect the keys, validate the inputs, log the decisions and rehearse the failure mode. The machines will continue to change; those disciplines will remain useful.

## Sources and image credits

1. [NASA computer room with IBM 7090 systems — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:NASAComputerRoom7090.NARA.jpg), NASA / U.S. National Archives, public domain in the United States.
2. [IBM punched card — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Ibm_punch_card.png), MdeVicente, CC0 1.0.
3. [IBM System/7 — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:IBM_System-7_(I197704).jpg), public domain.
4. [IBM PC 1981 — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:IBM_PC_1981_makffm.jpg), FA2010, public domain worldwide.
5. [W3 First Website — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:W3_First_Website.png), CERN / Tim Berners-Lee, public domain worldwide.
6. [Sun rack-mount servers, 2008 — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:SunRackMountServers.jpg), Raysonho / Open Grid Scheduler / Grid Engine, public domain worldwide.
7. [Rear of rack at NERSC data center — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Rear_of_rack_at_NERSC_data_center_-_closeup.jpg), Derrick Coetzee, CC0 1.0.
8. [Artificial Intelligence — Wikimedia Commons](https://commons.wikimedia.org/wiki/File:Artificial-Intelligence.jpg), Pixabay, CC0 1.0.
9. [The NIST Definition of Cloud Computing](https://csrc.nist.gov/pubs/sp/800/145/final), NIST SP 800-145.
10. [General Data Protection Regulation — EUR-Lex](https://eur-lex.europa.eu/eli/reg/2016/679/oj), Regulation (EU) 2016/679.
