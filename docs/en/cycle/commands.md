# Cycle ORM - Console Commands
Cycle ORM integration provides multiple commands for easier control. You can get help for any of the commands using

```bash
php app.php help cycle...
```

> Make sure to enable `Spiral\Bootloader\CommandBootloader` after the cycle bootloaders to active helper commands.

## Cycle Commands
To update the ORM schema without modifying the database run:

```bash
php app.php cycle
```

To update the schema and automatically modify database schema:

```bash
php app.php cycle:sync
```

> Attention, do not use this command in combination with migrations.

To generate a migration file to reflect the current ORM schema:

```bash
php app.php cycle:migrate
```

> Make sure to run `migrate:init` first.

You can also run generated migration automatically:

```bash
php app.php cycle:migrate -r
```

> You can run any cycle command with `-vv` flag to see a list of modified tables.
