---
title: "Buildpacks and SBOM Integration Opportunities"
date: 2022-01-10T04:00:00
draft: false
series:
 - sbom
 - buildpacks
---

# Buildpacks and SBOM integration
Buildpacks provide a natural location for integrating SBOMs into developer build environments and CI/CD workflows.

*Author's note: the goal here isn't to bash the current state. This is fantastic work and represents a great start, and can evolve into something much more robust. The first iteration should be small in scope and demonstrate opportunity, which this does beautifully.*

## A short introduction on Buildpacks
### Buildpacks Definition

Buildpacks provide a structured framework for building an application and layering the application on a well-defined runtime.

More precisely, a **buildpack** is an OCI (container) image with a set of applications that are executed in two phases:

- Detect: Is this buildpack applicable to the inputs?
- Build: Perform whatever action the buildpack is designed to perform

For example, a golang compiler buildpack may have the following phases:

- Detect: Is there a go.mod in the source code root?
- Build: run ```go install``` and copy the result to the correct layer

Every buildpack follows the above two-step process.

### Buildpacks are stackable
The developer can stack multiple buildpacks into pipelines which allow for more advanced configurations while maintaining the modularity of each step. E.g., I may create a set of buildpack that executes in the following order:


1. Compile protobuf definitions
2. Validate licenses for all project files
3. Validate licenses for all consumed libraries
4. Run code security scanner
5. Compile golang code

Each of these buildpacks performs one function (UNIX philosophy). The final result is an OCI image built using the well-defined process defined by the buildpack framework.

## SBOM integration
### Current Status
First, I'd like to applaud the Buildpacks team for putting an initial release towards generating an SBOM as seen at https://buildpacks.io/docs/buildpack-author-guide/create-buildpack/adding-bill-of-materials/

A quick analysis of the provided document:

```js
    {
      "name": "ruby",
      "metadata": {
        "version": "2.5.0"
      },
      "buildpacks": {
        "id": "examples/ruby",
        "version": "0.0.1"
      }
    }
```

In the example above, `name` is established in a ``[[BOM]]`` section in launch.toml. The build script of the buildpack populated the `metadata.version`. The `buildpacks` section lists the buildpacks which were involved in the creation of this image. A separate image I created with a nodejs builder using Google's buildpack resulted in an id of `google.nodejs.runtime` versioned at `0.9.2`.

The metadata presented above is all helpful information but is currently insufficient if you are trying to meet the requirements of the NTIA.

Likewise, this provides us with no ability to check the integrity of the build or runtime images. I can tell something named `examples/ruby` was executed but it has no integrity to determine if the buildpack was modified or replaced. We are also missing non-repudiation properties that describe whether the buildpack successfully performed its task.


### Opportunities
There are several opportunities to increase the integrity and non-repudiation claims of the SBOM. The following is a list of ideas which may be useful for someone who is looking to engage in increasing these capabilities. This is not a thorough investigation of how each can be added, but is meant to foster ideas for people to possibly persue.

#### More advanced SBOM information
Adding more information to the SBOM would be welcome. For example, which organizations provided the resulting OCI image and provided each buildpack with additional integrity and non-repudiation information for all buildpacks.

#### Application SBOM Generation

![An image showing build tool generated SBOMs being collected from each layer](/images/buildpacks-sbom-opportunities/buildpack-sbom-from-builds.png)

It may be possible to provide an environment that assists in creating and capturing SPDX and CycloneDX SBOMs. Capturing information about what each buildpack consumes and contributes would be of high value.

The compiler buildpack will likely need to produce the application SBOM. The buildpack framework itself will not have insights into what files or libraries were consumed by the build tools to create the actual build artifacts.

#### Capture metadata between layers

![An image showing metadata of changes in persistent layers being captured](/images/buildpacks-sbom-opportunities/buildpack-capture-metadata.png)

Each pack may contribute outputs that the buildpack framework should capture in the SBOM or by in-toto. Capturing metadata on the delta helps identify what each layer has contributed.

#### Lifecycle Attestation

![An image showing each buildpack layer's OCI SBOM being captured](/images/buildpacks-sbom-opportunities/buildpack-record-pack-sboms.png)

Buildpack lifecycle information may be captured and attested using in-toto, which provides cryptographic attestation that each action in the lifecycle occurred, including ordering of lifecycle tasks.

#### Generate the SBOM for the Resulting Image

![An image showing how all the generated SBOM information is captured in the resulting OCI image SBOM](/images/buildpacks-sbom-opportunities/sbom-generate-sbom.png)

The buildpacks framework may help facilitate the production of the application's SBOM by ensuring the generated SBOM is available once the packing process completes.

The OCI image should also have an SBOM in either (or both) SPDX and CycloneDX format, which references the application SBOM if available.

The resulting SBOM(s) should not be in the OCI image. Instead, it should be generated as an artifact that the CI/CD system can separately record.

#### Sigstore Support
Finally, the output of the buildpack and associated SBOMs may be recorded in sigstore or a similar transparency ledger. Best practices on achieving this when using the Buildpack Framework may be necessary. I suspect this is best left to the CI/CD rather than in a Buildpack, but demonstrating how to capture the information and integrate it with sigstore would be valuable.

## Summary

The Buildpack Framework provides an exciting path that may help facilitate the creation of SBOMs of portable build layers. There are opportunities for improvement which may accelerate the generation of SBOMs and in-toto attestations. 
