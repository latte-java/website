---
layout: docs
title: Dependencies
description: You specify your project's dependencies using the "dependencies" definition in the project file.
weight: 24
---

## Installing dependencies

The easiest way to add a dependency to your project is with the `install` command:

~~~~ shell
$ latte install org.apache.commons:commons-collections:3.1.0
~~~~

This downloads the artifact from the configured repositories and adds it to the `compile` dependency group in your project file.

To add a dependency to a different group, pass the group name as a second argument:

~~~~ shell
$ latte install org.testng:testng:6.8.7 test-compile
~~~~

~~~~ shell
$ latte install javax.servlet:servlet-api:4.0.0 provided
~~~~

## Manual dependency definition

You can also add dependencies directly in your project file. Here's an example:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  dependencies {
    group(name: "compile") {
      dependency(id: "org.apache.commons:commons-collections:3.1.0")
    }
  }
}
~~~~ 

This defines a single compile-time dependency on the Commons Collections library version 3.1.0.


## Dependency groups

Dependency groups are completely free-form. You can name them whatever you want, but Latte and some of its plugins might use specific dependency groups by default. Here are the default groups:

* provided - Used during compilation but provided by the container or environment at runtime
* compile - Used to compile the project
* compile-optional - Used to compile the project but are not included at runtime because they are optional
* runtime - Used to execute the project
* test-compile - Used to compile the project's tests
* test-runtime - Used to run the project's tests

### Exporting

You can select which dependency groups are exported when a project is released. Non-exported dependency groups will not be known to other projects and are effectively hidden. This is a nice way to keep dependency graphs clean. You can define a dependency group as being non-exported like this:

~~~~ groovy
dependencies {
  group(name: "test-compile", export: false) {
    dependency(id: "org.testng:testng:6.8.7")
  }
}
~~~~ 


## Artifact IDs

Latte uses a shorthand notation for declaring a project's dependencies (or plugins or any other artifact). The form of this notation is:

~~~~ 
<group>:<project>:<version>
~~~~ 

This format implies that the artifact is a JAR file that ends with **.jar**.

Since Latte does not restrict projects from publishing multiple artifacts or the type of the artifact, there are a couple of extended formats as well:

~~~~ 
<group>:<project>:<artifact>:<version>
<group>:<project>:<artifact>:<version>:<type>
~~~~ 

An example of two dependency declarations from the same project might be something like:

* **com.mycompany:someproject:first-project-artifact:4.1.2:bin**
* **com.mycompany:someproject:second-project-artifact:4.1.2:jar**

These definitions might map to these URLs in a Latte repository:

~~~~ 
http://myLatteRepository.mycompany.com/com/mycompany/someproject/4.1.2/first-project-artifact-4.1.2.bin
http://myLatteRepository.mycompany.com/com/mycompany/someproject/4.1.2/second-project-artifact-4.1.2.jar
~~~~ 


## Workflow processes

Latte needs to know where to download dependencies from. This is done using a workflow definition in your project file:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
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
}
~~~~ 

This tells Latte to first check the local cache for dependencies. If they aren't found there, download them from either the Latte repository or the Maven repository (Maven Central) and cache them locally (that's what the `publish` section is for).

You can simplify this with the `standard()` shorthand, which includes all three of these by default:

~~~~ groovy
workflow {
  standard()
}
~~~~ 

The local cache for Latte is stored at **~/.cache/latte** (or `$XDG_CACHE_HOME/latte` if `XDG_CACHE_HOME` is set).

There are 4 main workflow processes you can use from Latte. Each process has different parameters you can specify to control its behavior:

* cache - Fetches and stores dependencies in the local cache
  * directory - (Optional) If this is blank, Latte will use the default location of **~/.cache/latte**
* url - Fetches artifacts from a Latte repository at an HTTP/HTTPS URL
  * url - (Required) The URL to fetch from
  * username - (Optional) The username to use when downloading from the URL. This is used with HTTP Basic Authentication
  * password - (Optional) The password to use when downloading from the URL. This is used with HTTP Basic Authentication
* maven - Fetches artifacts from a Maven repository at an HTTP/HTTPS URL
  * url - (Required) The URL to fetch from
  * username - (Optional) The username to use when downloading from the URL. This is used with HTTP Basic Authentication
  * password - (Optional) The password to use when downloading from the URL. This is used with HTTP Basic Authentication
* s3 - Fetches and publishes artifacts using any S3-compatible object storage (AWS S3, CloudFlare R2, MinIO, etc.)
  * endpoint - (Required) The S3-compatible endpoint URL (e.g. `https://account-id.r2.cloudflarestorage.com`)
  * bucket - (Required) The bucket name
  * accessKeyId - (Required) The S3 access key ID
  * secretAccessKey - (Required) The S3 secret access key
  * region - (Optional) The AWS region. Defaults to `auto` for CloudFlare R2 compatibility

Here is an example of using the S3 process:

~~~~ groovy
workflow {
  fetch {
    cache()
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: "AKTEST", secretAccessKey: "secret123")
  }
  publish {
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: "AKTEST", secretAccessKey: "secret123", region: "us-east-1")
  }
}
~~~~

## Version compatibility

Latte uses Semantic Versioning to manage version compatibility for individual artifacts. Here is an example:

* Your project depends on library Foo version **1.0**
* Your project also depends on library Bar version **2.0**
* Library Bar depends on library Foo version **1.3**

In this case, Latte needs to determine which version of Foo to include in the classpath at runtime. It can pick either **1.0** or **1.3**. SemVer states that **1.0** is runtime compatible with **1.3**, therefore, Latte will choose to include version **1.3** in the classpath.

### Incompatible versions

Let's take the case above but change one thing. Instead of depending on version **1.3**, library Bar depends on version **3.2**. In this case, SemVer states that these versions are not runtime compatible. In this case, Latte will fail the build and spit out an error message like this:

~~~~ 
The artifact [org.example:Foo] has incompatible versions in your dependencies. The versions are [1.0, 3.2]
~~~~ 

In this case, Latte is attempting to prevent your application from failing at runtime due to an incompatible library.

### Skip compatibility check

There are plenty of libraries in the world today that don't use SemVer or don't version correctly. Likewise, there are plenty of libraries that correctly use SemVer, but make incompatible changes freely and regularly. In some cases, you don't really have a choice but to use the latest version of a dependency, even though it might be incompatible. And in some cases you know that the versions are compatible, just have bad versions. Latte solves this problem by allowing your project to define a dependency and skip the compatibility check on that dependency. Here is how you accomplish this:

~~~~ groovy
dependencies {
  group(name: "runtime") {
    dependency(id: "org.example:Foo:3.2", skipCompatibilityCheck: true)
  }
}
~~~~ 

By setting the **skipCompatibilityCheck** flag, Latte will not fail the build when the versions of an artifact are incompatible.

This flag is not transitive. This means that projects that depend on your project will need to specify the flag on the dependency as well. This forces each project to make a informed decision about which version of the library they want to use.
