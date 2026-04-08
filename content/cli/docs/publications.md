---
layout: docs
title: Publications
description: Publications are the artifacts produced by your project that can be used by other projects.
weight: 27
---

## Overview

Publications are the artifacts produced by your project that can be used by other projects. These are defined in the project file inside the `project` definition.

Any project can have zero or more publications. This means that a single project might produce multiple artifacts that can be used by other projects.

Publications are split into two groups **main** and **test**. Main publications are generally used at runtime and test publications are generally used at test time. Each publication is specified by a **name**, **type** and **file**. Publications can also optionally have a source file specified (i.e. a source JAR). Here's how you define publications:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
  
  publications {
    main {
      publication(name: "my-project", type: "tar.gz", file: "build/my-project.tar.gz")
      publication(name: "my-project", type: "jar", file: "build/jars/my-project-${project.version}.jar", source: "build/jars/my-project-${project.version}-src.jar")
    }
    test {
      publication(name: "my-project-test", type: "tar.gz", file: "build/my-project-test.tar.gz")
      publication(name: "my-project-test", type: "jar", file: "build/jars/my-project-test-${project.version}.jar", source: "build/jars/my-project-test-${project.version}-src.jar")
    }
  }
}
~~~~

To reduce the amount of repeated code, Latte provides a shortcut for Java projects that adds a test and main publication for the projects JAR files. Here is the shorthand for that:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
  
  publications {
    standard()
  }
}
~~~~

You can still add additional publications to the **standard** definition above like this:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
  
  publications {
    standard()
    main {
      publication(name: "my-project", type: "tar.gz", file: "build/my-project.tar.gz")
    }
    test {
      publication(name: "my-project-test", type: "tar.gz", file: "build/my-project-test.tar.gz")
    }
  }
}
~~~~

## Publish workflow

When projects are released, the **publishWorkflow** is used to transfer the project's publications to a location that other project's can find them. This is different than an **integration** build for a project. Integration builds use the main **workflow** of the project and not the **publishWorkflow**.
  
Here's how you define a **publishWorkflow** using S3-compatible storage:

~~~~ groovy
project(group: "org.example", name: "my-project", version: "1.0", licenses: ["Apache-2.0"]) {
  workflow {
    standard()
  }
  
  publishWorkflow {
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: "AKTEST", secretAccessKey: "secret123")
  }
  
  publications {
    standard()
  }
}
~~~~

This **publishWorkflow** will upload the project's publications to the S3-compatible bucket at the given endpoint. This works with any S3-compatible storage including AWS S3, CloudFlare R2, MinIO, and others.
