# DBAL - Schema Introspection
Spiral Database layer provides the ability to read and analyze basic properties of a given database or a given table.
DBAL layer include set of "abstract" types assigned to each column based on DBMS specific mapping in order to unify 
different engines.

## List of database tables
To check if database has table use `hasTable`:

```php
if ($database->hasTable('users')) {
    //...
}
```

> Read how to get Database instances [here](/database/entity.md).

Receive all database tables (array of `Spiral\Database\Table`):

```php
foreach ($database->getTables() as $table) {
    dump($table->getName());
}
```

> Only tables specific to database prefix (if any) are resulted.

Schema Reader/Builder (`AbstractTable`) is available using `getSchema` method:

```php
foreach ($database->getTables() as $table) {
    dump($table->getSchema());
}
```

## Reading table properties using AbstractTable
The AbstractTable provides low level access to table information such as column types (internal and abstract), indexes,
foreign keys and etc. You can use this information to perform database export, build your own ORM or migration mechanism 
(see [schema declaration](/database/declaration.md)).

Table primary keys:

```php
dump($schema->getPrimaryKeys());
```

Table indexes:

```php
foreach ($schema->getIndexes() as $index) {
    dump($index->getName());
    dump($index->getColumns());
    dump($index->isUnique());
}
```

Table foreign keys (references):

```php
foreach ($schema->getForeignKeys() as $foreign) {
    dump($foreign->getColumns());      // local columns name
    dump($foreign->getForeignTable()); // global table name!
    dump($foreign->getForeignKeys());

    dump($foreign->getDeleteRule());   // NO ACTION, CASCADE
    dump($foreign->getUpdateRule());   // NO ACTION, CASCADE
}
```

> Attention, `getForeignTable` returns full table name ignoring db prefix.

Table columns:

```php
foreach ($schema->getColumns() as $column) {
    dump($column->getName());

    dump($column->getInternalType());  // Internal database type
    dump($column->getAbstractType());  // Abstract type like string, bigInt, enum, text and etc.
    dump($column->getType());          // PHP type: int, float, string, bool

    dump($column->hasDefaultValue()); 
    dump($column->getDefaultValue());  // Can be instance of Fragment
    
    dump($column->getSize());          // Only for strings and decimal values

    dump($column->getPrecision());     // Decimals only
    dump($column->getScale());         // Decimals only

    dump($column->isNullable());
    dump($column->getEnumValues());    // Only for enums

    dump($column->getConstraints());

    dump($column->sqlStatement());     // Column creation syntax
}
```

> Some types can be mapped incorrectly if the table was created outside migrations or ORM.

You can find a complete list of available abstract types [here](/database/declaration.md).

## Console Commands
You can also use console commands to get information about configured tables and their schemas:

Command         | Description 
---             | ---
db:list         | Get list of databases, their tables and records count.
db:table        | View table schema of default or specific database.

```
> ./spiral.cli db:table people --database=postgres
Columns of postgres.people:
+---------+-------------------------+----------------+-----------+------------------------------------+
| Column: | Database Type:          | Abstract Type: | PHP Type: | Default Value:                     |
+---------+-------------------------+----------------+-----------+------------------------------------+
| id      | bigserial               | bigPrimary     | int       | nextval('people_id_seq'::regclass) |
| name    | character varying (255) | string         | string    | ---                                |
| income  | numeric (20,2)          | decimal        | float     | ---                                |
| cityID  | bigint                  | bigInteger     | int       | ---                                |
+---------+-------------------------+----------------+-----------+------------------------------------+

Indexes of postgres.people:
+-----------------------------------+-------+----------+
| Name:                             | Type: | Columns: |
+-----------------------------------+-------+----------+
| people_index_income_54ea144908c7c | INDEX | income   |
+-----------------------------------+-------+----------+

Foreign keys of postgres.people:
+-------------------------------------+---------+----------------+-----------------+------------+------------+
| Name:                               | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+-------------------------------------+---------+----------------+-----------------+------------+------------+
| people_foreign_cityID_550ef169f1818 | cityID  | cities         | id              | CASCADE    | NO ACTION  |
+-------------------------------------+---------+----------------+-----------------+------------+------------+
```
