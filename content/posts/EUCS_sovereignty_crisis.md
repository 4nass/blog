---
author: ["Anass"]
title: "EUCS and European Cloud Sovereignty"
date: "2024-09-09"
description: "Why the EUCS debate is about more than cloud security: separating cyber assurance, legal exposure, operational control and European strategic autonomy."
tags: ["EUCS", "Cloud Sovereignty", "European Cybersecurity", "Technology Policy", "France"]
categories: ["Technology Policy"]
topics: ["Technology & Policy"]
content_type: "industry-brief"
ShowToc: true
---

## The short version

The **European Union Cybersecurity Certification Scheme for Cloud Services**, or **EUCS**, was designed to create a common European approach to the cybersecurity certification of cloud services.

It became controversial because the debate quickly moved beyond technical assurance. Member States and industry also asked whether the scheme should address ownership, foreign legal exposure, administrative control, supply-chain dependency and the ability of Europe to keep operating if a provider or technology becomes unavailable.

Those are legitimate concerns, but they are not the same question. A provider can operate a technically secure service without being independent of a non-European jurisdiction. Conversely, a European-owned provider can still deliver a poorly secured or operationally fragile service. Treating both dimensions as one label makes procurement decisions harder to explain and certification harder to trust.

> **Status note — 29 August 2026:** EUCS remains a candidate scheme under development in ENISA's European cybersecurity certification framework. It should not be presented as an already-adopted European sovereignty label. The European Commission is also pursuing a separate cloud and AI sovereignty framework, which reinforces the need to distinguish the two subjects.

## What EUCS is — and what it is not

The EUCS is being prepared under the **Cybersecurity Act**, Regulation (EU) 2019/881. Its intended purpose is to harmonise cybersecurity expectations for cloud services, improve trust and reduce fragmentation between national approaches.

ENISA describes EUCS as a candidate cloud-services scheme. On its certification framework page, EUCS is still listed among the schemes under development, while EUCC—the European Common Criteria scheme for ICT products—is already in operation.

This distinction matters:

- **EUCS is not GDPR.** It does not replace data-protection obligations or a data-processing agreement.
- **EUCS is not a location guarantee.** A workload being hosted in an EU region does not, by itself, answer who controls the provider, the administration plane or the keys.
- **EUCS is not a complete architecture assessment.** A certified cloud service can still be misconfigured by its customer.
- **EUCS is not automatically a sovereignty guarantee.** Cybersecurity certification and protection against foreign legal or strategic dependency require related, but distinct, evidence.

