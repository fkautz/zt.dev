---
title: "Creating an SBOM for a golang app using spdx-sbom-generator"
date: 2021-11-14T14:14:20-08:00
draft: false
---

The SPDX community created the `spdx-sbom-generator` that makes it trivial to create SPDX based SBOMs. In this post, we will generate an SPDX SBOM.

We begin by installing the tool to `$GOPATH/bin`

```sh
git clone https://github.com/opensbom-generator/spdx-sbom-generator.git
cd spdx-sbom-generator
go install cmd/generator
```

For this example, we use the example repo which was used in the NTIA SBOM plug fests: [github.com/fkautz/serve](https://github.com/fkautz/serve).

```sh
git clone https://github.com/fkautz/serve
```



In this scenario, we will produce an SBOM using `app`.

```sh
generator
```


We will analyze the contents of the SBOM in a future post.

The final version of the app is as follows:


```rdf
SPDXVersion: SPDX-2.2
DataLicense: CC0-1.0
SPDXID: SPDXRef-DOCUMENT
DocumentName: github.com/fkautz/serve
DocumentNamespace: http://spdx.org/spdxpackages/github.com/fkautz/serve-b4565823-039b-4a8c-95da-4d9334dc5b31
Creator: Tool: spdx-sbom-generator-source-code
Created: 2021-11-14T23:04:04Z


##### Package representing the github.com/fkautz/serve

PackageName: github.com/fkautz/serve
SPDXID: SPDXRef-Package-github.com.fkautz.serve
PackageVersion: 1ef9a7e
PackageSupplier: Organization: github.com/fkautz/serve
PackageDownloadLocation: git+https://github.com/fkautz/serve
FilesAnalyzed: false
PackageChecksum: SHA256: 018b2e6e536d5d9c19c8d3cf9e39cca3696d53b6d149675bc16f92120f9d6f9a
PackageHomePage: https://github.com/fkautz/serve
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

##### Package representing the github.com/felixge/httpsnoop

PackageName: github.com/felixge/httpsnoop
SPDXID: SPDXRef-Package-github.com.felixge.httpsnoop-v1.0.2
PackageVersion: v1.0.2
PackageSupplier: Organization: github.com/felixge/httpsnoop
PackageDownloadLocation: https://github.com/felixge/httpsnoop/releases/tag/v1.0.2
FilesAnalyzed: false
PackageChecksum: SHA256: a82e213272380f8124a9f8d237f422728e7628d0c397481a443fee4646eddc69
PackageHomePage: https://github.com/felixge/httpsnoop
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

##### Package representing the github.com/gorilla/handlers

PackageName: github.com/gorilla/handlers
SPDXID: SPDXRef-Package-github.com.gorilla.handlers-v1.5.1
PackageVersion: v1.5.1
PackageSupplier: Organization: github.com/gorilla/handlers
PackageDownloadLocation: https://github.com/gorilla/handlers/releases/tag/v1.5.1
FilesAnalyzed: false
PackageChecksum: SHA256: 6faab27101992ab3756fce7f80c14934675a9f5dd24261facafeb9f8d2305210
PackageHomePage: https://github.com/gorilla/handlers
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

##### Package representing the github.com/russross/blackfriday/v2

PackageName: github.com/russross/blackfriday/v2
SPDXID: SPDXRef-Package-github.com.russross.blackfriday.v2-v2.1.0
PackageVersion: v2.1.0
PackageSupplier: Organization: github.com/russross/blackfriday/v2
PackageDownloadLocation: https://github.com/russross/blackfriday/v2/releases/tag/v2.1.0
FilesAnalyzed: false
PackageChecksum: SHA256: e6d03584f031b220f3467486da8851e18564e741238b24f0b0985245148fd433
PackageHomePage: https://github.com/russross/blackfriday/v2
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

##### Package representing the github.com/cpuguy83/go-md2man/v2

PackageName: github.com/cpuguy83/go-md2man/v2
SPDXID: SPDXRef-Package-github.com.cpuguy83.go-md2man.v2-v2.0.0
PackageVersion: v2.0.0
PackageSupplier: Organization: github.com/cpuguy83/go-md2man/v2
PackageDownloadLocation: https://github.com/cpuguy83/go-md2man/v2/releases/tag/v2.0.0
FilesAnalyzed: false
PackageChecksum: SHA256: 6667d54b2dfc231ce3856454ba9dfb5bd057ef45dd15ef2905505f169b400043
PackageHomePage: https://github.com/cpuguy83/go-md2man/v2
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

##### Package representing the github.com/urfave/cli

PackageName: github.com/urfave/cli
SPDXID: SPDXRef-Package-github.com.urfave.cli-v1.22.5
PackageVersion: v1.22.5
PackageSupplier: Organization: github.com/urfave/cli
PackageDownloadLocation: https://github.com/urfave/cli/releases/tag/v1.22.5
FilesAnalyzed: false
PackageChecksum: SHA256: 6759f02badc8f60c1ab02d6825d4f1a0ba73d72734abf7a7f0dacc56228a9476
PackageHomePage: https://github.com/urfave/cli
PackageLicenseConcluded: NOASSERTION
PackageLicenseDeclared: NOASSERTION
PackageCopyrightText: NOASSERTION
PackageLicenseComments: NOASSERTION
PackageComment: NOASSERTION

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