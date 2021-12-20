---
title: "SBOM-A-Rama (2021) Day 1 Write Up"
date: 2021-12-16T00:31:00-07:00
draft: false
series:
 - sbom
 - sbom-a-rama
---

# SBOM-a-Rama, Day 1


## Short note
These are my personal notes.
As such, they may be incomplete and inaccurate at times.
If you feel I misrepresented a speaker or a topic, please reach out to me so that we can collaborate on improving this article.
I was also trying to participate in some of the chats simultaneously, so I could not take a complete set of notes for the latter part of the event.
I intend to relisten to the later portions of the event and write more about it once the videos are available.

I also plan to write a more sophisticated post on VEX later.

## Opening
* SBOM will give us visibility into what is in our software.
* SBOM will provide us with an understanding of how our software works 

## History of SBOM by Josh Corman
During the launch of healthcare.gov, significant issues led to the discovery that significant gaps in the law did not cover integrated code, only code written.
Heartbleed in OpenSSL showcased this gap nicely.
Organizations had significant difficulty determining where they were affected.
In 2014, the "Cyber Supply Chain Management and Transparency Act of 2014" was introduced but did not gain traction and did not enter law.
In 2015 CISA taskforce was started.
In 2016, an attack on MedStar Health via a ransomware attack led to ambulance and critical medical care interruption. [https://www.csoonline.com/article/3048825/ransomware-attack-hits-medstar-health-network-offline.html](CSO Online link)
In 2019, the FDA developed premarket guidance for FDA-approved medical devices.
There was worry that this would lead to a de facto standard without taking inputs from other industries.
Allan Friedman stepped up in NTIA to help create SBOM guidance that could span multiple industries.
In parallel, the financial industry applied Deming Principles to identify where software is for the express purposes of *cost savings*.
In short, the faster we can identify where we are affected, the more lives will be saved and the less cost to mitigate/fix vulnerabilities.

## Roles & Benefits by Audra Hatch

SBOMs help finds where components are used, which helps mitigate risk and adds additional context to establish trust.
For example, if we track IV pump components, ideally, we can follow each part as it progresses through the chain.

```
part -> compound part -> final goods assembled -> operator
```

If we cannot remediate vulnerabilities upstream, the whole chain is vulnerable.
The takeaway is we're all in the supply chain, with most of us in the middle.
Without a BOM, our supply chain is opaque with many unknown flaws that could be easily mitigated had you known what was in your final assembled item.
Popular compound parts manufacturer with SBOMs leads to herd immunity.
In a world without SBOM, remediation requires serial notification and hand-offs, with many opportunities for errors or miscommunication.
With SBOMs, downstream consumers can plan and mitigate before the upstream producer begins mitigation steps.
Complete visibility leads to more awareness of the risk.
We can see more depth of the tree instead of just the first layer below us.
"We have found no shortage of examples where SBOMs were effective" - Audra Hatch.
Benefits span across the whole supply chain and all sectors.
Deming principle lets us pick the highest quality suppliers across sectors, with cumulative effects.

## Practitioner Perspective by Cassie Crossley (Schneider Electric)

Schneider Electric started using SBOMs for vulnerability management.
Suppliers need to be identified and targeted for SBOMs.
Schneider Electric did SBOM adoption initially for impact assessment and developer time.
You may not have staff on immediate call for a given problem.
SBOM gives us tools to determine which staff needs to be brought on in the critical infrastructure.
Cassie Crossley is significantly involved in the energy POC and has released multiple playbooks about supply chain defense and SBOM.

## Practitioner Perspective
Jennings Aske CISO for (NYP)

Jennings is a lawyer turned CISO.
During the heartbleed event, he couldn't tell customers whether they were affected by it.
From then on, Jennings began to research SBOMs and has promoted them since.
For every piece of software, and SBOM is created. 
Without visibility, consumers cannot determine the quality of software.
Better information leads to better procurement and management.
SBOMs are designed to provide additional information to determine the quality of software.
Informed buyers can make better decisions and build more effective partnerships.
Jennings is also announcing Daggerboard in January. 

## What is an SBOM? Art Manion (SEI)

Every vulnerability mitigation involves the supply chain.
Who was affected by log4j? Supply chain analysis on SNMP led to a similar answer to log4j: nearly everyone.
The fundamental problem has existed for over 20 years.
These topics are explored in more depth in the Framing Software Component Transparency document available at ntia.gov/sbom.

## SPDX and CycloneDX
Basics for both SPDX and CycloneDX were covered.
Highlights for me include:

* Differentiating between "source" and "binary" SBOMs,
* Differentiating between producers, consumers, and transformers,
* Defined document types/profiles which can be composed to convey rich information.

## NTIA Consumer and Supplier Playbooks by JC Herz
JC introduced the consumer and supplier playbooks, which are available on ntia.gov/sbom.

I strongly recommend reading through these documents, of which JC is a core co-author.

## VEX
I will be covering VEX in a separate article.

VEX is a specification where a vendor can declare whether their software is affected or not affected by a given vulnerability, such as a CVE. VEX is a parallel specification designed to integrate SBOM. It is not a part of the SBOM itself but instead can be correlated with VEX.

CSAF 2.0 is a JSON-based open standard designed by the OASIS community that is machine-readable and designed for automation.
CSAF 2.0 is designed as the successor to CSAF CVRF 1.2
VEX is a CSAF 2.0 profile implemented as a profile in CSAF.

## Minimum Elements
The minimum elements are available at https://ntia.gov/files/ntia/publications/ntia_sbom_framing_2nd_edition_20211021.pdf

This guidance is just a starting point. The recommended elements may change as the industry adopts SBOMs and discovers best practices.

We're taking a crawl/walk/run approach, where we are still learning the basics but need to transition to a walk phase.

## PoC efforts
There were three highlighted PoC efforts:
* Healthcare
* Energy
* Automotive

Each highlights initial industry work to determine how SBOMs relate to their respective industries. There is a healthcare section in https://ntia.gov/sbom where you can read the current results of the Healthcare POC.

## Some final thoughts
The speakers did not cover several questions on the first day of the event. A short survey of some of these topics include:

* Protecting SBOMs from tampering.
* How to go from SBOMs to CVEs.
* How do we deliver SBOMs in containerized environments? e.g., Microsoft's Ratify project.
* How can SBOMs be used to strengthen zero trust.
* Vulnerabilities shouldn't be in an SBOM.
* Why we should bias towards source-based SBOM generation at build time.
* How do we capture information on dynamic systems? These are likely out of scope for SBOMs.
* How do we use an SBOM to inform SaaS-based solutions? (spoiler: The answer isn't SBOMs as the initial point of contact).

[Continue to the Day 2 Write-Up]({{< ref "/posts/2021-sbom-a-rama-day-2">}})
