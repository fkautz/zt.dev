---
title: "What is VEX? It's the Vulnerability Exploitability eXchange!"
date: 2021-12-19T23:50:30-07:00
draft: false
series:
 - sbom
 - vex
 - nvd
---

# The Problem

CVEs play an essential role in announcing vulnerabilities for a given product or library.
Recently, we saw the high-impact announcement of CVE-2021-44228 (Log4Shell).
Analysts and engineers are inspecting every installation of Java for susceptibility of Log4Shell. 

However, the presence of a vulnerability does not necessarily mean that a product consuming that library is affected.
An implementation may not exercise the vulnerable feature. 
Alternatively, the data sent to the vulnerability may be sufficiently sanitized, preventing the exploit from materializing.
The presence of a vulnerability indicates risk but does not indicate whether a product is affected or not.
Applications depending on libcurl 7.64.1, which is vulnerable to CVE-2019-5436 (libcurl TFTP buffer overflow), are not affected by the vulnerability if they do not use TFTP.
Assuming SBOM adoption solves the problem of knowing what is in our infrastructure, we still don't know whether a given product is *affected* by a vulnerability.

If the presence of a CVE is not indicative of being affected by a CVE, how do we determine whether we need to take action?

Options include but are not limited to:

* Customer may create a policy to *ALWAYS* upgrade when a CVE is present. However, this may increase the total cost of ownership higher than necessary.
* A vendor may post a message on a known website or blog claiming they are not vulnerable.
* Customers may ask their product/vendor representative whether they are affected by the vulnerability.
* Customers may **mitigate** the vulnerability with secondary controls (e.g., firewall, IDS).
* Customers may **accept** the risk.
* Customers may **eliminate** the risk by disabling the product.
* (In some cases, customers may also **transfer** the risk via insurance.)

The above actions are necessary because of a major gap:

> ***There is no machine-readable industry-standard protocol to state whether a product is affected by a vulnerability, nor under what conditions result in being affected by a vulnerability.***

# Enter VEX

