# Database - Migrations
Spiral ships with a set of embedded commands to control your database migrations, [component](https://github.com/spiral/migrations) 
is build upon DBAL and support virtual databases and prefixes.

## Configure Migrations (optional)
You can configure what database and table to use to store information about schema version in migrations config. Create
`app/config/migrations.php` to alter default values:

```php
return [
    // directory to store migration files
    'directory' => directory('application') . 'migrations/',

    // Table name to store information about migrations status (per database)
    table'     => 'migrations',
   
    // When set to true no confirmation will be requested on migration run. 
    'safe'      => env('SPIRAL_ENV') == 'develop'
];
```

Migration state table can be automatically initiated using the command `migrate:init`.

## Create a migration
We can create our migrations manually or use scaffolder module for such purposes with a set of helper options:

```bash
$ php app.php create:migration -t sample_table -f id:primary -f name:string my_migration
Declaration of 'MyMigrationMigration' has been successfully written into '20170401.160544_0_my_migration.php'.
```

The migration will be automatically placed into your `app/migrations` directory:

```php
class MyMigrationMigration extends Migration
{
    /**
     * Create tables, add columns or insert data here
     */
    public function up()
    {
        $this->table('sample_table')
            ->addColumn('id', 'primary')
            ->addColumn('name', 'string')
            ->create();
    }

    /**
     * Drop created, columns and etc here
     */
    public function down()
    {
        $this->table('sample_table')->drop();
    }
}
```

> Install [scaffolder module](/cookbook/scaffolding.md) first.

## Working with migrations
You can run all outstanding migrations using `migrate` command.

```bash
$ php app.php migrate
```

You can now view your table using `db:table` command:

```bash
$ php app.php db:table sample_table
Columns of primary.sample_table:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int (11)       | primary        | int       | ---            |
| name    | varchar (255)  | string         | string    | ---            |
+---------+----------------+----------------+-----------+----------------+
```

Use `-vv` flag to get more information about generated queries:

```bash
$ php app.php migrate:replay -vv
Rolling back executed migration(s)...
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
[MySQLDriver] Begin transaction
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'sample_table'
[MySQLDriver] SHOW FULL COLUMNS FROM `sample_table`
[MySQLDriver] SHOW INDEXES FROM `sample_table`
[MySQLDriver] SELECT * FROM `information_schema`.`referential_constraints` WHERE `constraint_schema` = 'sample_2' AND `table_name` = 'sample_table'
[MySQLDriver] SHOW INDEXES FROM `sample_table`
[MySQLDriver] SHOW TABLE STATUS WHERE `Name` = 'sample_table'
[MySQLDriver] DROP TABLE `sample_table`
[MySQLDriver] Commit transaction
[MySQLDriver] DELETE FROM `migrations`
WHERE `migration` = 'my_migration'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
Migration my_migration was successfully rolled back.

Executing outstanding migration(s)...
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
[MySQLDriver] Begin transaction
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'sample_table'
[MySQLDriver] CREATE TABLE `sample_table` (
    `id` int (11) NOT NULL AUTO_INCREMENT,
    `name` varchar (255) NOT NULL,
    PRIMARY KEY (`id`)
) ENGINE InnoDB
[MySQLDriver] Commit transaction
[MySQLDriver] INSERT INTO `migrations` (`migration`, `time_executed`)
VALUES ('my_migration', '2017-04-01 16:12:21')
[MySQLDriver] Given insert ID: 2
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
Migration my_migration was successfully executed.
```

> Enable DEBUG mode first.

Run command `migrate:rollback` and `migrate:replay` to rollback your migrations.

## Update existed schema
Create migrations to alter existed table schema:

```php
class NewFieldMigration extends Migration
{
    public function up()
    {
        $this->table('sample_table')
            ->addColumn('field', 'float')
            ->update();
    }
    
    public function down()
    {
        $this->table('sample_table')
            ->dropColumn('field')
            ->update();
    }
}
```

```
$ php app.php migrate -vv
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'new_field'
[MySQLDriver] Begin transaction
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'sample_table'
[MySQLDriver] SHOW FULL COLUMNS FROM `sample_table`
[MySQLDriver] SHOW INDEXES FROM `sample_table`
[MySQLDriver] SELECT * FROM `information_schema`.`referential_constraints` WHERE `constraint_schema` = 'sample_2' AND `table_name` = 'sample_table'
[MySQLDriver] SHOW INDEXES FROM `sample_table`
[MySQLDriver] SHOW TABLE STATUS WHERE `Name` = 'sample_table'
[MySQLDriver] ALTER TABLE `sample_table` ADD COLUMN `field` float NOT NULL
[MySQLDriver] Commit transaction
[MySQLDriver] INSERT INTO `migrations` (`migration`, `time_executed`)
VALUES ('new_field', '2017-04-01 16:23:36')
[MySQLDriver] Given insert ID: 4
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'new_field'
Migration new_field was successfully executed.
[MySQLDriver] SELECT COUNT(*) FROM `information_schema`.`tables` WHERE `table_schema` = 'sample_2' AND `table_name` = 'migrations'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'my_migration'
[MySQLDriver] SELECT
`id`, `time_executed`
FROM `migrations`
WHERE `migration` = 'new_field'
```

## Compatibility with DBAL
All migration methods are based on DBAL functions, feel free to use same abstract types as in 
[direct schema declarations](/database/declaration.md).

> Note that ORM component can create migrations automatically.
