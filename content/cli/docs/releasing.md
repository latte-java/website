---
layout: docs
title: Releasing
description: Releasing projects is simple with the Git Release plugin.
weight: 90
---

## Overview

Releasing a project involves a number of steps, depending on the type of project and how the project is version controlled. For this example, we will assume that the project is a Java project and managed in Git. For this, we will use the [Release Git Plugin](/cli/docs/plugin-release-git/).

The steps required to release a Java/Git project are:

1. Check for dependencies on integration builds of other projects, libraries, etc.
2. Check for plugins that are integration builds
3. Ensure the project is a Git project
4. Perform a **git pull**
5. Ensure the project has no local changes
6. Ensure the project changes have been pushed to the remote
7. Ensure there isn't a tag in the Git repository for the version being released
8. Creates a tag whose name is the version being released (i.e. 1.0.8)
9. Publishes the project's artifacts (publications) using the publishWorkflow of the project

Luckily, the [Release Git Plugin](/cli/docs/plugin-release-git/) does most of the work. All you need to do is add a few things to your project file and you will be able to release projects quickly and easily.

## Publish workflow

First, add the publish workflow. This is the process that Latte uses to publish the project's artifacts. Here's a sample publish workflow:

~~~~ groovy
project(...) {
  publishWorkflow {
    s3(endpoint: "https://account-id.r2.cloudflarestorage.com", bucket: "my-repo",
       accessKeyId: global.repoAccessKey, secretAccessKey: global.repoSecretKey)
  }
}
~~~~ 

This workflow instructs Latte to publish the project's artifacts to the S3-compatible bucket. This works with any S3-compatible storage provider such as AWS S3, CloudFlare R2, MinIO, and others.

## Release Git plugin

Next, include the [Release Git Plugin](/cli/docs/plugin-release-git/) in the project file:

~~~~ groovy
release = loadPlugin(id: "org.lattejava.plugin:release-git:0.1.0")
~~~~ 

## Target

Finally, add a target to the project file that will run the release:

~~~~ groovy
target(name: "release", description: "Releases a full version of the project", dependsOn: ["test"]) {
  release.release()
}
~~~~ 

That's it! Your project should now be setup to be released using the process described above.
