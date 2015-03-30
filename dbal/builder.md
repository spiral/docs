# Schema Builder
Additionally to schema reading, TableSchema instances provide database agnostic way to write and update schema, such abilities includes creating enum fields, define foreign keys, indexes and etc.

Schema building mainly used in dbal migrations.

> Make sure you read about [Schema Reader] (reader.md) previously to understand about read abilities of `TableSchema`.

## Schema creation flow
To efficenty use schema builder let's walk thought process used to create schema in database.

First step in any creation process is receiveing instance of TableSchema class, every driver provides it's own implementation of schema builder so it has to be requested using associated database.
Provided example assumed that `$database` is instance of `dbal\Database`.

```php
$schema = $database->table('table')->schema();
```

Inside migrations you can use simplified syntax:

```php
public funcntion up()
{
	$schema = $this->schema('table'); //Table name withou prefix
}

```

Secondly you have to specify table schema properties such as columns, indexes and foreign keys and operations (column renames, removals and etc). This side of schema generation covered in next section of documentation, but let's check simple example:

```php
$schema = $database->table('table')->schema();

$schema->primary('id');
$schema->string('email');
```

And the final step is applying schema to database which will either
create or update existed table in database with added/removed columns and etc.

```php
$schema = $database->table('table')->schema();

$schema->primary('id');
$schema->string('email');

$schema->save();
```

> While calling `save()` TableSchema will compare declared and already existed schemas to make sure that update is reasonable. If nothing were altered - no queries will be sent to database.

## Declaring table schema

### Columns


### Primary keys

### Indexes

### Foreign keys

## High level table operations
 