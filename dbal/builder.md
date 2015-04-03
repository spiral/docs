# Schema Builder
In additional to schema reading, TableSchema instances provides a database agnostic way to write and update schema, including creating enum fields, and defining foreign keys, indexes and etc.

Schema building is mainly used in dbal migrations.

> Make sure you read about [Schema Reader] (reader.md) to understand the read abilities of `TableSchema`.

## Schema creation flow
To efficenty use schema builder let's walk through the process used to create schema in database.

The first step in any creation process is to get an instance of the TableSchema class. Every driver provides its own implementation of schema builder, so it has to be requested using the associated database.
The provided example assumed that `$database` is instance of `dbal\Database`.

```php
$schema = $database->table('table')->schema();
```

Inside migrations you can use simplified syntax:

```php
public function up()
{
	$schema = $this->schema('table'); //Table name without prefix
}

```

Secondly you have to specify table schema properties such as columns, indexes and foreign keys and operations (column renames, removals and etc). This type of schema generation is covered in the next section of documentation, but let's check a simple example:

```php
$schema = $database->table('table')->schema();

$schema->primary('id');
$schema->string('email');
```

And the final step is applying schema to database which will either
create or update the existing table in the database with the added/removed columns and etc.

```php
$schema = $database->table('table')->schema();

$schema->primary('id');
$schema->string('email');

$schema->save();
```

> While calling `save()` TableSchema will compare declared and already existing schemas to make sure that update is reasonable. If nothing were altered - no queries will be sent to database.

## Declaring table schema

### Columns


### Primary keys

### Indexes

### Foreign keys

## High level table operations
 
