# Migrations
Spiral DBAL Migrations is a simple and quick way to managae database schemas and schema udpdates. While main job of schema migration performed by database driver schemas, migrations are responsible for remembering the  state of database and performing high level sets of operations.

Please read about schema building [here] (builder.md).

> Every migration is divided into two parts which are represented by the methods `up` and `down`.

## Create Migration
Creating a migration is a very simple process which requires the developer to place class implementing `Spiral\Components\DBAL\Migrations\Migration` into the application migration folder (specified in dbal config and usually pointed to `application/migrations`). The filename used to store this implementation should clearly determine the migration path.

The filename should not follow default naming coversions as it will be loaded using migrator and not composer.

A simple migration can look like:

```php
use Spiral\Components\DBAL\Migration;

class MyMigration extends Migration 
{
	public function up()
	{
		$table = $this->schema('my_table');
		$table->column('id')->primary();
		$table->column('name')->string();
		
		$table->save();
	}

	public function down()
	{
		$this->schema('my_table')->drop();
	}
}

```

While you can create and move migrations manually, there is a much more convenient CLI command designed for that purposes.

```
./spiral.cli create:migration my_migration
Migration successfully created: 20150401_130123_my_migration.php
```

This command will create an empty migration class in the migrations directory. The migration filename will include the current date and time, so migrations will always be in the correct order.

The file contents will look like:

```php
<?php
/**
 * This file is generated automatically on 2015-04-01T13:02:21+00:00.
 */
use Spiral\Components\DBAL\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Executing migration.
     */
    public function up()
    {
    }

    /**
     * Dropping (rollback) migration.
     */
    public function down()
    {
    }
}
```

The migration creator contains a set of pre-defined patterns used in migrations.

### Table Creation
One of the most common operations you can use in migrations is table creation. If you want to include a table(s) creation pattern into generated class, use `--create` option (do not include table prefix).

`./spiral.cli create:migration my_migration --create="my_table"`

The generated file will look like:

```php
<?php
/**
 * This file is generated automatically on 2015-04-01T13:08:19+00:00.
 */
use Spiral\Components\DBAL\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Executing migration.
     */
    public function up()
    {
        //Creating table "my_table"
        $schema = $this->schema('my_table');
        $schema->column('id')->primary();

        $schema->save();
    }

    /**
     * Dropping (rollback) migration.
     */
    public function down()
    {
        //Dropping table "my_table"
        $this->schema('my_table')->drop();
    }
}
```

> You have no limits on how many tables you can create in one migration, the `--create` option can be used multiple times.

### Table Altering
When your migration is designed to only modify existing table(s) use the `--alter` prefix.

`./spiral.cli create:migration my_migration --alter="my_table"`

Result:

```php
<?php
/**
 * This file is generated automatically on 2015-04-01T13:11:54+00:00.
 */
use Spiral\Components\DBAL\Migrations\Migration;

class MyMigration extends Migration
{
    /**
     * Executing migration.
     */
    public function up()
    {
        //Altering table "my_table"
        $schema = $this->schema('my_table');

        $schema->save();
    }

    /**
     * Dropping (rollback) migration.
     */
    public function down()
    {
        //Rolling back changes in table "my_table"
        $schema = $this->schema('my_table');

        $schema->save();
    }
}
```

> You have no limits how many tables you can alter in one migration, the `--alter` option can be used multiple times.

### Migration Comment
If you want to add an unique comment to your migration class try using the `--comment` option.

`./spiral.cli create:migration my_migration --comment="This is demo migration."`

Output:

```php
<?php
/**
 * This file is generated automatically on 2015-04-01T13:05:32+00:00.
 */
use Spiral\Components\DBAL\Migrations\Migration;

/**
 * This is demo migration.
 */
class MyMigration extends Migration
{
    /**
     * Executing migration.
     */
    public function up()
    {
    }

    /**
     * Dropping (rollback) migration.
     */
    public function down()
    {
    }
}
```

> To get more information about migration creator options execute `./spiral.cli create:migration --help`.

## Configure Migrator

## Migrations status

## Run Migration

## Rollback Migration

## Replay Migration

## Register Migration (for Modules)
The Spiral DBAL module provides an internal API to register migrations using its class implmenentation. This functionality can be used by modules and some specific applications.



> Spiral Module installer provides a simplified method to register migrations while intalling modules (see [Modules] (../modules.md)).
