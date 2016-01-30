## Schema Declaration  - OUTDATED DOCUMENTATION 
One of the most important part of Spiral DBAL is ability to alter database table schema using set of entity abstractions. Such abstractions provide ability to describe desired table rather than create it operation by operation.

> Attention, declarative schemas are useful in RAD development but can't cover all possible scenarios. Spiral is planned to utilize
internal table schema comparator to generate needed Phinx migrations based on table diff to give developer more control.

Guide TODO: drop command based syntax, write about SyncronizationBus (transaction and table sorter), write about modifying existed tables using pre-declaration.

## Principle of Work
Before any operation/declaration can be applied to table schema, DBAL will load currently existed structure from database and [normalize it into internal format](/database/introspection.md). As result, you are allowed to apply modification to table schema using declarative way instead of imperative, once schema **save** are requested - DBAL will generate set of creation and altering operations based on difference between declared and existed schemas. 
> Unfortunatelly some SQL features got simplified to fit, for example primary table key is described as column type, not index.

You can also use additional schema operations to remove or rename table elements.

> Please remember to execute multiple table syncronization using `SyncronizationBus` class, this implementation will sort your tables in a vaild dependency order and execute every operation under connection specific transaction, see examples below.

## To Start
To get instance of TableSchema which we can manipulate with, we can use similar way described in [Schema Introspection (make sure your read them first)](/database/instrospection.md). The only difference - we don't need to check table existence. 

Let's use controller action to write an example:

```php
protected function indexAction(Database $database)
{
    //Attention, database will add it's prefix to final table name
    $schema = $database->table('new_table')->schema();
    
    //Schema suppose to be empty
    dump($schema);
    dump($schema->exists());
}
```

Once we have our schema instance requested we can start describing it.

## Columns and Abstract Types
You can add columns to specific schema by simply setting their type, spiral DBAL provides fairly simple way of doing that. Let's try to describe few basic columns in our table schema and later jump to details:

```php
$schema = $database->table('new_table')->schema();

$schema->column('id')->primary();
$schema->column('name')->string(64); //String length 64 characters
$schema->column('email')->string();  //Default string length is 255 symbols
$schema->column('balance')->decimal(10, 2);
$schema->column('description')->text();
```

> All of listed methods are added into table and column doc comments so your IDE/editor will highlight them.

As alternative to such definition you can use shorter (magical) version based on your preferences:

```php
$schema = $database->table('new_table')->schema();

$schema->primary('id');
$schema->string('name', 64); //String length 64 characters
$schema->string('email');    //Default string length is 255 symbols
$schema->decimal('balance', 10, 2);
$schema->text('description');
```

The only thing we have to do now to push desired schema to our database is execute `$schema->save()`. Depending on database driver you using DBAL will generate different SQL statements to create table, in our case we are using default MySQL database (with prefix "primary_") so our statement will look like:

```sql
CREATE TABLE `primary_new_table` (
    `id` int (11) NOT NULL AUTO_INCREMENT,
    `name` varchar (64) NULL,
    `email` varchar (255) NULL,
    `balance` decimal (10, 2) NULL,
    `description` text NULL,
    PRIMARY KEY (`id`)
) ENGINE = InnoDB
```

We can always switch our database to different connection (for example Postgres) to generate another statement:

```sql
CREATE TABLE "secondary_new_table" (
    "id" serial NOT NULL,
    "name" character varying (64) NULL,
    "email" character varying (255) NULL,
    "balance" numeric (10, 2) NULL,
    "description" text NULL,
    PRIMARY KEY ("id")
)
```

> By default, Spiral marks every column as nullable, you can easily alter such behaviour (see below), in addition to that - ORM will make every column NOT NULL with non empty default value fetched from model schema or automatically casted by ORM component.

You can try to execute such script again and notice that no table altering sql were generated (check [Profiler] (/modules/profiler.md) logging tab), this happens due database component fetched exsited schema from databases and found that there is no differences between it and declared structure. If you want to add new column to your table or even change type of exsited one you can simply alter your code:

```php
$schema = $database->table('new_table')->schema();

$schema->primary('id');
$schema->string('name', 64); //String length 64 characters
$schema->string('email');    //Default string length is 255 symbols
$schema->decimal('balance', 10, 2);

$schema->longText('description'); //New type
$schema->integer('count_visits'); //New column

$schema->save();
```

