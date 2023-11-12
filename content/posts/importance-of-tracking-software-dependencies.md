---
title: "On the Importance of Tracking Software Dependencies"
date: 2023-11-12T12:44:00-06:00
draft: false
series:
 - sbom
 - log4shell
 - vulnerabilities
---
# On the Importance of Tracking Software Dependencies

While contributing to the development of NIST SP 800-204D, a primary objective I pursued was to address a particular deficiency in the realm of software supply chain security. Here's the critical insight from this endeavor as written in the document:

> SSC security should also account for discovering and tracking software defects rather than simply mitigating attackers.

Pick any widespread library commonly deployed through a non-distribution based package manager and ask the question: "Do I know where this is deployed in my organization?" I'm guessing the answer as of the time of writing is a resound no for all but the smallest or most disciplined teams.

## The Rationale Behind This Inclusion

While thwarting cyber-attacks is crucial, we must also recognize and mitigate risks inherent in the production and usage of software arising from the interconnectedness of today's software, regardless of whether it is open source or closed. A pertinent example is the industry's response to the Log4Shell vulnerability.

Log4Shell's emergence, as far as we know, was not the result of any deliberate malicious intent. There's no indication that the developers aimed to inflict harm. Yet, the predominant focus in securing software supply chains has been primarily on averting unauthorized or malevolent activities within the supply chain. Discussions around Software Bill of Materials (SBOMs) often focus on detecting tampering. Paradoxically, despite the non-malicious origin of Log4Shell, insufficient effort has been directed at addressing the underlying issues that made Log4Shell so devastating.

Addressing Log4Shell entailed substantial costs, primarily in identifying and patching affected systems. Large organizations tasked their development, operations, and IT teams with scrutinizing their software to determine the presence of a vulnerable Log4j installation. The challenge of limited visibility and accountability hindered the response teams, delaying and complicating their efforts to counteract this threat effectively.

Moreover, understanding where the compromised libraries and software are utilized becomes imperative if we envisage a scenario where a sophisticated adversary orchestrates a future vulnerability. This underscores the need for an accurate and exhaustive inventory beyond just SBOMs. It's essential to pinpoint the deployment locations of affected products and their libraries. Information regarding software content needs to transcend organizational limits. Furthermore, any viable solution must also safeguard the confidentiality and privacy of developers and end-users (a topic I plan to explore in-depth over time).
