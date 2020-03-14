# Console Commands
The framework exposes several commands to control the build and help in development.

To list all available console commands:

```bash
$ php app.php
```

To get help about any specific command:

```bash
$ php app.php help db:table
```

Alternative: 

```bash
$ php app.php db:table -h
```

> You can read how to create your commands in [the following section](/console/commands.md).

## Aliases
Spiral Console based on Symfony/Console, it means you can use short command names as far as there is enough information to find the target command:

```bash
# resolved into `update`
$ php app.php up 

# Collides with `configure`, `cache:clean`, `cycle:*` commands.
$ php app.php c
```

## Configure
Your application includes one central command `configure`. This command will run the sequence of operations to ensure
that the application correctly configured, create needed directions, and verify permissions to the resources.

To run this command in verbose mode:

```bash
$ php app.php configure -vv
```

> Always run `configure` before running a newly installed application.

## Application Server
The application server includes it's own set of commands, to list all available server commands run:

```bash
$ ./spiral
```

You can read more about server commands [here](https://roadrunner.dev/docs/beep-beep-cli).
