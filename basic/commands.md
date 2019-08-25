# Console Commands
The framework exposes the number of commands to control the build and help in development.

To list all available console commands:

```bash
$ php app.php
```

To get help about any specific command:

```bash
$ php app.php help db:table
```

> You can read how to create your own commands in a following sections.

## Configure
Your application includes one main command `configure`. This command will run the sequence of operations to ensure
that application is properly installed, create needed directions and verify permissions to the resources.

To run this command in verbose mode:

```bash
$ php app.php configure -vv
```

> Always run `configure` before running newly installed application.

## RoadRunner
Application server includes it's own set of commands, to list all available server commands run:

```bash
$ ./spiral
```

You can read more about server commands [here](https://roadrunner.dev/docs/usage-server-commands).