The [Cybersecurity Act](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A02019R0881-20250204) establishes the European certification framework and explains the role of European schemes in the internal market. The [ENISA certification framework](https://www.enisa.europa.eu/topics/product-security-and-certification/cybersecurity-certification-framework) identifies EUCS as a cloud scheme under development.

## Why did the debate become political?

Cloud infrastructure is not only a place where data is stored. It is a stack of dependencies:

1. compute, storage and network infrastructure;
2. the provider's control plane and management APIs;
3. software, hardware and managed services supplied by third parties;
4. administrators, support teams and incident-response capabilities;
5. encryption keys, backups, logs and identity systems;
6. the legal entities and jurisdictions able to influence the service.

A certification focused only on the first four layers may provide strong cybersecurity evidence while leaving strategic dependency questions unanswered. This is why the EUCS discussion attracted pressure from governments, national authorities, cloud providers and customers with critical workloads.

France's position has been particularly visible. The French approach, expressed through **SecNumCloud**, places importance on protection against certain extraterritorial legal pressures as well as on the location and control of sensitive operations. The [ANSSI guidance](https://cyber.gouv.fr/enjeux-technologiques/cloud/faq-qualification-secnumcloud/) makes clear that SecNumCloud addresses resilience against non-EU legal injunctions and covers more than the physical location of data.

The disagreement is therefore not simply “France versus foreign providers”. It is a question of scope: should a European cybersecurity scheme include sovereignty requirements, or should those requirements be addressed by a separate instrument?

## Cybersecurity and sovereignty are different dimensions

The cleanest way to analyse the debate is to separate the properties being evaluated:

| Dimension | Main question | Typical evidence |
| --- | --- | --- |
| Cybersecurity assurance | Can the service prevent, detect and recover from cyberattacks? | Security controls, testing, vulnerability management, incident response |
| Data sovereignty | Where are data and backups stored and processed? | Region restrictions, backup topology, transfer controls |
| Jurisdictional exposure | Which legal orders can affect the provider or service? | Ownership, applicable law, access procedures, legal and technical safeguards |
| Operational control | Who can administer, support and modify the platform? | Personnel, privileged access, key custody, support model, audit trails |
| Strategic autonomy | Can customers continue operating without a single external dependency? | Portability, open interfaces, exit plans, substitute providers, local capability |

These dimensions interact, but none is a proxy for the others. “Data in Europe” is not equivalent to “European control”. “European ownership” is not equivalent to “secure architecture”. A high assurance level is not automatically proof of technological autonomy.

That separation also improves procurement. A public authority can decide that a workload needs high cyber assurance, EU-only operations, protection from extraterritorial legal access, or a credible exit strategy—and document each decision rather than hiding several policy choices behind the word *sovereign*.

## The location question is necessary but insufficient

European data residency remains important. It can support regulatory compliance, reduce transfer complexity and limit some operational exposure. It is not, however, the complete sovereignty analysis.

For a sensitive workload, also ask:

- Who owns and controls the provider and its key subcontractors?
- Where are the administration and support teams located?
- Who can access the control plane, metadata, logs and backups?
- Who generates, stores and operates the encryption keys?
- Can the provider be compelled by a non-EU legal order?
- Does the customer have a practical way to export data and configuration?
- Could a provider suspension, licence decision or supply-chain event stop the service?
- Can the organisation operate the workload during a provider incident or geopolitical disruption?

The answer may be different for a public website, an internal collaboration platform, a healthcare record system and an identity provider. Sensitivity and criticality should drive the control set; nationality alone is too coarse a risk model.

## What the EUCS can contribute

A well-designed European cloud certification could provide real value even without trying to solve every sovereignty concern. It could:

- establish a common vocabulary for cloud security;
- reduce the need to interpret several national schemes;
- make control evidence comparable across providers;
- strengthen requirements for secure development, isolation and resilience;
- improve vulnerability handling, incident notification and auditability;
- give customers clearer information about assurance levels and residual risk;
- support cross-border procurement without lowering the security baseline.

The value disappears if a certificate is marketed as a universal answer to data protection, jurisdiction, business continuity and strategic autonomy. The certificate should state exactly what it covers and what remains the customer's responsibility.

## What a sovereignty layer would need to show

If the policy objective is to protect particularly sensitive European workloads, a sovereignty assessment should be explicit rather than implied. Useful evidence would include:

### Legal and ownership control

The assessment should identify the provider's ownership, controlling entities, applicable jurisdictions and material dependencies. It should explain how the provider handles a request from a non-EU authority and what technical, legal and organisational safeguards are available.

### Administrative independence

It should cover the people and systems that can administer the service: privileged identities, remote support, break-glass accounts, maintenance paths, logging and approval workflows. A European data centre does not provide much sovereignty if its control plane can be operated entirely from outside Europe.

### Key and trust-service control

Key management deserves separate attention. The location of encrypted data is not enough if another party controls the keys, certificate authority, identity provider or recovery process. Customers should understand who can create, use, revoke and recover keys.

### Supply-chain and continuity resilience

The assessment should include critical hardware, software, support contracts, external dependencies, service concentration and exit readiness. The relevant question is not whether every component is European; it is whether a dependency is understood, substitutable and manageable for the workload's criticality.

### Evidence and transparency

Claims should be backed by audit evidence, clear scope, expiry and change-management rules. A sovereignty claim that cannot be tested by a customer or auditor is a marketing statement, not a control.

## The French reference point: SecNumCloud

SecNumCloud is a useful reference because it shows why sovereignty is broader than region selection. ANSSI's guidance describes requirements covering the provider's legal situation, third-party services, operational autonomy, protection against extraterritorial law and the territoriality of data, administration and supervision.

That does not mean SecNumCloud should simply be copied into an EU-wide scheme. A national qualification can reflect a national risk appetite and procurement doctrine. A European scheme must also remain understandable across Member States, compatible with the internal market and proportionate for different classes of cloud service.

The productive question is therefore not “which label wins?” but “which property does each label prove, at what assurance level, and with what residual risk?”

## A practical assessment model for cloud workloads

Before selecting a provider, classify the workload across four axes:

| Axis | Example decision |
| --- | --- |
| Data sensitivity | Public, internal, confidential, regulated or highly sensitive |
| Business criticality | Degraded service acceptable, or service interruption intolerable |
| Threat and jurisdiction | Ordinary cyber risk, targeted threat, or concern about state access |
| Portability | Easy export, partial migration or material provider lock-in |

Then map the required controls:

1. Use EUCS or another recognised certification as evidence of cybersecurity assurance, not as a shortcut for the complete risk assessment.
2. Verify data, backup, log and administration locations contractually and technically.
3. Separate customer identity, provider identity and privileged break-glass access.
4. Keep control of keys or use a key-management model that matches the workload's sensitivity.
5. Test export of data, configuration, policies and audit history before an incident occurs.
6. Define an alternative operating mode for a provider outage or legal disruption.
7. Review the assessment after major ownership, architecture, subcontractor or jurisdiction changes.

## Why this matters for identity platforms

Identity services concentrate control. A cloud-hosted IAM platform may hold user profiles, authentication factors, signing keys, administrator accounts, federation metadata, session state and detailed access logs. The sovereignty decision therefore applies to more than the database containing user attributes.

For an identity platform, review at least:

- the location and legal control of the identity data and backups;
- the provider's privileged administration path;
- the custody of signing keys, encryption keys and recovery secrets;
- the availability of the directory, token endpoint and management plane;
- federation dependencies on external IdPs, DNS, certificate authorities and email/SMS providers;
- the ability to export realms, clients, policies, users and audit data;
- the break-glass procedure if the cloud control plane is unavailable;
- the evidence needed to prove who accessed or changed identity configuration.

This is where cybersecurity, sovereignty and resilience meet in practice. A technically secure IAM service can still create strategic dependency if the organisation cannot recover its identities, keys or policy configuration without the provider.

## Where the European debate is heading

The institutional direction increasingly points toward a separation of instruments. ENISA's 2024 activity report described the EUCS candidate scheme as still unresolved, while later ENISA planning documents continue to list EUCS among candidate schemes under development.

In June 2026, the European Commission presented the **Cloud and AI Development Act** as part of its technology sovereignty package. The Commission describes a proposed framework with four sovereignty levels for cloud and AI services, including criteria related to EU location, third-country independence, ownership and software supply-chain control. This is a policy proposal, not a substitute for an adopted EUCS scheme.

That evolution is useful conceptually. Cybersecurity certification can answer “how well is this service protected?” A sovereignty framework can answer “how much control and strategic independence does this service provide?” Customers can then combine both answers according to the workload rather than forcing one certificate to carry every policy objective.

## A better position than “European or not”

The debate becomes unproductive when sovereignty is reduced to the provider's passport or the address of a data centre. A more useful position has five parts:

1. **Require strong, comparable cybersecurity assurance.** The baseline should be measurable and independently assessed.
2. **Make jurisdictional exposure visible.** Customers should know who can legally influence the service and how the provider responds.
3. **Protect administrative and cryptographic control.** The most sensitive paths deserve more scrutiny than the storage region alone.
4. **Fund European capability.** Strategic autonomy requires skills, operators, software, hardware and viable alternatives—not only procurement rules.
5. **Design for exit.** Portability, interoperability and tested recovery reduce dependency whatever the provider's origin.

This approach does not ignore geopolitical risk. It turns that risk into explicit requirements that can be tested, audited and matched to the importance of the workload.

## Conclusion

The EUCS debate is not merely a dispute about whether Europe should favour local cloud providers. It is a design problem for digital trust.

Europe needs secure cloud services, a functioning internal market and the ability to protect critical workloads from legal, operational and strategic dependency. Those objectives are compatible, but they should not be confused.

The strongest model is layered: use cybersecurity certification for security assurance, assess sovereignty and jurisdiction separately, require operational resilience and preserve a credible exit path. A cloud service should be judged not only by where its data sits, but by who controls the service, who can influence it, how it can fail and whether the customer can continue without it.

## Sources

1. [ENISA — Cybersecurity Certification Framework](https://www.enisa.europa.eu/topics/product-security-and-certification/cybersecurity-certification-framework)
2. [ENISA — EUCS Cloud Services Scheme](https://www.enisa.europa.eu/publications/eucs-cloud-service-scheme)
3. [EUR-Lex — Cybersecurity Act, Regulation (EU) 2019/881](https://eur-lex.europa.eu/legal-content/EN/TXT/?uri=CELEX%3A02019R0881-20250204)
4. [ANSSI — FAQ avant de se lancer dans la qualification SecNumCloud](https://cyber.gouv.fr/enjeux-technologiques/cloud/faq-qualification-secnumcloud/)
5. [European Commission — Cloud and AI Development Act](https://digital-strategy.ec.europa.eu/en/policies/cloud-and-ai-development-act)
6. [ENISA — Consolidated Annual Activity Report 2024](https://www.enisa.europa.eu/sites/default/files/2025-06/Consolidated%20Annual%20Activity%20Report%202024.pdf)
