---
layout: docs
title: Repositories
description: Learn how to setup your own Latte repository.
weight: 31
---

## Overview

Latte's dependency management system uses repositories to download the artifacts a project depends on. These repositories are usually HTTP web servers.

The `workflow { }` and `publishWorkflow { }` DSL shown on this page is covered in full on the [Workflows](../workflows/) page — including the complete list of processes (`cache`, `url`, `maven`, `s3`, `mavenCache`) and their attributes.

In most commercial environments, project's will use 2 different repositories:

1. A public repository for all of the open source artifacts
2. A internal repository for all of the companies artifacts

Company artifacts are usually IP and therefore most companies put these artifacts into a secure HTTP server that only their employees have access to.


## Latte's public repository

Latte provides a free repository for you to use. This repository is not like Maven Central in that it only contains the open source libraries and frameworks that Latte has added. The URL for this repository is: https://repository.lattejava.org

This repository is also the default repository that Latte will use when you specify the `standard()` set in your workflow like this:

~~~~ groovy
workflow {
  standard()
}
~~~~

## Maven Central

Latte can fetch artifacts directly from Maven Central (or any Maven-style repository). This is useful when a dependency is available on Maven Central but has not yet been added to the Latte public repository.

To fetch from Maven Central, use the `maven` workflow process:

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    maven(url: "https://repo1.maven.org/maven2")
  }
  publish {
    cache()
  }
}
~~~~

This configuration tells Latte to first check the local cache, then the Latte public repository, and finally Maven Central. The `standard()` workflow shorthand includes all three of these by default:

~~~~ groovy
workflow {
  standard()
}
~~~~

When Latte fetches an artifact from Maven Central, it reads the Maven POM file to determine the artifact's dependencies and translates them into Latte's dependency model. There are a few important differences to be aware of:

* **Semantic Versioning** — Maven does not enforce SemVer, but Latte does. If a Maven artifact has a non-SemVer version, Latte will reject it until a version mapping has been added to your project file.
* **License information** — Maven POMs often lack or have non-standard license declarations. Latte requires license information for all artifacts. When importing from Maven Central, Latte will attempt to read the license from the POM, but you may need to provide it manually.
* **Dependency scopes** — Latte maps Maven scopes (`compile`, `runtime`, `test`, `provided`) to Latte dependency groups. Maven's `optional` flag maps to the `compile-optional` group.

You can also use Maven repositories other than Maven Central. For example, to fetch from a custom Maven repository:

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    maven(url: "https://repo1.maven.org/maven2")
    maven(url: "https://maven.example.com/releases", username: "user", password: "pass")
  }
  publish {
    cache()
  }
}
~~~~

## Referencing a private repository

A private repository will house anything an organization uses but doesn't wish to publish publicly.

~~~~ groovy
workflow {
  standard() // for the Latte public repo
  fetch {
      url(url: "http://example.com/internal/", username: "repo-username", password: "repo-password")
  }
}
~~~~

### SemVer

Maven is not SemVer compliant, Latte is. If a Maven artifact has an invalid version, you will need to fix it.

> It should be noted that some Maven artifacts have incorrect SemVer versions that are compliant, but malformed. An example is that some projects put meta as pre-release information.
> For example, the PostgreSQL JDBC driver uses a Maven version like ~~~~ 9.3.1102-jdbc41~~~~ . This is actually a SemVer pre-release version. In SemVer, this version should really be ~~~~ 9.3.1102+jdbc41~~~~  because the JDBC implementation version is meta-data NOT pre-release data.

### Licenses

Maven doesn't require license information and the license information is not 100% standardized. Latte uses a strict set of licenses for artifacts and all artifacts must have 1 or more licenses. Therefore, you will need to specify the license(s) for each artifact you are importing.

