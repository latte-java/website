---
layout: docs
title: Failing the Build
description: Sometimes you need to fail a build. The "fail" method does just that.
weight: 28
---

## Overview

If you need to fail a build for any reason, you simply call the fail method from your project file (or plugin) like this:

~~~~ groovy
fail("Something wicked this way comes")
~~~~
