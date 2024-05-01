---
title: "Tarballs are Both Reproducible and Non-Reproducible!"
date: 2024-04-29T08:00:00-08:00
draft: false
series:
 - sbom
---

# TLDR:

* You can rely on the inputs and outputs being repeatable, as long as the tarball captures the data. (E.g. does yoru tarball capture extended attributes)
* You cannot rely on the .tar being bit-for-bit repeatable.
* You can rely on a (lossless) compression algorithm’s input and output being repeatable.
* You cannot rely on the actual compressed form being repeatable.

**Author’s Note:** There’s a common suggestion that the hash of a tarball from a directory could be used to ascertain the system’s state. This approach might be viable if you’re downloading a signed tarball from a trusted source. However, it’s often proposed as a method within active software development contexts, such as using CI/CD systems to snapshot the state of data or shared objects directories by creating and hashing a tarball. In practice, this method faces challenges due to the inherent non-repeatability of tarball creation, which means that even slight variations in the tar process can lead to different hashes, thus making this approach unreliable for consistent state verification.

# The Challenge of Non-Repeatability

Tarballs are created using the tar utility, which can exhibit different behaviors based on the version and configuration. This includes how metadata is handled, the order of files in the archive, and the specifics of compression techniques used. Such differences, although sometimes subtle, can lead to significant challenges in environments that rely on deterministic builds—where the same source code should always produce an identical binary output.

# Impact on Software Supply Chain Security

The non-repeatability of tarball creation can introduce complexity into the software supply chain. Inconsistent byte-representation of the tarballs themselves limit their utility for this specific problem. Even minor differences in configuration or `tar` patch level may result in different output, negating the intended effect.

# Strategies for Achieving Repeatable Builds

To combat these challenges and enhance the security of the software supply chain, the following strategies can be employed:

* Use other techniques to determine the state of the directory. Golang’s sumdb.HashDir provides one such example. My work with OmniTrail also provides a solution to this problem.
* Explicit Configuration: Define and enforce consistent configurations and flags for all tools involved in the build process to ensure they behave identically in all environments.
* Short Term Approach - Standardize Tooling: Utilize the exact same versions of tools like tar and compression utilities across all development and build environments to minimize variability. However, you will likely see divergence over time as these tools receive security and feature patches.

