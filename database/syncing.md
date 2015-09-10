## Schema Writers
One of the most important part of Spiral DBAL is ability to alter database table schema using set of schema abstractions. Such abstractions provide ability to describe desired table rather than creating it operation by operation.

## Principle of Work
Before any operation/declaration can be applied to table schema, DBAL will load currently existed structure from database and [normalize it into internal format] (reading.md). As result, you are allowed to apply modification to table schema using declarative way instead of imperative, once schema **save** are requested - DBAL will generate set of creation and altering operation based on difference between declared and existed schemas. 
> Unfortunatelly some SQL features got simplified to fit, for example primary table key is described as column type, not index.
> The side effect of using this methodic - you can not remove any table element like column or index by *non declaring* it.

You can also use additional schema operations to remove or rename table elements.

## To Start
To get instance of TableSchema which we can manipulate with, we can use similar way described in [Schema Readers (make sure your read them first)] (reading.md). The only difference - we don't need to check table existence. We are going to use controller actions as example:

```php
public function index(Database $database)
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

The only thing we have to do now to push desired schema to our database is execute: `$schema->save()`. Depending on driver you using DBAL generate different SQL statements to create table, in our case we are using default MySQL database (with prefix "primary_") so our statement will look like:

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

> By default spiral marks every column as nullable, you can easily alter such behaviour (see below), in addition to that - ORM will make every column NOT NULL with not empty default value.

If you will try to execute such script again, you might notice that no table altering sql were generated (check [Profiler] (/modules/profiler.md) logging tab), this happen due spiral fetch exsited schema from databases and found no differences between it and declared one. If you want to add new column to your table or even change type of exsited one you can simply alter your code:

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

> You can also notice that Postgres does not have statement to change type of "description" column, this happend due declared abstract type "longText" does not differs from simple "text" type in postgres database.

> Attention, not every type can be easily changed in some databases. Make sure you are not violating DBMS specific cross type conversion.

### Abstract Types
As you can notice based on provided example we defined required columns using types named "string", "integers" etc. Such types called "abstract" and they are mapped to different internal values inside specific DBMS. Some types might additionaly require set of parameters, for example string type requires it's length. Let's try to list 
all supported abstract types:

Type        | Parameters            | Description
---         | ---                   | ---
**primary** | ---                   | Special column type, usually mapped as integer + auto incrementing flag and added as table primary index column. You can define only one primary column in your table (you still can create compound primary key, see below).
bigPrimary  | ---                   | Same as primary but uses bigInteger to store it's values.
boolean     | ---                   | Boolean type, some databases will store it as integer (1/0).
integer     | ---                   | Database specific integer (usually 32 bits).
tinyInteger | ---                   | Small/tiny integer, check your DBMS to check it's size.
bigInteger  | ---                   | Big/long integer (usually 64 bits), check your DBMS to check it's size.
**string**  | [length:255]          | String with specified lenth, perfect type for emails and usernames as it can be indexed. 
text        | ---                   | Database specific type to store text data. Check DBMS to find size limitations.
tinyText    | ---                   | Tiny text, same as "text" for most of databases. Differs only in MySQL.
longText    | ---                   | Long text, same as "text" for most of databases. Differs only in MySQL.
double      | ---                   | [Double precision number.] (https://en.wikipedia.org/wiki/Double-precision_floating-point_format)
float       | ---                   | Single precision number, usually mapped into "real" type in database. 
decimal     | precision, [scale:0]  | Number with specified precision and scale.
datetime    | ---                   | To store specific date and time, DBAL will automatically force UTC timezone for such columns.
date        | ---                   | To store date only, DBAL will automatically force UTC timezone for such columns.
time        | ---                   | To store time only.
*timestamp* | ---                   | Timestamp without timezone, DBAL will automatically convert incoming values into UTC timezone. Do not use such column in your objects to store time (use datetime instead) as timestamps will behave very specific to select DBMS.
binary      | ---                   | To store binary data. Check specific DBMS to find size limitations.
tinyBinary  | ---                   | Tiny binary, same as "binary" for most of databases. Differs only in MySQL.
longBinary  | ---                   | Long binary, same as "binary" for most of databases. Differs only in MySQL.
json        | ---                   | To store JSON structures, such type usually mapped to "text", only Postgres support it nativelly.

> Attention, in some cases type returned by `ColumnSchema->abstractType()` migh not be equal to declared one, this happen in cases when DBMS uses same internal type for multiple abstract, for example most of databases does not differentiate long/short/medium text and binary types. This is not a problem for schema syncronization as DBAL creates operations based on difference in internal database type, not based on declared abstract one.

### Enum Type


### Nullable columns

### Default values

## Primary Index




## Indexes

## Foreign Keys

## Additional Operations