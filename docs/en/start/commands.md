# Console Commands

## Introduction

In the Spiral Framework, console commands allow you to perform various tasks from the command line.
These tasks can range from simple operations like clearing a cache, to more complex ones like running migrations or seeding a database.
Console commands are a useful tool for automating repetitive tasks and can save you time when developing and maintaining your application.

To list all available console commands use the following command:

```bash
php app.php
```

To get help about any specific command:

```bash
php app.php help db:table
```

Alternative:

```bash
php app.php db:table -h
```

> **Note**
> You can read how to create your own commands [here](https://spiral.dev/docs/console-commands/3.3/en).

## Aliases

To use command aliases, simply type the abbreviated version of the command you wish to use instead of the full command name.
For example, if you have a command called "send-email", you can use the alias "se" by typing "se" instead of "send-email".

```bash
# Common variant
php app.php send-email 

# Can be resolved into `send-email`
php app.php se
```

## Verify Application Installation

The `configure` command included in your application performs a series of operations to set up and verify the proper installation of the application.
It creates necessary directories and checks permissions for resources.
To run this command with verbose output, use the `--verbose` flag.

```bash
php app.php configure -vv
```

> **Note**
> Always run `configure` before running a newly installed application.

## Application Server

The application server (RoadRunner) includes its own set of commands. To list all the available server commands, run:

```bash
./rr
```

> **Note**
> You can read more about server commands [here](https://roadrunner.dev/docs/app-server-cli/2.x/en).