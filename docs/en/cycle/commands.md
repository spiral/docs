# Cycle ORM - Console Commands

Cycle ORM integration provides multiple commands for easier control. You can get help for any of the commands using

```bash
php app.php help cycle...
```

> **Note**
> Make sure to enable `Spiral\Cycle\Bootloader\CommandBootloader` after the cycle bootloaders to active helper commands.

### Migrations

| Command            | Description                                                                                  |
|--------------------|----------------------------------------------------------------------------------------------|
| `migrate`          | Perform one or all outstanding migrations.<br/>`--one` Execute only one (first) migration.   |
| `migrate:replay`   | Replay (down, up) one or multiple migrations.<br/>`--all` Replay all migrations.             |
| `migrate:rollback` | Rollback one (default) or multiple migrations.<br/>`--all` Rollback all executed migrations. |
| `migrate:init`     | Init migrations component (create migrations table).                                         |
| `migrate:status`   | Get list of all available migrations and their statuses.                                     |

### Database

| Command            | Description                                                                                                      |
|--------------------|------------------------------------------------------------------------------------------------------------------|
| `db:list [db]`     | Get list of available databases, their tables and records count.<br/>`db` database name.                         |
| `db:table <table>` | Describe table schema of specific database.<br/>`table` Table name (required).<br/>`--database` Source database. |

### ORM and Schema

| Command         | Description                                                                          |
|-----------------|--------------------------------------------------------------------------------------|
| `cycle`         | Update (init) cycle schema from database and annotated classes.                      |
| `cycle:migrate` | Generate ORM schema migrations.<br/>`--run` Automatically run generated migration.   |
| `cycle:render`  | Render available CycleORM schemas.<br/>`--no-color` Display output without colors.   |

> **Note**
> You can run any cycle command with `-vv` flag to see a list of modified tables.