Schema will detect that new column beign added and existed column got new type, as result it will generate driver specific set of sql statements:

```sql
ALTER TABLE `primary_new_table` CHANGE `description` `description` longtext NULL;
ALTER TABLE `primary_new_table` ADD COLUMN `count_visits` int (11) NULL;
```

> You can also notice that Postgres does not have statement to change type of "description" column, such thing happend due declared abstract type "longText" does not differs from simple "text" type in Postgres databases.

> Attention, not every type can be easily changed in some databases. Make sure you are not violating DBMS specific cross type conversion (string => integer for example).

### Abstract Types
You can notice, based on provided example, we defined required columns using types named "string", "integers" etc. Such types called **abstract** and they are mapped to different internal values inside specific DBMS. Some types might additionaly require set of parameters, for example string type requires it's length. Let's try to list 
all supported abstract types:

Type        | Parameters                | Description
---         | ---                       | ---
**primary** | ---                       | Special column type, usually mapped as integer + auto incrementing flag and added as table primary index column. You can define only one primary column in your table (you still can create compound primary key, see below).
bigPrimary  | ---                       | Same as primary but uses bigInteger to store it's values.
boolean     | ---                       | Boolean type, some databases will store it as integer (1/0).
integer     | ---                       | Database specific integer (usually 32 bits).
tinyInteger | ---                       | Small/tiny integer, check your DBMS to check it's size.
bigInteger  | ---                       | Big/long integer (usually 64 bits), check your DBMS to check it's size.
**string**  | [length:255]              | String with specified lenth, perfect type for emails and usernames as it can be indexed. 
text        | ---                       | Database specific type to store text data. Check DBMS to find size limitations.
tinyText    | ---                       | Tiny text, same as "text" for most of databases. Differs only in MySQL.
longText    | ---                       | Long text, same as "text" for most of databases. Differs only in MySQL.
double      | ---                       | [Double precision number.] (https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
float       | ---                       | Single precision number, usually mapped into "real" type in database. 
decimal     | precision,&nbsp;[scale:0] | Number with specified precision and scale.
datetime    | ---                       | To store specific date and time, DBAL will automatically force UTC timezone for such columns.
date        | ---                       | To store date only, DBAL will automatically force UTC timezone for such columns.
time        | ---                       | To store time only.
*timestamp* | ---                       | Timestamp without timezone, DBAL will automatically convert incoming values into UTC timezone. Do not use such column in your objects to store time (use datetime instead) as timestamps will behave very specific to select DBMS.
binary      | ---                       | To store binary data. Check specific DBMS to find size limitations.
tinyBinary  | ---                       | Tiny binary, same as "binary" for most of databases. Differs only in MySQL.
longBinary  | ---                       | Long binary, same as "binary" for most of databases. Differs only in MySQL.
json        | ---                       | To store JSON structures, such type usually mapped to "text", only Postgres support it nativelly.

> Attention, in some cases type returned by `ColumnSchema->abstractType()` migh not be the same as declared one, such problem may occure in cases when DBMS uses same internal type for multiple abstract type, for example most of databases does not differentiate long/short/medium text and binary types. However, such thing does not breaks anything in schema syncronization as DBAL creates operations based on difference in internal database type, not based on declared abstract one.

### Enum Type
One of the types which require additional description is enum. Such type nativelly exists only in MySQL database, in other DBMS it will be emulated using string type with associated constrain. To define enum type you have to list it's values:

```php
$schema->column('status')->enum(['active', 'disabled']);

//Alternative definition
$schema->enum('statusB', ['active', 'disabled']);
```

As in other cases declared schema will be synced will database one, so you can add and remove enum values at any moment. 

### Default values
It's recommended to set default value for enum and some other columns, setting default value can be performed using `defaultValue()`:

```php
$schema->column('status')->enum(['active', 'disabled'])->defaultValue('disabled');
$schema->enum('status_b', ['active', 'disabled'])->defaultValue('active');
```

And again, you can change default value at any moment. If you wish to drop default value simple set method argument as `null`.

```php
$schema->enum('statusB', ['active', 'disabled'])->defaultValue(null);
```

### Nullable columns
Spiral created nullable columns by default so you can alter structure of already existed table without getting errors from database. This behaviour might not work in some cases so you are given ability to mark column as nullable not nullable at any moment, simply declare:

```php
$schema->string('name', 64)->nullable(false);
```

You can change NULL/NOT NULL flag at any moment you want unless it's violates your table data. Additionally you can try to combine NOT NULL column with non empty default value, this will allow you to add new columns to non empty table.

```php
$schema->integer('new_column')->nullable(false)->defaultValue(0);
```

> ORM will automatically resolve default value for casted columns, this provides you ability to add columns to non empty tables without being worring about DBMS reject such update.

## Primary Index
Table primary index can be set only while creation, as you remember it will be set automatically once column with type "primary" or "bigPrimary" declared in your schema. In some cases you would like to declare compound or custom primary keys, you can use table method `setPrimaryKeys()` for that and feed it with name(s) of columns.

```php
$schema->primary('id');
$schema->string('something', 16);
$schema->setPrimaryKeys(['id', 'something']);
```

> You are not able to change primary keys after table beign created.

## Indexes
There is no table can existed without few indexes being added, DBAL simplifies index definition by only requiring column names from used to be set.

```php
$schema = $database->table('other_table')->schema();

$schema->primary('id');
$schema->string('name', 64)->nullable(false);

$schema->string('email');

$schema->index('email'); //Simple index
$schema->column('email')->index(); //You can also use alternative declaration for simple indexes

$schema->index('name', 'email'); //Compound index

$schema->save();
```

If you wish to add unique index you can change your code a little bit:

```php
$schema->column('email')->unique(); //Simple unique index
$schema->unique('name', 'email');  //Compound index
$schema->index('name', 'email')->unique(true);  //Alternative definition
```

If you wish to change index from unique to non unique, simple state that:

```php
$schema->index('name', 'email')->unique(false);
```

> Attention, you can not add indexes to text or binary columns. You have to remember about limitations current DBMS applies to it's indexes, for example you can not create unique index for non empty talbe with invalid (from standpoint of index) data. Some databases also has maximum index size and etc.

## Foreign Keys
When we talking about relational databases we obviously talking about relations and constraints. DBAL provides simple way to declare extenrnal table reference (foreign key) for any desired column in your schema, the only requiment is to make sure that inner and outher columns has same type. Let's try to create two simple tables first:

```php
$first = $database->table('first')->schema();

$first->primary('id');
$first->string('name', 64);
$first->string('email');

$first->save();

$second = $database->table('second')->schema();

$second->bigPrimary('id');
$second->string('title');

$second->save();
```

Now, we might want to create foreign key from second to first table, we can do that by declaring column "first_id" and adding reference to it (again, we can do it at any moment). Due we want to link our inner key to primary key of "first" table, we have to choose it's type, in our case primary correlates to integer (see abstraction types table):

```php
$second->integer('first_id')->references('first', 'id');
```

If we using MySQL connnectio DBAL will generate following SQL:

```sql
ALTER TABLE `primary_second` ADD COLUMN `first_id` int (11) NULL;
ALTER TABLE `primary_second` ADD CONSTRAINT `primary_second_foreign_first_id_55f205f594a3a` FOREIGN KEY (`first_id`) REFERENCES `primary_first` (`id`) ON DELETE NO ACTION ON UPDATE NO ACTION;
```

Such foreing key will only link two tables together, but it will not allow us to control data integriry (remove child data when parent is removed), we can do that by specifying delete and update rules:

```php
$foreignKey = $second->integer('first_id')->references('first', 'id');

$foreignKey->onDelete(ReferenceInterface::CASCADE);
$foreignKey->onUpdate(ReferenceInterface::CASCADE);
```

Now, when record in "first" table will be removed related data from "second" table will be wiped also. You can read more about different actions [here] (https://en.wikipedia.org/wiki/Foreign_key#Referential_actions).

> Please note that not every DBMS support actions outside of NO ACTION and CASCADE. In addition to that some databases (hi, Microsoft) may forbid multiple foreing keys with CASCADE action in one table to avoid reference loop.

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
    $schema->column('column');
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
