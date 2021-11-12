---
title: "Creating an SBOM for a golang app using cyclonedx-gomod"
date: 2021-11-11T01:18:52-08:00
draft: false
---

We can easily create CycloneDX SBOMs for Golang using a new tool from the CycloneDX team.

We begin by installing the tool to `$GOPATH/bin`

```sh
go install github.com/CycloneDX/cyclonedx-gomod@v1.0.0
```

For this example, we use the example repo which was used in the NTIA SBOM plug fests: [github.com/fkautz/serve](https://github.com/fkautz/serve).

```sh
git clone https://github.com/fkautz/serve
```

The cyclonedx-gomod tool provides three subcommands which we can use to create the submodule.

* app: generates an sbom for an application, only including what is actually in the final binary.
* bin: generates an sbom from an existing binary.
* mod: generates an sbom from all imported dependencies, ignoring build constraints.

In this scenario, we will produce an SBOM using `app`.

```sh
cyclonedx-gomod app > sbom.xml
```


We will analyze the contents of the SBOM in a future post.

The final version of the app is as follows:


```xml
<?xml version="1.0" encoding="UTF-8"?>
<bom xmlns="http://cyclonedx.org/schema/bom/1.3" serialNumber="urn:uuid:072ddef2-5299-4131-8554-b143b41f6b86" version="1">
  <metadata>
    <timestamp>2021-11-11T18:48:25-08:00</timestamp>
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
  </metadata>
  <components>
    <component bom-ref="pkg:golang/github.com/cpuguy83/go-md2man/v2@v2.0.0" type="library">
      <name>github.com/cpuguy83/go-md2man/v2</name>
      <version>v2.0.0</version>
      <scope>required</scope>
      <hashes>
        <hash alg="SHA-256">1285034b469f6ebb17019f5843d8ddbbf999dac5e04f5ff6cb23678383c69723</hash>
      </hashes>
      <purl>pkg:golang/github.com/cpuguy83/go-md2man/v2@v2.0.0</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/cpuguy83/go-md2man</url>
        </reference>
      </externalReferences>
    </component>
    <component bom-ref="pkg:golang/github.com/felixge/httpsnoop@v1.0.2" type="library">
      <name>github.com/felixge/httpsnoop</name>
      <version>v1.0.2</version>
      <scope>required</scope>
      <hashes>
        <hash alg="SHA-256">fa74bd83cd8a31771c27fc29d33c915bd6411c511398c1ad924fb60934eb5b8a</hash>
      </hashes>
      <purl>pkg:golang/github.com/felixge/httpsnoop@v1.0.2</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/felixge/httpsnoop</url>
        </reference>
      </externalReferences>
    </component>
    <component bom-ref="pkg:golang/github.com/gorilla/handlers@v1.5.1" type="library">
      <name>github.com/gorilla/handlers</name>
      <version>v1.5.1</version>
      <scope>required</scope>
      <hashes>
        <hash alg="SHA-256">f65458ea3f0311e7814f5d02bcef61196d209a4cb4069ae7bc3239bdf8541c7e</hash>
      </hashes>
      <purl>pkg:golang/github.com/gorilla/handlers@v1.5.1</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/gorilla/handlers</url>
        </reference>
      </externalReferences>
    </component>
    <component bom-ref="pkg:golang/github.com/russross/blackfriday/v2@v2.1.0" type="library">
      <name>github.com/russross/blackfriday/v2</name>
      <version>v2.1.0</version>
      <scope>required</scope>
      <hashes>
        <hash alg="SHA-256">248387e79ff4716c8eba296bf7faa5ae6d0149795daa7ab032c7f7e4b77aee69</hash>
      </hashes>
      <purl>pkg:golang/github.com/russross/blackfriday/v2@v2.1.0</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/russross/blackfriday</url>
        </reference>
      </externalReferences>
    </component>
    <component bom-ref="pkg:golang/github.com/urfave/cli@v1.22.5" type="library">
      <name>github.com/urfave/cli</name>
      <version>v1.22.5</version>
      <scope>required</scope>
      <hashes>
        <hash alg="SHA-256">94dabdb001d72b6a9f748f16f86448b63084908fb6a11e1df8c107cb508a5e85</hash>
      </hashes>
      <purl>pkg:golang/github.com/urfave/cli@v1.22.5</purl>
      <externalReferences>
        <reference type="vcs">
          <url>https://github.com/urfave/cli</url>
        </reference>
      </externalReferences>
    </component>
  </components>
  <dependencies>
    <dependency ref="pkg:golang/github.com/fkautz/serve@v1.0.0">
      <dependency ref="pkg:golang/github.com/gorilla/handlers@v1.5.1"></dependency>
      <dependency ref="pkg:golang/github.com/urfave/cli@v1.22.5"></dependency>
    </dependency>
    <dependency ref="pkg:golang/github.com/cpuguy83/go-md2man/v2@v2.0.0">
      <dependency ref="pkg:golang/github.com/russross/blackfriday/v2@v2.1.0"></dependency>
    </dependency>
    <dependency ref="pkg:golang/github.com/felixge/httpsnoop@v1.0.2"></dependency>
    <dependency ref="pkg:golang/github.com/gorilla/handlers@v1.5.1">
      <dependency ref="pkg:golang/github.com/felixge/httpsnoop@v1.0.2"></dependency>
    </dependency>
    <dependency ref="pkg:golang/github.com/russross/blackfriday/v2@v2.1.0"></dependency>
    <dependency ref="pkg:golang/github.com/urfave/cli@v1.22.5">
      <dependency ref="pkg:golang/github.com/cpuguy83/go-md2man/v2@v2.0.0"></dependency>
    </dependency>
  </dependencies>
</bom>
```