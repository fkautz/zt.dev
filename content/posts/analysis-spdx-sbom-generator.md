---
title: "Analysis of a spdx-sbom-generator generated SBOM"
date: 2021-11-14T19:57:00-08:00
draft: false
---

We generated an SBOM with the SPDX SBOM Generator in a [previous post](/posts/creating-spdx-sbom/). In this post, we perform an initial analysis of the contents of the generated SBOM.

The focus of this analysis is on the latest main branch of [https://github.com/opensbom-generator/spdx-sbom-generator](https://github.com/opensbom-generator/spdx-sbom-generator) at `2d55f67b8b1fbcaa722bd22a54c3e406ffe884a9`.

The SPDX SBOM Generator also supports a rich set of languages:

* GoMod (go)
* Cargo (Rust)
* Composer (PHP)
* DotNet (.NET)
* Maven (Java)
* NPM (Node.js)
* Yarn (Node.js)
* PIP (Python)
* Pipenv (Python)
* Gems (Ruby)
* Swift Package Manager (Swift)

The generator is still early in development, so I do not expect all languages or features to be fully available at this time. I expect it to mature into a powerful tool that teams will rely on to generate rich SPDX based SBOMs.

# Opening
```yaml
SPDXVersion: SPDX-2.2
DataLicense: CC0-1.0
SPDXID: SPDXRef-DOCUMENT
DocumentName: github.com/fkautz/serve
DocumentNamespace: http://spdx.org/spdxpackages/github.com/fkautz/serve-b4565823-039b-4a8c-95da-4d9334dc5b31
Creator: Tool: spdx-sbom-generator-source-code
Created: 2021-11-14T23:04:04Z
```

## Format

SPDX supports two primary formats:

* SPDX - tag:value
* RDF - XML based

By default, the SPDX SBOM Generator exports an SBOM compliant to `SPDX-2.2`.

## SBOM Contents License

```yaml
DataLicense: CC0-1.0
```

The Data License is an exciting addition. A commonly overlooked property of software and related artifacts is the license of the document. This document is explicitly set with a [CC-0](https://creativecommons.org/share-your-work/public-domain/cc0/) which is effectively *Public Domain*. The license enables people and organizations to freely modify or share the SBOM with others without concern that a copyright violation will occur. It's important to note that legal or contractual obligations may help control the *correctness* of the SBOM. Signing the SBOM is also essential to preserve the provenance of the metadata.


## SPDXID, DocumentName, DocumentNamespace fields

```yaml
SPDXID: SPDXRef-DOCUMENT
DocumentName: github.com/fkautz/serve
DocumentNamespace: http://spdx.org/spdxpackages/github.com/fkautz/serve-b4565823-039b-4a8c-95da-4d9334dc5b31
```

`SPDXID` is an identifier field that allows values in other parts of the document to reference the current block. The `SPDXID` emitted by the tool here is set to `SPDXRef-DOCUMENT`. The `SPDXRef-DOCUMENT` identifier is reserved and appears to be an internal reference to the current document.

The DocumentName and DocumentNamespace allow for unique identification. The URI listed in the DocumentNamespace is randomly generated, providing a unique identifier for this specific document.

## Creator and Created fields
```
Creator: Tool: spdx-sbom-generator-source-code
Created: 2021-11-14T23:04:04Z
```

The Creator field is a required element that is formatted in a specific way. The formatting may include, according to the [spec](https://spdx.github.io/spdx-spec/v2-draft/document-creation-information/#681-description).

* Person - Person's name and optional email
* Organization - name and optional email
* Tool - toolidentifier-version

```
Author's note: Please `DO NOT REQUIRE` a person's legal name in this field. I will write
an article on this at a future date describing why this is dangerous.
```

In this case, the spdx-sbom-generator tool is listed. There is no version listed in this example since it was compiled from source code. In a proper release, `source-code` would be replaced with the tool's version number.

The tool included the timestamp of when the SBOM was generated.

# Package format

The first package generated in the SBOM is the top-level package being analyzed. We can also match the DocumentName with the PackageName (unknown if this is always the case). 


```yaml
PackageName: github.com/fkautz/serve
SPDXID: SPDXRef-Package-github.com.fkautz.serve
PackageVersion: 1ef9a7e
```

Every package begins with a PackageName which in Golang matches the package name.

The `SPDXID` for each package is unique. The format is `SPDXRef-[idstring]`, which suggests that `-Package-` is injected to help deconflict other references and make this reference more human-readable. 

The tool picked up metadata from git to generate the PackageVersion.

```
git-ref        1ef9a7e8507b43ffaf99160112b8765b7c860b22
spdx-generated 1ef9a7e
```

The PackageVersion may be a specific git tag or released package. In this case, we are building from a git repo, so the git hash is picked up instead.

## Package Supplier and Download Location

```yaml
PackageSupplier: Organization: github.com/fkautz/serve
PackageDownloadLocation: git+https://github.com/fkautz/serve
```
The tool generates an Organization tag with the git URL listed as the organization. The package download location also indicates how to retrieve the source, including which Version Control software is used (in this case, git). 

## File Analysis
```yaml
FilesAnalyzed: false
```
If the file contents have been analyzed, this setting may be set to true. The contents were not analyzed by this tool which results in a false value.

## PackageChecksum
```yaml
PackageChecksum: SHA256: 018b2e6e536d5d9c19c8d3cf9e39cca3696d53b6d149675bc16f92120f9d6f9a
```

The package checksum is a checksum of all the package contents, as long as the SBOM itself is not included in the package. I modified a file and was able to verify that the hash changed. This property helps increase non-repudiation in build systems since the consumer may compare the source hashes to the claims in the SBOM.

## PackageHomePage

```yaml
PackageHomePage: https://github.com/fkautz/serve

```
The `PackageHomePage` includes a link to the project's webpage. This field may be modified as necessary post-generation but before the SBOM is signed by sigstore (or similar).

## Package assertions
```yaml
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION
```

The default settings do not perform license analysis.

# Relationshipos

After each package and dependency is described in the format listed above, the relationship between packages is defined.

```yaml
Relationship: SPDXRef-DOCUMENT DESCRIBES SPDXRef-Package-github.com.fkautz.serve
Relationship: SPDXRef-Package-github.com.fkautz.serve DEPENDS_ON SPDXRef-Package-github.com.russross.blackfriday.v2-v2.1.0
Relationship: SPDXRef-Package-github.com.fkautz.serve DEPENDS_ON SPDXRef-Package-github.com.urfave.cli-v1.22.5
Relationship: SPDXRef-Package-github.com.fkautz.serve DEPENDS_ON SPDXRef-Package-github.com.cpuguy83.go-md2man.v2-v2.0.0
Relationship: SPDXRef-Package-github.com.fkautz.serve DEPENDS_ON SPDXRef-Package-github.com.felixge.httpsnoop-v1.0.2
Relationship: SPDXRef-Package-github.com.fkautz.serve DEPENDS_ON SPDXRef-Package-github.com.gorilla.handlers-v1.5.1
Relationship: SPDXRef-Package-github.com.gorilla.handlers-v1.5.1 DEPENDS_ON SPDXRef-Package-github.com.felixge.httpsnoop-v1.0.2
Relationship: SPDXRef-Package-github.com.cpuguy83.go-md2man.v2-v2.0.0 DEPENDS_ON SPDXRef-Package-github.com.russross.blackfriday.v2-v2.1.0
Relationship: SPDXRef-Package-github.com.urfave.cli-v1.22.5 DEPENDS_ON SPDXRef-Package-github.com.cpuguy83.go-md2man.v2-v2.0.0
```

The very first relationship listed is the reserved `SPDXRef-DOCUMENT` which points to the top-level package. The relationship between each package is described as:

```yaml
Relationship: [SPDXID-A] DEPENDS_ON [SPDXID-B]
```

Every dependency edge is defined in this format.

# What's missing in the default?

## Files
The files are not explicitly called out in the generated SPDX file. However, there does appear to be an SPDX spec for providing information on analyzed files. The tool at this time does not provide the analysis.

However, issues with non-repudiation are mitigated by the PackageChecksum. Any change to the file contents or metadata such as the name results in a change in the `PackageChecksum`. When `PackageChecksum` is combined with the `PackageDownloadLocation` and `PackageVersion`, enough information is present to enable non-repudiation as long as the packages are available for download.

## Only inputs are analyzed, not the environment
This tool is currently only analyzing the inputs as they exist in source control. Environmental details are currently not captured by this tool. Details about the CI/CD system used to build the software will need to be captured through other means.

Details not yet picked up include:

* Go compiler metadata such as version
* Golang standard library metadata or content

An SBOM doesn't have to pick up these environmental factors since other tools can cover the gap. Allowing other tools to capture and report this information will enable SPDX to provide a high-quality ontology for the set of inputs represented in the package. Future versions of the spec and tool could evolve to capture this information over time.

## Licenses
SPDX has a rich set of properties for capturing and documenting license information. The [SPDX License Identifiers](https://spdx.org/licenses/) is gaining traction throughout the industry. I could not get the tool to capture license information in Golang, but I expect this to be added in future versions. Also, license capture may work well in other languages. 

## Minimum Elements of an SBOM

* Supplier name - present
* Component Name - present
* Version of the component - present
* Other unique identifiers - present with unique namespaces
* Dependency relationship - present at a component level.
* Author of the SBOM - present as `Creator` in `SPDXRef-DOCUMENT`
* Timestamp - present

# Conclusion
Generating a Golang SBOM with spdx-sbom-generator is trivial. The tool is still under active development and has yet to achieve a 1.0 release. However, I expect this tool to become invaluable to the SPDX community as it matures. In particular, the licensing compliance information is rich and nuanced. There is enough information in the default settings to help determine whether the inputs have been modified as long as you trust the CI/CD and tool to report appropriately. 