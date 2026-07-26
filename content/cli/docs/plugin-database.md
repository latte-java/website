---
layout: docs
title: Database Plugin
description: The database plugin allows you to create databases, run migrations and compare schemas.
weight: 55
plugin: true
---

## Overview

The Latte Database Plugin provides the ability to work with databases. It can create databases, execute SQL scripts, apply SQL migrations and compare two schemas.

**LATEST VERSION: 0.5.0**

## Loading the plugin

Here is how you load this plugin:

~~~~ groovy
database = loadPlugin(id: "org.lattejava.plugin:database:0.5.0")
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

This will cause the plugin to use the `foobar` database. The default for the name of the database is constructed from the project name with dashes replaced by underscores. For example, if your project name is `my-project` the database name will be `my_project`. Dots in the project name are **not** replaced by the default, so a project named `com.example.app` gets a default `settings.name` of `com.example.app` — this is accepted by MySQL and PostgreSQL, but if you'd rather have underscores, set `settings.name` explicitly or use `createMainDatabase()` / `createTestDatabase()` (see below), both of which replace `.` with `_`.

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

Sets the username and password used when executing SQL scripts. These default to `dev` and `dev`. The same credentials are used to connect when running migrations:

~~~~ groovy
db.settings.executeUsername = "app_user"
db.settings.executePassword = "app_password"
~~~~

### Execute arguments

Additional arguments passed when executing SQL scripts:

~~~~ groovy
db.settings.executeArguments = "--set ON_ERROR_STOP=1"
~~~~

### Migrations directory

The directory that contains the SQL migration scripts, resolved against the project directory. This defaults to `src/main/resources/db`:

~~~~ groovy
db.settings.migrationsDirectory = "db/migrations"
~~~~

The default puts the scripts in the project's resources, which means the same files are packaged into the JAR and your application can apply them from the classpath at startup. See [Running migrations](#running-migrations) below.

### Migration table

The name of the table that migrations record applied versions in. This defaults to `versions`:

~~~~ groovy
db.settings.migrationTable = "schema_versions"
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


## Creating the database

The `createDatabase`, `createMainDatabase` and `createTestDatabase` methods on the plugin allow you to create the project databases.

The `createDatabase` method creates a database using the `name` setting.

The `createTestDatabase` creates a database using the project name appended with `_test` as the database name by first updating the settings object with this calculated name and then calling `createDatabase()`. Both dashes and dots in the project name are replaced with underscores. For example, if your project is named `foo-bar` this will create a database named `foo_bar_test`, and if it is named `com.example.app` it will create `com_example_app_test`.

The `createMainDatabase` creates a database using the project name as the database name by first updating the settings object with this calculated name and then calling `createDatabase()`. Both dashes and dots in the project name are replaced with underscores. For example, if your project is named `foo-bar` this will create a database named `foo_bar`, and if it is named `com.example.app` it will create `com_example_app`.

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

## Running migrations

The `migrate` method applies the SQL migration scripts in the [migrations directory](#migrations-directory) to the database named by the `name` setting. Migrations are files named `<semver>.sql` (for example `1.0.0.sql`) that are applied in SemVer order by the [Latte database library](/database/). Each applied version is recorded — along with the script's SHA-256 checksum — in the [migration table](#migration-table), so running `migrate` again only applies the scripts that are new.

Here is how you call this method:

~~~~ groovy
database.settings.type = "postgresql"
database.settings.name = "myapp"
database.migrate()
~~~~

Unlike `execute`, this method does not shell out to the `mysql` or `psql` programs. It connects over JDBC using the `host`, `executeUsername` and `executePassword` settings, which means no `~/.pgpass` file is needed for PostgreSQL.

The method returns the versions it applied, in the order they were applied. The list is empty when the database is already up to date:

~~~~ groovy
def applied = database.migrate()
applied.each { println "Applied ${it}" }
~~~~

The database itself must already exist — `migrate` connects to it, it doesn't create it. A build file usually has one target for each job:

~~~~ groovy
target(name: "create-database", description: "Creates the project databases") {
  database.settings.type = "postgresql"
  database.createMainDatabase()
  database.createTestDatabase()
}

target(name: "migrate", description: "Applies the pending database migrations") {
  database.settings.type = "postgresql"
  database.settings.name = "myapp"
  database.migrate()
}
~~~~

The plugin always reads the scripts from a directory. Because that directory is in the project's resources by default, the same scripts are packaged into your JAR and your application can apply them from the classpath when it starts up. See the [database](/database/docs/running-migrations/) documentation for how to do that.

The build fails if the migrations directory doesn't exist, a script isn't named `<semver>.sql`, a pending version is lower than the highest version already applied, an already-applied script has been modified (its checksum no longer matches), or a statement fails — in which case the message names the version, the statement and the line it started on.

## Comparing databases

Often you need to verify that your database migrations work correctly. To achieve this, you can compare two different databases to ensure that they are identical. The Database Plugin uses jOOQ to compare databases. You can use the `compare` or `ensureEqual` methods to compare databases.

The `compare` method returns a `DatabaseComparison` object whose `differences` are the DDL statements that would transform the left database into the right database. Its `areEqual()` method returns true when there are no differences. The `ensureEqual` method runs the same comparison and fails the build if the databases aren't equal, printing the DDL statements that differ.

Here is how you call these methods:

~~~~ groovy
database.settings.type = "mysql"
database.settings.compareUsername = "dev"
database.settings.comparePassword = "dev"
def comparison = database.compare(left: "database1", right: "database2")
if (!comparison.areEqual()) {
  comparison.differences.each { println it }
}
~~~~ 

~~~~ groovy
database.settings.type = "mysql"
database.settings.compareUsername = "dev"
database.settings.comparePassword = "dev"
database.ensureEqual(left: "database1", right: "database2")
~~~~

A common use of this is to verify that a database built by replaying every migration matches one built from a known-good schema:

~~~~ groovy
target(name: "verify-migrations", description: "Ensures the migrations produce the expected schema") {
  database.settings.type = "postgresql"
  database.createTestDatabase()
  database.migrate()
  database.ensureEqual(left: "myapp_test", right: "myapp")
}
~~~~
