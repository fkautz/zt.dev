---
title: "Inline analysis of the OMB Memorandum MB-22-09: Zero Trust Memo"
date: 2022-01-31T20:12:56-08:00
draft: false
series:
 - zt
 - identity
 - policy
 - cisa
---
# OMB M-22-09
This article is a long one.  It contains my detailed notes of the entire OMB.  I am not expecting most people to read through the whole thing.  It also includes some of my thoughts and opinions as of 30 January 2022.

You can read the original at https://www.whitehouse.gov/wp-content/uploads/2022/01/M-22-09.pdf

One significant omission is the OMB says nothing about Software or Hardware Supply Chain provenance.  I suspect we will see more on this in a later memo since EO 14028 explicitly calls it out.

## My thoughts

The OMB is following the guidance of CISA and NIST.  The deadline for some of the implementation is the end of FY 2024.  FY 2024 ends on 30 September 2024.

The largest callout I can make is the requirement for authentication/authorization to occur at the application level, rather than relying on network authentication as the primary mechanism.

The second-largest callout I can make is the required move from RBAC (Role-Based Access Control) to ABAC (Attribute-Based Access Control).

I have attempted to include as much as possible, but the memo is a lengthy document with detailed requirements.  Omissions will be present, but I have tried to match the document's spirit.

## Notes for OMB-22-09

### User Identity 
#### Enterprise-Wide Identity Systems and MFA
* Single Sign-On for all Users with device-proximity Multi-Factor Authentication.  One Time Password/Code (OTP) and response challenges are explicitly not allowed.  SMS, voice calls, and push notifications for authentication are also not allowed.
* Authentication and authorization explicitly occur at the application, not the network.  The federal government will eventually remove network authentication entirely.
* Password requirements have been modernized, eliminating the need for short-term rotation and reducing the character classes such as special characters.
* When possible, PIV and PIV-derived credentials should be used for primary authentication, aligning with OMB Memorandum M-19-17.  PIV credentials contain a certificate and key pair, typically stored on a secure card or device.
* Guidelines are given on integrating systems that cannot fully move to Zero Trust.  For example, a Privileged Access Management (PAM) system may obtain exceptions when no Zero Trust integration is possible.  However, PAM and similar solutions may not be a general solution to securing systems.
* Third-party organizations with MFA capabilities that feed into US Government systems have one year to move to phishing-resistant MFA.

#### Authorization
* Move from Role-Based Access Control (RBAC) to Attribute-Based Access Control (ABAC).
* ABAC authorization may include authenticated attributes and bind on claims such as identity, attributes of the accessed resource, and environment at access time.
* At least one "device signal" must accompany user identity should be present.

### Devices
#### Inventory
* Each agency must maintain a full inventory of devices per agency.
* Continuous Diagnostics of devices.
* CISA to design the Continuous Diagnostics and Mitigation (CDM) program.
* Agencies must deploy Endpoint Detection and Response required.
* Federal Civilian Agencies will be required to participate in the CDM program.
* CDM to expand to include cloud-oriented architectures and APIs for asset management.

#### Endpoint Detection and Response
* The US Govt will move towards proactive detection of cybersecurity incidents and threat hunting.
* Agency-selected solutions must adhere to EDR standards established by CISA.
* EDR reports must be available to CISA.
* When EDR is not possible, the agency must implement other Zero-Trust controls.

### Networks
The initial focus is on L4/7 layers, specifically DNS and HTTP traffic.  **I hope that other network concerns will also be addressed, such as BGP.**

* Agencies will move towards encrypted DNS with a protective program designed by CISA.
* All HTTP payloads and APIs traversing the network must be encrypted.
* Requirements will feed into FedRAMP. FedRAMP controls the standards all vendors must adhere to for receiving an "Authorization to Operate" in the US Government.

#### Network Visibility and Attack Surface
The document appropriately calls out an issue with monitoring tools.  Observability is a crucial requirement of Zero Trust.  However, it also poses a risk in regards to compromised observability platforms.

* Agencies should avoid static keys with broad access. 
* Use current standards such as TLS 1.3 for mitigating bulk decryption attacks.
* Deep packet inspection (DPI) should be performed when guarding highly sensitive data with small, predicable access patterns.
* Metadata should be recorded when DPI is not used.  See OMB Memorandum M-19-26.

#### Encrypted DNS

* Agencies must use encrypted DNS such as DNS-over-HTTPS(DoH) or DNS-over-TLS(DoT).
* All new agency-created software using DNS must use encrypted DNS.
* Agencies should not rely on automatic network discovery.
* All uses of unencrypted DNS that agencies cannot upgrade must be included in the Agency's Zero Trust Migration Report
* DNS is already traversing through CISA infrastructure.  CISA needs to support cloud infrastructure and mobile.

#### Encrypted HTTPS

* All HTTP traffic must be encrypted, including in internal environments.
* US will coordinate with browsers via the DotGov program to "preload" .gov as HTTPS. 
* One challenge is the use of internal sites that use .gov extension but do not use HTTPS
* DotGov project to work with browsers to force .gov encryption.

####  Encrypted E-Mail Traffic

* Actual e-mail itself has no clear path to encryption in transit with third parties.
* Will work with clouds to encrypt what it can.
* Will work with CISA to evaluate future technologies.

