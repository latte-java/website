---
layout: docs
title: Database Plugin
description: The database plugin allows you to create databases for testing your software.
weight: 55
plugin: true
---

## Overview

The Latte Database Plugin provides the ability to work with databases.

**LATEST VERSION: 0.1.5**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
database = loadPlugin(id: "org.lattejava.plugin:database:0.1.5")
~~~~ 

## Configuring the plugin

The Database Plugin has a number of configuration options that are stored in the `DatabaseSettings` object instance inside the plugin. There are a couple of settings that are required. Other settings have defaults that should work for you, but you can change anything you want.

### Database type

You must specify the database type to use. This is defined using the **type** field on the settings class like this:

~~~~ groovy
database.settings.type = "mysql"
~~~~ 

Currently, this plugin supports `mysql` and `postgresql` for the type. You can also change the type and re-execute the plugin methods if necessary. This allows you to set up multiple databases with different types.

### Database name

You can change the name of the database that the plugin uses. This is specified by the `name` field on the settings class like this:

~~~~ groovy
database.settings.name = "foobar"
~~~~ 

This will cause the plugin to use the `foobar` database. The default for the name of the database is constructed using the project name. Dashes and dots in the project name are replaced with underscores. For example, if your project name is `my-project` the database name will be `my_project`, and if it is `com.example.app` it will be `com_example_app`.

### Host

The host where the database is running. This defaults to `127.0.0.1`:

~~~~ groovy
db.settings.host = "db.example.com"
~~~~

### Create arguments

You can supply additional command-line arguments to the `mysql` or `psql` programs when the database is created. These are specified like this:

~~~~ groovy
database.settings.createArguments = "-f"
~~~~ 

This will pass the -f argument to the command-line when creating the database.

### Create suffix

A string appended to the database `create` statement:

~~~~ groovy
db.settings.createSuffix = "ENCODING 'UTF8'"
~~~~

### Create username

You can change the user used to create the database. This is configured like this:

~~~~ groovy
database.settings.createUsername = "superuser"
~~~~ 

The default value for `createUsername` depends on the database type you specified. For `mysql` it is `root` and for `postgresql` it is `postgres`.

### Execute username and password

Sets the username and password used when executing SQL scripts. These default to `dev` and `dev`:

~~~~ groovy
db.settings.executeUsername = "app_user"
db.settings.executePassword = "app_password"
~~~~

### Execute arguments

Additional arguments passed when executing SQL scripts:

~~~~ groovy
db.settings.executeArguments = "--set ON_ERROR_STOP=1"
~~~~

### Compare username and password

When comparing two databases (described below), you can change the user and password that are used to connect to the database using the `compareUsername` and `comparePassword` fields on the settings object. Here is an example:

~~~~ groovy
database.settings.compareUsername = "user1"
database.settings.comparePassword = "secret"
~~~~ 

### Grant user

After the database is created, the plugin also grants permissions to the new database to a single user. The username and password of the grant are controlled by the `grantUsername` and `grantPassword` fields on the settings object. Here is an example:

~~~~ groovy
database.settings.grantUsername = "dbuser"
database.settings.grantPassword = "supersecret"
~~~~ 


# Creating the database

The `createDatabase`, `createMainDatabase` and `createTestDatabase` methods on the plugin allow you to create the project databases.

The `createDatabase` method creates a database using the `name` setting.

The `createTestDatabase` creates a database using the project name appended with `_test` as the database name by first updating the settings object with this calculated name and then calling `createDatabase()`. For example, if your project is named `foo-bar` this will create a database named `foo_bar_test`.

The `createMainDatabase` creates a database using the project name as the database name by first updating the settings object with this calculated name and then calling `createDatabase()`. For example, if your project is named `foo-bar` this will create a database named `foo_bar`.

Here is how you call these methods:

~~~~ groovy
database.settings.type = "mysql"
database.settings.name = "foo-bar"
database.createDatabase()

database.createMainDatabase()

database.createTestDatabase()
~~~~

## Executing scripts

The `execute` method allows you to execute arbitrary SQL scripts on a database. This uses the command-line tools rather than JDBC, which means that handling stored procedures and complex SQL is much simpler. Here is how you call this method:

~~~~ groovy
database.settings.name = "foo-bar"
database.settings.type = "mysql"
database.execute(file: "foo.sql")
~~~~ 

This will use the user `dev` to execute the script since that is the default for the `executeUsername` setting. For MySQL, it will also use the `executePassword` setting to execute the script.

However, PostgreSQL doesn't allow passwords to be passed in via the command-line. Therefore, you must set up a `~/.pgpass` file in your home directory to configure the password to use. This file should look something like this:

~~~~ 
localhost:5432:*:postgres:password
localhost:5432:*:dev:dev
~~~~ 

## Comparing databases

Often you need to verify that your database migrations work correctly. To achieve this, you can compare two different databases to ensure that they are identical. The Database Plugin uses Liquibase to compare databases. You can use the `compare` or `ensureEqual` methods to compare databases.

The `compare` method returns a `DiffResult` object that contains the results of the comparison. The `ensureEqual` method fails the build if the databases aren't equal.

Here is how you call these methods:

~~~~ groovy
database.settings.type = "mysql"
database.settings.compareUsername = "dev"
database.settings.comparePassword = "dev"
def result = database.compare(left: "database1", right: "database2")
result.areEqual()
result.getReferenceSnapshot().getDatabase().close()
result.getComparisonSnapshot().getDatabase().close()
~~~~ 

~~~~ groovy
database.settings.type = "mysql"
database.settings.compareUsername = "dev"
database.settings.comparePassword = "dev"
database.ensureEqual(left: "database1", right: "database2")
~~~~
