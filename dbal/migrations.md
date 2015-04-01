# Migrations
Spiral DBAL Migratios is simple and quick way to managade database schemas and schema udpdates. While main job of schema migration performed by database driver schemas, migrations are responsible to remember state of database and perform high level set of operations.

Please read about schema building [here] (builder.md).

> Every migration divided by two parts which represented by methods `up` and `down`.

## Create Migration
Creating migration is very simple process which requires developer to place class implements `Spiral\Components\DBAL\Migrations\Migration` into application migration folder (specified in dbal config and usually points to `application/migrations`). Filename used to store such implementation should clearly determinate migration path.

Filename should not follow default naming coversions as it will be loaded using migrator and not composer.

Simple migration can look like:

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

While you still can create and move migrations manually you can try much more convinient CLI command designed for that purposes.

```
./spiral.cli create:migration my_migration
Migration successfully created: 20150401_130123_my_migration.php
```

Such command will create empty migration class into migrations directory. Migration filename will include current date and time, so migrations will always be in correct order.

File content will looks like:

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

Migration creator contains set of pre-defined patterns used in migrations.

### Table Creation
One of the most often operations you can met in migrations is table creation. If you want to include table(s) creation pattern into generated class, use `--create` option (do not include table prefix).

`./spiral.cli create:migration my_migration --create="my_table"`

Generated file will look like:

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

> You have no limits how many tables you want to create in one migration, `--create` option can be used multiple times.

### Table Altering
When your migration designed to only modify existed table(s) use `--alter` prefix.

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

> You have no limits how many tables you want to alter in one migration, `--alter` option can be used multiple times.

### Migration Comment
If you want to add unique comment to your migration class try to use `--comment` option.

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
Spiral DBAL module provides internal API to register migration using it's class implmenentation. This functionality can be used by modules and some specific applications.



> Spiral Module installer provides simplfilied method to reister migration while intalling modules (see [Modules] (../modules.md)).