




## SyncronizationBus

## Removals and Existest Table modifications

## Working with Comparator directly
In some cases you might want to dedicate table operations to external migration mechanism (for example Phinx), in this case you can access internal TableSchema state and it's comparator:

```php
$schema->integer('some_value');
$comparator = $schema->comparator();

dump($comparator->addedColumns());
```

> Comparator will provide you list of created, updated, removed columns, indexes and foreign keys. You can also use your own version of SyncronizationBus to write and run migrations instead of performing altering operations.

If you dont want to deal with external migration mechanism but some data has to be moved, or column to renamed simply utilize introspection part of your schema:

```php
if (!$schema->hasColumn('column')) {
    $schema->renameColumn('other_column', 'column');
    //moving stuff around (don't forget to save schema)
}
```

> Spiral has planned to have additional module which provides ability to generate migration files based on a changed state of database, it will provide developer ability to alter and tweak migration files before executing them.

## Table related operations
You can also apply some operations on table level, such commands does not require schema saving and executed immidiatelly:

```php
//Database prefix will be automatically assigned
$schema->rename('new_name');
```

To remove table simply run:

```php
$schema->drop();
```