VEX is a profile in the [Draft CSAF 2.0 Standard](https://github.com/oasis-tcs/csaf/blob/master/csaf_2.0/prose/csaf-v2-editor-draft.md) which allows for organizations to make a statement about whether a product is **affected** by a vulnerability. 

Every VEX document has the following elements:

* Product Tree - Lists all products referenced in the CSAF document.
* Affected status - Whether the product is affected by the vulnerability, regardless of the presence. The `status` may be one of the following:
  * Fixed
  * Known Affected
  * Known Not Affected 
  * Under Investigation
* At least one vulnerability CVE or ID (Not all vulnerabilities may have a CVE, but should have at least one)
* Notes - details about the vulnerability.

If a product is known to be affected, the vendor must include additional remediation information. It is also possible to set **no_fixed_planned** or **none_available** as remediation actions.

If a product is listed at **known not affected**, an impact statement with details must contain why the product is not affected.

[A complete example with a VEX document is provided in the CSAF 2.0 examples](https://github.com/oasis-tcs/csaf/blob/master/csaf_2.0/examples/csaf/CVE-2018-0171-modified.json)

In the example above, I truncated the `product_tree` and `known_affected` to focus on the elements of the vulnerability statement. 


Truncated VEX document:
```json
{
  "document" {
    "most fields skipped for brevity": "",
    "tracking": {
      "id" "cisco-sa-20180328-smi2",
      "status": "final",
      "version": "3.0.0",
      "revision_history": [
        { "other versions skipped for brevity":""},
        {
          "number":"3.0.0",
          "date":"2018-04-17T15:08:41Z",
          "summary": "Updated IOS Software Checker with products found to be vulnerable."
        }
      ],
      "initial_release_date": "2018-03-28T16:00:00Z",
      "current_release_date": "2018-03-28T16:00:00Z",
      "generator": {
        "engine" {
          "name": "TVCE"
        }
      },
    },
  },
  "product_tree": {
    "name": "Cisco",
    "category": "vendor",
    "branches": [
      {
        "name": "IOS",
        "category": "product_name",
        "branches": [
          {
            "name": "12.2EY",
            "category": "product_version",
            "branches": [
              {
                "name": "12.2(55)EY",
                "category": "service_pack",
                "product": {
                  "product_id": "CVRFPID-103559",
                  "name": "Cisco IOS 12.2EY 12.2(55)EY"
                }
              }
            ]
          }
        ]
      }
    ]
  },
  "vulnerabilities": [
    {
      "title": "Cisco IOS and IOS XE Software Smart Install Remote Code Execution Vulnerability",
      "id": {
        "system_name": "Cisco Bug ID",
        "text": "CSCvg76186"
      },
      "notes": [
        {
          "title": "Summary",
          "category": "summary",
          "text": "A vulnerability in the Smart Install feature of Cisco IOS Software and Cisco IOS XE Software could allow an unauthenticated, remote attacker to trigger a reload of an affected device, resulting in a denial of service (DoS) condition, or to execute arbitrary code on an affected device.\n\n\n\nThe vulnerability is due to improper validation of packet data. An attacker could exploit this vulnerability by sending a crafted Smart Install message to an affected device on TCP port 4786. A successful exploit could allow the attacker to cause a buffer overflow on the affected device, which could have the following impacts:\n\n\n    Triggering a reload of the device\n    Allowing the attacker to execute arbitrary code on the device\n    Causing an indefinite loop on the affected device that triggers a watchdog crash"
        },
        {
          "title": "Cisco Bug IDs",
          "category": "other",
          "text": "CSCvg76186"
        }
      ],
      "cve": "CVE-2018-0171",
      "product_status": {
        "known_affected": [
          "CVRFPID-103559"
        ]
      },
      "scores": [
        {
          "products": [
            "CVRFPID-103559"
          ],
          "cvss_v3": {
            "version": "3.0",
            "baseScore": 9.8,
            "baseSeverity": "CRITICAL",
            "vectorString": "CVSS:3.0/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:H"
          }
        }
      ],
      "remediations": [
        {
          "details": "There are no workarounds that address this vulnerability for customers who require the use of Cisco Smart Install. For customers not requiring Cisco Smart Install, the feature can be disabled with the no vstack command. In software releases that are associated with Cisco Bug ID CSCvd36820 [\"https://bst.cloudapps.cisco.com/bugsearch/bug/CSCvd36820\"], Cisco Smart Install will auto-disable if not in use.\n\nAdministrators are encouraged to consult the informational security advisory on Cisco Smart Install Protocol Misuse [\"https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-20170214-smi\"] and the Smart Install Configuration Guide [\"http://www.cisco.com/c/en/us/td/docs/switches/lan/smart_install/configuration/guide/smart_install/concepts.html#23355\"].",
          "category": "workaround"
        }
      ],
      "references": [
        {
          "url": "https://tools.cisco.com/security/center/content/CiscoSecurityAdvisory/cisco-sa-20180328-smi2",
          "summary": "Cisco IOS and IOS XE Software Smart Install Remote Code Execution Vulnerability"
        }
      ]
    }
  ]
}
```

In this example, we see:

* A document containing metadata:
  * Document summary
  * CSAF Version information
  * Publisher information
  * Document Tracking Information
* Product Tree
  * Itemizing the products in question.
* Vulnerability list
  * The title of the vulnerability in a human-readable format.
  * Cisco Bug ID
  * A human-readable summary of the vulnerability
  * The CVE ID of the (not all vulnerabilities may have a CVE)
  * A list of product statuses with each product ID
  * Any known remediations
  * References
  * A remediation describing a workaround.

# Gaps

* A VEX document is a claim about whether a software product is affected by a given vulnerability.
* Whether you should trust the publisher is left as an exercise to the reader.
* Vex does not provide a way to normalize vendor and product information.
* It may be difficult to reference a vulnerability if no CVE or common Bug ID exists.

There is also no subcomponent information to identify what CVEs may be present in the subcomponents of the product.

We are provided with:
* Software is affected by `vulnerability`
* Software is not affected by `vulnerability`

However, a consumed software component can have a CVE with no issued VEX statement.
Without the vendor's assistance, there is no way for the consumer to determine if there are uninvestigated CVEs in the subcomponent.

CSAF 2.0 does provide a URL path to download an SBOM in the `product_tree`, which may help resolve this issue.
This comes with the standard set of problems in URLs, such as broken links and a lack of non-repudiation.

The savvy security analyst will consume the product SBOMs when available and ask the vendor to issue a VEX statement about the susceptibility of library dependencies to known CVEs.

# Conclusion

In short, VEX provides us with a promising standard format to declare whether a product is vulnerable to a given vulnerability. 
Ideally, we will see VEX feeds become a standard tool that will assist security analysts and service operators in determining risk and prioritize work.
