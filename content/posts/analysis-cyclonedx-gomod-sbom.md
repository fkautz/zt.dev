---
title: "Analysis of a cyclonedx-gomod generated SBOM"
date: 2021-11-13T13:29:52-08:00
draft: false
series:
 - sbom
---

We generated an SBOM with cyclonedx-gomod in a [previous post](/posts/creating-cyclonedx-gomod-sbom). In this post, we perform an initial analysis of the contents of the generated SBOM.

The focus of this analysis is on `cyclonedx-gomod@v1.0.0`.

# Punchline
Run cyclonedx-gomod with the following flags:

```sh
cyclonedx-gomod app --files --licenses --std
```

# Opening
```xml
<?xml version="1.0" encoding="UTF-8"?>
<bom xmlns="http://cyclonedx.org/schema/bom/1.3" serialNumber="urn:uuid:072ddef2-5299-4131-8554-b143b41f6b86" version="1">
```

We begin with an initial XML version and encoding of UTF-8. The `bom` tag represents the root bill of materials type defined in the schema listed in the `xmlns` parameter. The serial number is a Universal Resource Name with a unique UUID that identifies the document's `identity`. This UUID is likely randomly generated.

# Metadata
A metadata tag has three components:

## Timestamp
```xml
<timestamp>2021-11-11T18:48:25-08:00</timestamp>
```

The `timestamp` tag describes the SBOM's creation time.

## Tools
```xml
    <tools>
      <tool>
        <vendor>CycloneDX</vendor>
        <name>cyclonedx-gomod</name>
        <version>v0.0.0-unset</version>
        <hashes>
          <hash alg="MD5">0738f1e7bc821d0bb630f3ad00ba0b52</hash>
          <hash alg="SHA-1">b2cdc16eb791d7d2c6387e9ff566abf687518132</hash>
          <hash alg="SHA-256">c82f6cb776191aea89bad6513f0fde4654df6e78befc07cdaee73e3c5c9daa30</hash>
          <hash alg="SHA-384">f91edd9b60b9f73a22bc479daac7638e4e6f1bbb3bd49a5bac68f7ca3a632e07ddef253c70d235fa00602b992e7ea7c0</hash>
          <hash alg="SHA-512">133e9467c92948cd250cb71d24aa048cd6533c1f126baad59ff1983990a2bbfc02f7d3bc560a74f99eee7d5b10e57c20c0e8e50d6cd0e2697a7f3d795457b373</hash>
        </hashes>
      </tool>
    </tools>
```
The `tools` tag states that we are using cyclonedx-gomod to create the SBOM. Interestingly, the version is set to `v0.0.0-unset` despite installing from a specific tag:

```sh
go install github.com/CycloneDX/cyclonedx-gomod@v1.0.0
```

Five hashes are provided, which represent the hashes of the tool's binary. Each of the hashes matches what is in the SBOM.

```sh
% m5sum cyclonedx-gomod
0738f1e7bc821d0bb630f3ad00ba0b52  cyclonedx-gomod
% sha1sum cyclonedx-gomod
b2cdc16eb791d7d2c6387e9ff566abf687518132  cyclonedx-gomod
% ha256sum cyclonedx-gomod
c82f6cb776191aea89bad6513f0fde4654df6e78befc07cdaee73e3c5c9daa30  cyclonedx-gomod
% sha384sum cyclonedx-gomod
f91edd9b60b9f73a22bc479daac7638e4e6f1bbb3bd49a5bac68f7ca3a632e07ddef253c70d235fa00602b992e7ea7c0  cyclonedx-gomod
% sha512sum cyclonedx-gomod
133e9467c92948cd250cb71d24aa048cd6533c1f126baad59ff1983990a2bbfc02f7d3bc560a74f99eee7d5b10e57c20c0e8e50d6cd0e2697a7f3d795457b373  cyclonedx-gomod
```