#### Enterprise-Wide Architecture and Isolation Strategy
NIST described three techniques for Zero Trust:
* Enhanced Identity Governance
* Local Micro-Segmentation
* Network-Base Segmentation

*In my opinion, this section has conflicting guidance.  The memo states that networks will be made publicly available over time.  This section claims that clouds are well suited for Zero Trust because of the virtualized isolation they provide.  However, this approach implicitly trusts the VPC network where the application is running.  This section appears to be antithetical to the concept of Zero Trust. **To be clear, I am not advocating to get rid of VPCs.  Instead, we cannot rely on VPCs as the primary mechanism to secure applications.** Use a multi-layered security approach, but ensure the primary controls are cryptographically secure and minimize the blast radius of any given component, including a compromised network.*

### Applications and Workloads

*In my opinion, this is one of the areas where we can see the most significant gains and will also be one of the most difficult to get buy-in for existing solutions.  An application designed with Zero Trust methodologies at its core will generally be more secure than a system designed with Perimeter Defense methodologies.  This section also calls for many process changes that will increase the application's security at the expense of higher initial CAPEX.  I would expect these CAPEX costs to be offset by lower OPEX since the software will generally be better architected with more mature architectures.  There is also a requirement to pick one application to migrate to the Internet on a deadline.  This requirement will force the agency to take on the Zero Trust in a controlled way.  There is also a call to move towards more Cloud-Native architectures with 12-factor style applications.*

#### Application Testing
* All software tested by dedicated security testing programs, regardless of origin.
* Required Security Assessment Reports are required to gain authorization to operate
* Increased requirements on the types of tests to be executed.  E.g., a vulnerability scanner alone would be insufficient.

#### Easily available third-party testing
* Use firms with third-party firms with good reputations to evaluate application security.
* CISA and GSA to create a program for the rapid procurement of security testing services, with an SLA of 1 month or a few days for high priority items.

#### Vulnerability Disclosure Program
* Agencies must participate in a public vulnerability disclosure program (VDP) by September 2022, including internal sites.
* CISA has established a VDP that agencies may use.
* CISA will establish requirements for vendors through FedRAMP.

#### Internet-accessible application
* All agencies must pick one "FISMA Moderate" application to operationalize and make accessible over the Internet fully.
* Selected workload may not use a VPN or network tunnel.
* Selected workload should integrate with the Government identity solution
* Shifting some "FISMA Low" before shifting the moderate workload is considered acceptable.

#### Discovering internet-accessible systems
* Agencies will establish internal inventories for tracking.
* CISA will perform external scanning of systems and send reports to the agencies.
* CISA and GSA will conduct vulnerability scan reports.
* CISA will monitor domain registrations and DNS.
* CISA owns the .gov domain, making it ideal to discover services.
* Agencies to provide a complete list of non-gov domains to CISA for inclusion.

#### Immutable workloads
* Agencies should deploy systems with Infrastructure as Code and with mature DevSecOps.
* Workloads should be immutable with automated deployments.
* Agencies should avoid manual changes.
* Software Development Lifecycle should include processes to promote reliable, scalable, predictable deployments.
* Agencies should follow CISA's Cloud Security Technical Reference Architecture for migrating third-party services from on-premise to cloud.

### Data
* Joint committee to be established by Fed Chief Data Officers and CISOs.
* Agencies to implement data categorization and security processes for managing data.
* Agencies must audit encrypted data at rest in the cloud on access.
* Agencies must work with CISA to establish an observability platform.  (OMB Memo M-21-31)

#### Data Security Strategy
* Security to go beyond "dataset" level controls.
* Processes are yet to be determined based on decisions from the joint committee.

#### Automated Security Responses
* SOAR (Security Orchestration, Automation, and Response) will be necessary.
* Rich information on types of data being accessed and who is accessing it will be necessary
* ML/AI approaches to categorize data will be necessary for scaling.
* The strategy will also require ML/AI for discovering anomalies in access patterns.
* Initial suggestions to be done by the joint committee.

#### Auditing Access to Sensitive Data in the Cloud.
* Encryption at rest is required.
* Encryption at rest does not protect data in use.
* Cloud-managed infrastructure may be relied upon to help close this gap.  
    * Cloud applications may use KMS, decryption, and audit logs managed by cloud infrastructure.
* Audit log of attempted decryption access (regardless of success/failure) is a requirement
* Audit system must include reliably logging to a separate system.
* Eventually, observability platforms will combine audit logs with other sources for more advanced monitoring.

#### Timely access to logs
* Federal Government to investigate both internal and third-party breaches.
* SLA and retention strategies are to be determined.
* Each agency must maintain a highly-capable SOC.
* OMB Memorandum M-21-31 establishes requirements, with deadlines set for achieving an initial maturity level.
* Cryptographic verification of logs is required.
* DNS must be logged.

### Other Considerations
* IPv6 rollout to continue, but must not slow down Zero Trust or Cloud Migration.
* When PIV or derived-PIV are unavailable, FIDO2 and similar devices are acceptable.
* OMB Memorandum M-15-13 did not require internal connections to be encrypted.  This memorandum increased the scope to now require internal connections use of encryption.

