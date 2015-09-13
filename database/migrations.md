# Migrations
Spiral Migrations mechanism based on DBAL [schema writes] (syncing.md), this specific section only covers migration creation and flow, check previous sections to read about possible schema manipulations.

> Technically migrations are simple set of classes which can be executed and rolled back, spiral ORM can handle most of DB updates, hovewer migrations might help in cases where schema updates can not be handled by ORM (for example when column is removed).

## Migrataion Creation

## Running Migrations


## Registering Migrations
In some cases (usually for modules) you might pre-create migration classes. To add such migration into queue we might use migrator method `registerMigration`, component will copy class source into migrations directory and execute it when appropriate console command will be runned. Migration component can be retrieved using dependency `MigratorInterface` in your code.

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
        dump($migration->getStatus());
    }
}
```