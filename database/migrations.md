# Migrations
Spiral Migrations mechanism based on DBAL [schema writers] (syncing.md), this specific section only covers migration creation and flow, check previous sections to read about possible schema manipulations.

> Technically migrations are simple set of classes which can be executed and rolled back, spiral ORM can handle most of DB updates, hovewer migrations might help in cases where schema updates can not be handled by ORM (for example when column is removed).

## Migrataion Creation
The easiest way to create migration in spiral is to use console command 'create:migration'. You are able to pre-generate migration code by specifying if you need to create table using key `-c` or alter existed table using key `-a`. Our example command is 'create:migration create_sample_table -c sample'. You can locate generated migration in your migration directory (usually 'application/migrations'), it's code will look like:

```php
class CreateSampleTableMigration extends Migration 
{
    /**
     * Executing migration.
     */
    public function up()
    {
        //Create table 'sample'
        $this->create('sample', function(AbstractTable $schema) {
           $schema->column('id')->primary();
        });
    }

    /**
     * Dropping (rollback) migration.
     */
    public function down()
    {
        //Drop table 'sample'
        $this->schema('sample')->drop();
    }
}
```

Mogration class will declate two basic methods for you `schema(table, database)` and `table(table, database)` to receive instances of `AbstractTable` (schema) and `Table` accordingly.

> Every migration resolved using Container, you can freely declare needed dependencies in class constuctor.

## Run/Rollback Migration
Before you will run your first migration check configuration located in 'application/config/migrations.php':

```php
return [
    'directory'    => directory('application') . '/migrations',
    'database'     => 'default',
    'table'        => 'migrations',
    'environments' => ['development', 'testing', 'staging']
];
```

Such configuration file delares location to store migration clases, database and table used to store information about executed migrations (default implementation uses database to store such information, however you can implement your own Migrator class and store data in other place). In addition to that file contain list of enviroments where you can run migrations without any confirmation.

First of all you have to initate migration table in selected database, to do that command 'migrate:init' must be executed.
To get list of exucuted and outstanding migrations use command 'migrate:status', it's output must look like:

```
+---------------------+-------------------------------------------+---------------------+---------------------+
| Migration:          | Filename:                                 | Created at          | Performed at        |
+---------------------+-------------------------------------------+---------------------+---------------------+
| my_migration        | 20150913_195655_0_my_migration.php        | 2015-09-13 19:56:55 | 2015-09-13 20:36:19 |
| test                | 20150913_202400_1_test.php                | 2015-09-13 20:24:00 | not executed yet    |
| create_sample_table | 20150913_202907_2_create_sample_table.php | 2015-09-13 20:29:07 | not executed yet    |
+---------------------+-------------------------------------------+---------------------+---------------------+
```

To run, rollback or replace migrations check commands 'migrate', 'migrate:rollback' and 'migrate:replay' accordingly. 

## Registering Migrations
In some cases (usually for modules) you might pre-create migration classes. To add such migration into queue we can use migrator method `registerMigration`, component will copy class source into migrations directory and execute it when appropriate console command will be runned. Migration component can be retrieved using dependency `MigratorInterface` in your code.

```php
public function index(MigratorInterface $migrator)
{
    $migrator->registerMigration('my_migration', \MyMigration::class);
}
```

You can also get list of currently available migrations and their statuses using method `getMigrations`.

```php
public function index(MigratorInterface $migrator)
{
    foreach ($migrator->getMigrations() as $migration) {
        //Status is an object with name, status, creation and execution time,
        //not string
        dump($migration->getState());
    }
}
```