## Component
```xml
    <component bom-ref="pkg:golang/github.com/fkautz/serve@v1.0.0" type="application">
      <name>github.com/fkautz/serve</name>
      <version>v1.0.0</version>
      <purl>pkg:golang/github.com/fkautz/serve@v1.0.0</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/fkautz/serve</url>
        </reference>
      </externalReferences>
      <properties>
        <property name="cdx:gomod:build:env:CGO_ENABLED">1</property>
        <property name="cdx:gomod:build:env:GOARCH">amd64</property>
        <property name="cdx:gomod:build:env:GOOS">linux</property>
        <property name="cdx:gomod:build:env:GOVERSION">go1.17.3</property>
      </properties>
    </component>
```


The component lists the name and version as expected. In the generated output, we receive a `purl` tag. A purl is a package URL that follows the specification listed at [https://github.com/package-url/purl-spec](https://github.com/package-url/purl-spec).

The component may also be listed in a variety of other formats enumerated in [https://cyclonedx.org/specification/overview/#components](https://cyclonedx.org/specification/overview/#components)

The external references property was automatically populated by the tool to include information introspected from .git/config. In this case, it fetched the upstream origin.

Properties show interesting, relevant information about how the application is compiled. E.g., CGO_ENABLED, GOARCH, GOOS, and GOVERSION are all properties that affect the result's binary and functional output. E.g., in go, you can name add `_linux` to the name of the file, which will cause that file to only be compiled on a Linux build. e.g., `syscalls_linux.go` might include system calls only available on Linux systems.

## Components

```xml
    <component bom-ref="pkg:golang/github.com/cpuguy83/go-md2man/v2@v2.0.0" type="library">
```

The tool generated a `components` tag after the `metadata` tag. Each component contains a bom-ref which describes a reference to where you can find other BOMs. Presumably, you can derive an SBOM location from `bom-ref` and recursively pull in additional SBOMs as necessary.

Every component also has a `name`, `version`, `scope`, `hashes`, and `externalReferences` defined using the same format as we saw in the metadata. The `scope` is included explicitly here since some build systems may have optional or excluded dependencies. In gomod's version, all scopes appear to be required.

## Dependencies
A list of dependencies is listed for each package. The `ref` for each package matches the `bom-ref` we saw in each component. The components represent a graph of dependencies and their associated dependencies recursively. The reference helps identify the import chain that leads to a specific package being pulled into a build.

# What's missing in the default?

## Files
The dependency metadata is stored in the default settings for an SBOM, but the file references and contents are explicitly excluded. For this test, I modified main.go as follows:

```golang
diff --git a/main.go b/main.go
index 13c0a7f..7cea2a4 100644
--- a/main.go
+++ b/main.go
@@ -27,7 +27,7 @@ import (

 func main() {
        app := cli.NewApp()
-       app.Name = "serve"
+       app.Name = "serve2"
        app.Usage = "Simple HTTP Server"
        app.Flags = []cli.Flag{
                cli.StringFlag{
```

This resulted in the following changes
```xml
2c2
< <bom xmlns="http://cyclonedx.org/schema/bom/1.3" serialNumber="urn:uuid:6d002c1b-9a00-4656-9d6f-da4fac202409" version="1">
---
> <bom xmlns="http://cyclonedx.org/schema/bom/1.3" serialNumber="urn:uuid:420928ac-a7b7-411e-b044-b41535bba92d" version="1">
4c4
<     <timestamp>2021-11-11T18:10:43-08:00</timestamp>
---
>     <timestamp>2021-11-12T18:19:09-08:00</timestamp>
```

Notice how the SBOM did not change despite the contents of main.go changing. This can be remedied using the `-files` flag when calling `cyclonedx-gomod`.

```sh
cyclonedx-gomod app -files > sbom.xml
```

results in

```xml
      <components>
        <component type="file">
          <name>main.go</name>
          <version>v0.0.0-3fb2132aec9c</version>
          <scope>required</scope>
          <hashes>
            <hash alg="MD5">f3ac7a60cd038e207b329d1c8568b293</hash>
            <hash alg="SHA-1">3fb2132aec9cf639f00409b508e27bdbf0c0527f</hash>
            <hash alg="SHA-256">39f3f4d5985c290926d957c689ba08486773e153f464c1d00f46bf4efd1ccd20</hash>
            <hash alg="SHA-384">eabc498f20af1247f83d5dbee2ead38a5ba476e2c0069448975a88bf7dbd2e4d6f91e36a482ee0e445ab1bb777270f7a</hash>
            <hash alg="SHA-512">d335ffab199b3ac2982580659a16a5a3e5e651b6094ff0cdc98c642da12073367e2bc671243d8fc05a6b42f75a5edff50e21897a4fb358506fc4472e773b2b13</hash>
          </hashes>
        </component>
      </components>
```

The `<version>` tag maps to a truncated `SHA-1` sum, rather than the version listed in git. 

The hashes are sufficient to tell whether a file has been modified, as long as you can trust the tool, build environment, and SBOM provider.

The `-files` flag will also cause every file in every dependency included in the application's build to be included in the manifest. This can lead to very large SBOMs when generating SBOMs for large projects. However, this behavior should help increase the overall accuracy of the SBOM itself.

## Golang STD inclusion
```xml
    <component bom-ref="pkg:golang/std@1.17.3" type="library">
      <name>std</name>
      <version>1.17.3</version>
      <description>The Go standard library</description>
      <scope>required</scope>
      <purl>pkg:golang/std@1.17.3</purl>
      <externalReferences>
        <reference type="documentation">
          <url>https://golang.org/pkg/</url>
        </reference>
        <reference type="vcs">
          <url>https://go.googlesource.com/go</url>
        </reference>
        <reference type="website">
          <url>https://golang.org/</url>
        </reference>
      </externalReferences>
    </component>
```

Adding the `--std` flag results in an std component tag. Unfortunately, there is no way to determine whether the std was modified from the information provided by the SBOM. Hashes and files are not included in the result.

## Go version
The tool indicates what version of golang was used as a property, but does not include any hashes or any other identifying information which may be used to identify custom compiler builds.

## Licenses
The tool will also analyze licenses for you and place the license in an `evidence` tag for each component.

```xml
      <evidence>
        <licenses>
          <license>
            <id>Apache-2.0</id>
          </license>
        </licenses>
      </evidence>
```

The `evidence` tag appears to be present only at the `component` level. If a specific file is under a separate license, the tool may miss the file's license. Hopefully, a future version of cyclonedx-gomod will solve this issue. One possible incomplete solution would be to look for `SPDX-License-Identifier` tags in the comments. There may also be other tags I am unaware of, which may help simplify this process.

An example in main.go:

```go
// Copyright (c) 2014-2021 Frederick F. Kautz IV
//
// SPDX-License-Identifier: Apache-2.0
//
// Licensed under the Apache License, Version 2.0 (the "License");
```

## Minimum Elements of an SBOM

* Supplier name - missing, but can be added after creation of the XML document and before signing in CI/CD.
* Component Name - present
* Version of the component - present
* Other unique identifiers - present, `bom-ref` is useful here.
* Dependency relationship - present at a component level. Some file level dependency is present at the global level, but intra-file dependency does not appear to be modelled.
* Author of the SBOM - missing, but can be added after creation of the XML document and before signing in CI/CD.
* Timestamp - present

# Conclusion
Overall, generating a golang SBOM in cyclonedx-gomod is trivial. I highly recommend running cyclonedx-gomod with the following flags to increase the overall utility of the generated SBOM and to help identify whether the contents of the inputs has been tampered with or whether the compiler is customized:

```sh
cyclonedx-gomod app --files --licenses --std
```

Some modification of the SBOM may be necessary to add the supplier name and SBOM author. Ideally, this would occur immediately after the XML generation and before the SBOM was signed by sigstore or an equivalent system. The SBOM results do not include sufficient Golang std support other than what version of the std was used.