---
layout: docs
title: Variables
description: There are a number of global variables available in the project file as well as a way to load local variables from properties files.
weight: 22
---

## Overview

Latte provides a couple of variables you can access from the build script. These are:

* `SYS` - System properties (e.g. -Dfoo=bar)
* `ENV` - Environment variables (e.g. export FOO=BAR)
* `switches` - An instance of the **org.lattejava.runtime.Switches** class that provides access to command-line switches (e.g. --foo=bar)
* `runtimeConfiguration` - The main instance of the **org.lattejava.runtime.RuntimeConfiguration** for the current Latte runtime
* `project` - The main instance of the **org.lattejava.domain.Project** class that represents the project
* `output` - The main instance of the **org.lattejava.output.Output** class that allows you to print messages
* `global` - A global configuration object (see below)

Here is an example of using these variables:

~~~~ groovy
output.info("The project version is ${project.version}")
output.error("System property foo ${SYS['foo']}")
output.warning("Environment variable foo ${ENV['foo']}")
output.debug("Has switch foo ${switches.has('foo')}")
output.info("Switch values ${switches.values('foo')}")
output.info("Global username is ${global.username}")
~~~~ 

## Global configuration

Latte allows you to create a properties file in your home directory and use values from it in your build script. This is useful for things like usernames and password. It can also be used for company specific configuration like URLs or IP addresses. Since each developer on a team can have a different configuration file on their local computer, you can have them setup differently as well.

This file must be located here:

~~~~
~/.config/latte/config.properties
~~~~ 

This is a standard Java properties file and might look like this:

~~~~ properties
internalRepositoryUsername=fred
internalRepositoryPassword=FluffyBunny
~~~~ 

You can then use these properties in your project file like this:

~~~~ groovy
workflow {
  fetch {
    url(url: "http://www.mycompany.com/latte-repository", username: global.internalRepositoryUsername, password: global.internalRepositoryPassword)
  }
}
~~~~ 