Latte uses [SPDX license identifiers](https://spdx.org/licenses/) for all licenses. Any valid SPDX license ID can be used (e.g., `Apache-2.0`, `MIT`, `GPL-3.0-only`). In addition, Latte supports custom license types: `Commercial`, `Other`, `OtherDistributableOpenSource`, and `OtherNonDistributableOpenSource`.

Custom license types require the full license text because they are not standardized. In these cases, you must provide the license text to the command line tool.

### Exclusions

Maven allows an artifact to exclude transitive dependencies. Technically, this is a broken dependency graph. It often indicates that the transitive dependency was defined in the wrong scope. In most cases, the transitive dependency should be marked as optional or provided, but instead was put in the compile or runtime scope. Likewise, Maven projects often put test dependencies in the `compile` scope, forcing developers to exclude those dependencies.

Latte supports Maven exclusions, but keep in mind that these are broken artifacts, and you might need to do some modifications to your classpath in order for your application to run properly.

### Optional dependencies

Maven allows a dependency in any scope to be marked as optional. This is somewhat unnecessary for the `test` and `provided` scope, but invaluable for the `compile` scope. Latte doesn't provide this same ability. Instead, Latte uses a dependency group name `compile-optional`. These dependencies are included at compile-time, but not at runtime. This is nearly the exact same thing as the `provided` scope, but Latte breaks them into two separate groups.

## Repository layout

The standard repository layout uses the artifact group, project, name, version, and type to store artifacts. Here is the layout for the Commons Collection artifact:

~~~~ 
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1.jar
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1.jar.md5
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1.jar.amd
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1.jar.amd.md5
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1-src.jar
org/apache/commons/commons-collections/3.2.1/commons-collections-3.2.1-src.jar.md5
~~~~ 

Project's can publish multiple artifacts, each with different names. The directory name remains the same.

Latte uses MD5 files to ensure that the artifacts are valid when they are downloaded.

## Manually building AMD files

Sometimes you have no option but to manually build the AMD file for an artifact. The AMD file is the **Artifact Meta Data** that tells Latte about an artifact's dependencies and licenses. AMD files use JSON format. If you need to manually edit or create an AMD file, here is an example:

~~~~ json
{
  "licenses": [
    {"type": "Apache-2.0"}
  ],
  "dependencyGroups": {
    "compile": [
      {"id": "org.slf4j:jcl-over-slf4j:jcl-over-slf4j:1.6.4:jar"},
      {"id": "org.apache.lucene:lucene-queries:lucene-queries:4.0.0:jar"},
      {"id": "org.apache.commons:commons-fileupload:commons-fileupload:1.2.1:jar"}
    ],
    "runtime": [
      {"id": "org.codehaus.woodstox:wstx-asl:wstx-asl:3.2.7:jar"}
    ],
    "provided": [
      {"id": "javax.servlet:servlet-api:servlet-api:2.4.0:jar"}
    ]
  }
}
~~~~ 

The dependency ID format is `group:project:name:version:type`.

Dependencies can also specify exclusions to exclude transitive dependencies:

~~~~ json
{
  "licenses": [
    {"type": "Apache-2.0"}
  ],
  "dependencyGroups": {
    "compile": [
      {
        "id": "org.example:my-lib:my-lib:1.0.0:jar",
        "exclusions": ["org.example:exclude-me:exclude-me:jar"]
      }
    ]
  }
}
~~~~ 

Some licenses require license text. For these licenses, you can provide the text inside the license entry:

~~~~ json
{
  "licenses": [
    {"type": "MIT", "text": "Copyright (c) 2004-2013 QOS.ch ..."}
  ]
}
~~~~

## Build your own repository

The easiest way to host your own private Latte repository is with S3-compatible object storage. Simply create a bucket, configure access credentials, and add it to your workflow. Latte handles the rest.

Here is an example workflow that uses a private S3-compatible repository:

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    maven(url: "https://repo1.maven.org/maven2")
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
  }
  publish {
    cache()
  }
}
~~~~

If your S3 bucket is fronted by an HTTP server (such as a CloudFlare R2 custom domain or an S3 static website endpoint), you can also use the `url` process to fetch from it. The `url` process supports HTTP Basic Authentication, making it a good option for private repositories hosted behind a VPN or internal network:

~~~~ groovy
workflow {
  fetch {
    cache()
    url(url: "https://repository.lattejava.org")
    url(url: "https://my-repo.internal.example.com", username: global.repoUsername, password: global.repoPassword)
    maven(url: "https://repo1.maven.org/maven2")
  }
  publish {
    cache()
  }
}
~~~~

### S3-compatible providers

Latte works with any S3-compatible storage provider. Here are some popular options:

* **CloudFlare R2** - No egress fees, S3-compatible API. Set the region to `auto`. https://developers.cloudflare.com/r2/
* **AWS S3** - The original S3 service from Amazon. https://aws.amazon.com/s3/
* **MinIO** - Self-hosted, open-source S3-compatible storage. https://min.io/
* **Backblaze B2** - Low-cost S3-compatible storage. https://www.backblaze.com/cloud-storage
* **DigitalOcean Spaces** - Simple S3-compatible object storage. https://www.digitalocean.com/products/spaces
