---
title: "[FIXED]: GCP BuildPacks Old Compiler Injection Write-Up"
date: 2022-02-03T22:29:38-08:00
draft: true
series:
 - sbom
 - buildpacks
 - vulnerabilities
---
# [FIXED]: GCP BuildPacks Old Compiler Injection Write-Up

**I would like to personally thank the GCP BuildPacks team for supporting this important project and fixing this issue!**

This article describes a **FIXED** vulnerability in GCP BuildPacks that I discovered and collaborated with the GCP team to fix. The buildpack now downloads the most recent stable compiler, effectively fixing the problem. 

The short version of this report is the GCP Golang Buildpack used to pull in an old, no-longer-maintained compiler. Today, it pulls in the latest stable compiler.

This bug also highlights the need for SBOM and in-toto attestations, including compiler information. People will make mistakes, and these tools will help prevent or detect these errors. A properly attested SBOM will eventually allow automated processes to detect and report vulnerabilities such as in this this write-up.

A side effect of fixing this bug in the GCP BuildPack is all future builds using this BuildPack will be protected from this failure mode.


Products:
* https://cloud.google.com/blog/products/containers-kubernetes/google-cloud-now-supports-buildpacks
* https://github.com/GoogleCloudPlatform/buildpacks

Fix: 
* https://github.com/GoogleCloudPlatform/buildpacks/commit/9b81ec3cca918acae5c1f82ba3d1dcf92c649986

# Write-up

Before the fix, GCP BuildPacks originally pulled in older compilers based upon the level of compatibility rather than the latest stable version.

For example, thub.com/fkautz/serve has the following go.mod:

```
module github.com/fkautz/serve

require (
        github.com/cpuguy83/go-md2man/v2 v2.0.0 // indirect
        github.com/felixge/httpsnoop v1.0.2 // indirect
        github.com/gorilla/handlers v1.5.1
        github.com/russross/blackfriday/v2 v2.1.0 // indirect
        github.com/urfave/cli v1.22.5
)

go 1.13
```

The buildpack would pull in the original `go 1.13` compiler and build the project. Using an older compiler to build resulted in insecure binaries with known vulnerabilities originating from the old compiler and golang stdlib.


```
$ pack build -B gcr.io/buildpacks/builder:v1 fkautz/serve

[omitted...]

=== Go - Runtime (google.go.runtime@0.9.1) ===                                                                                               
Using runtime version from go.mod: 1.13                                                                                                      
Installing Go v1.13                                                                                                                          
--------------------------------------------------------------------------------             
Running "bash -c curl --fail --show-error --silent --location --retry 3 https://dl.google.com/go/go1.13.linux-amd64.tar.gz | tar xz --directo
ry /layers/google.go.runtime/go --strip-components=1"                                                                                        
Done "bash -c curl --fail --show-error --silent --location --retry..." (9.997062215s)

[omitted...]
```

## GCP Build Pipelines Affected

I tried this with the buildpacks build feature in google cloud with a go1.14 compatible project and got the following:

```sh
gcloud alpha builds submit --pack image=gcr.io/fkautz-dev/sample-go
===> ANALYZING                                                        
[analyzer] Previous image with name "gcr.io/571261452737/sample-go" not found
===> RESTORING                    
===> BUILDING                     
[builder] === Go - Runtime (google.go.runtime@0.9.1) ===
[builder] Using runtime version from go.mod: 1.14
[builder] Installing Go v1.14                                         
[builder] --------------------------------------------------------------------------------
[builder] Running "bash -c curl --fail --show-error --silent --location --retry 3 https://dl.google.com/go/go1.14.linux-amd64.tar.gz | tar xz
 --directory /layers/google.go.runtime/go --strip-components=1"
[builder] Done "bash -c curl --fail --show-error --silent --location --retry..." (3.590012839s)
```
The system pulls in the oldest compiler rather than the latest golang compiler. You can see it pulling go1.14 instead of go1.17.6.

# Resolution
The most recent version of GCP BuildPacks now pulls in the latest compiler. The above is now fixed and deployed.

The fix is located in [this commit](https://github.com/GoogleCloudPlatform/buildpacks/commit/9b81ec3cca918acae5c1f82ba3d1dcf92c649986).

