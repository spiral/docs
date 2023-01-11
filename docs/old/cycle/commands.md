# Cycle ORM - Console Commands

The Cycle ORM integration provides multiple commands for easier control. You can get help for any of the commands using

```bash
php app.php help cycle...
```

> **Note**
> Make sure to enable `Spiral\Cycle\Bootloader\CommandBootloader` after the cycle bootloaders to activate helper commands.

### Migrations

| Command            | Description                                                                                  |
|--------------------|----------------------------------------------------------------------------------------------|
| `migrate`          | Performs one or all the outstanding migrations.<br/>`--one` Execute only one (first) migration.   |
| `migrate:replay`   | Replays (down, up) one or multiple migrations.<br/>`--all` Replays all the migrations.             |
| `migrate:rollback` | Rolls back  (default) or multiple migrations.<br/>`--all` Rolls back all the executed migrations. |
| `migrate:init`     | Initiates the migrations component (create a migrations table).                                         |
| `migrate:status`   | Gets a list of all available migrations and their statuses.                                     |

### Database

| Command            | Description                                                                                                      |
|--------------------|------------------------------------------------------------------------------------------------------------------|
| `db:list [db]`     | Gets a list of available databases, their tables and records count.<br/>`db` database name.                         |
| `db:table <table>` | Describes a table schema of a specific database.<br/>`table` A table name (required).<br/>`--database` A source database. 

### ORM and Schema

| Command         | Description                                                                          |
|-----------------|--------------------------------------------------------------------------------------|
| `cycle`         | Updates (init) the cycle schema from the database and annotated classes.                      |
| `cycle:migrate` | Generates the ORM schema migrations.<br/>`--run` Automatically runs a generated migration.   |
| `cycle:render`  | Renders the available CycleORM schemas.<br/>`--no-color` Displays output without colors.   |

> **Note**
> You can run any cycle command with the `-vv` flag to see a list of modified tables.
