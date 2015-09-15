# Spiral ORM, Basics
Spiral Framework provide simple but yet powerful ORM engine you can use in everyday development. Spiral ORM provides ability to work with multiple databases, automatically scaffold tables and define set of realtions which can be requested on demand or pre-loaded with parent entity. 

Spiral ORM model **Record** are based on [DataEntity] (/components/entity.md) class, so it will be the best to read about it first. In addition check this [behaviour schema] (/schemas.md) concept which used as base for ORM component.

## Record
Spiral ORM component does not require any configuration rather than making sure that [DBAL] (/database/overview.md) has valid database connections. The only thing you have to do to that using ORM is to create or generate desired **Record** model.

Such model must include property **schema** which is going to describe needed table colums and record relationships. Since ORM component uses DBAL as it's backbone, you can declare set of desired columns in a similar format as [schema writers] (/database/syncing.md).

Record generation command accept pre-defined set of fields using key `-f`, let's try to generate our first model: 'create:record user -d -f id:primary -f name:string(64) -f email:string -f status:enum(active,blocked)', we are using key `-d` to generate "defaults" property (see next). Resulted entity will be generated in `application/classes/Database/User.php`:

```php
class User extends Record 
{
    /**
     * @var array
     */
    protected $fillable = [
        
    ];

    /**
     * Entity schema.
     * 
     * @var array
     */
    protected $schema = [
        'id'     => 'primary',
        'name'   => 'string(64)',
        'email'  => 'string',
        'status' => 'enum(active,blocked)'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'   => [
            'notEmpty'
        ],
        'email'  => [
            'notEmpty'
        ],
        'status' => [
            'notEmpty'
        ]
    ];

    /**
     * @var array
     */
    protected $defaults = [
        
    ];
}
```

As you can see most of Record properties are really similar to `DataEntity` as can be configured same way, however we have few additional things we have to remember.

#### Schema
The most important part of any Record models is it's schema. Schema declares what columns you would like to see in associated table and, in addition to that, contains set of record [relations] (relations.md). In this tutorial we are going to stick to columns only.

Since we are creating model for new (non existed table) we would need to declare primary key, we can do it same way as in DBAL Schema Writers by assigning type "primary" or "bigPrimary".

```php
'id'     => 'primary',
```

>  Spiral ORM will detect your primary key name automatically so you don't nesessary need to use "id".

To provide additional type arguments, such as length, scale or enum values you can simply use braces as you would call `AbstractColumn` method.

```php
'name'   => 'string(64)',
'status' => 'enum(active,blocked)'
```

The only important difference between way how columns decribed in DBAL and ORM that Record model will force **NOT NULL** flag and default values for each column (to solve problems when schema changes modifies existed non empty table). If you wish to declare your column as nullable, simply add "nullable" to it's type definition.

```php
'column' => 'string, nullable'
```

#### Default Values
As you already know ORM will force set of default values for every declared column, usually such default values will be just an empty type casted value ("", 0 etc). You can declare your own default value using record property "defaults", in our case we have "enum" column, it's the best to force default value for it:

```php
protected $defaults = [
    'status' => 'active'    
];
```

#### Associated Table and Database
By default spiral will associte your Record model with default DBAL database and table which name are generated based on class name (in our case User => users). You can alter both of this values by declaring non empty properties `database` and `table`.

#### Indexes
In many cases you might want to declare indexes to be created in associated table, you can esily do it in your Record via propery `indexes`. Every index information must be located in a sub array of such property and include index type (self::INDEX or self::UNIQUE), let's try to add unique index on our `email` column.

```php
protected $indexes = [
    [self::UNIQUE, 'email']
];
```

> You can include as many columns into index as your DBMS allows you. Simply list columns as array values.

## Schema Update and Scaffolding
Once you done with declaring your `Record` model schema we can run ORM update. Such operation can be achieved by running command 'spiral up'. Such command should automatically generate related table "users" for your model using declared columns and indexes. We can view content of such table by executing 'db:describe users':

```
Columns of primary.users:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int (11)       | primary        | int       | ---            |
| name    | varchar (64)   | string         | string    | ---            |
| email   | varchar (255)  | string         | string    | ---            |
| status  | enum           | enum           | string    | active         |
+---------+----------------+----------------+-----------+----------------+

Indexes of primary.users:
+-------------------------------------------+--------------+----------+
| Name:                                     | Type:        | Columns: |
+-------------------------------------------+--------------+----------+
| newprefix_users_index_email_55f84337a039a | UNIQUE INDEX | email    |
+-------------------------------------------+--------------+----------+
```

As you can see our table looks exacly as it should (output may differ if you using different DBMS)!

> You have to execute `spiral up` every time you change any entity or orm property of your Record to pass updates into cached behaviour schema. There is no need to run schema update if you modifying custom properties or methods.

#### Adding new Columns
Since ORM based on DBAL syncronization mechanism, the only thing you need to do to add new columns into related record table - declare such column in model schema (same applied for indexes). Let's try add column "balance".

```php
protected $schema = [
    'id'     => 'primary',
    'name'   => 'string(64)',
    'email'  => 'string',
    'status' => 'enum(active,blocked)',
    'balance' => 'decimal(10,2)'
];
```

To add such column into our table we have to execute `spiral up` again (you cam set verbosity flag `-v` to see what is going on):

```
> spiral up -v
Migrator does not configured, skipping.

[primary_users] Adding column [`balance` decimal (10, 2) NOT NULL DEFAULT 0.000000] into table `primary_users`.
ORM Schema has been updated: 0.494 s, found records: 1
ODM Schema has been updated: 0.163 s, found documents: 1

Generating tooltips and hints for PHPStorm...
ODM virtual documentation were created: 0 classes
ORM virtual documentation were created: 3 classes

Inspecting available DataEntities...
+---------------+------+--------+----------+-----------+
| Entity        | Rank | Fields | Fillable | Validated |
+---------------+------+--------+----------+-----------+
| Database\User | Good | 5      | 0        | 3         |
+---------------+------+--------+----------+-----------+

Inspected entities 1, average rank Good (0.95).
```

> You can use model schema only to add new columns, renaming and removing must be done via migrations, meaning removing column declaration from model will not remove such column from table.

And again we can check if our table was successfully modified using command "db:describe users", this time i switched my connection to PostgresSQL:

```
Columns of primary.users:
+---------+-------------------------+----------------+-----------+-------------------------------------------+
| Column: | Database Type:          | Abstract Type: | PHP Type: | Default Value:                            |
+---------+-------------------------+----------------+-----------+-------------------------------------------+
| id      | serial                  | primary        | int       | nextval('primary_users_id_seq'::regclass) |
| name    | character varying (64)  | string         | string    | ---                                       |
| email   | character varying (255) | string         | string    | ---                                       |
| status  | character (7)           | enum           | string    | active                                    |
| balance | numeric (10, 2)         | decimal        | float     | ---                                       |
+---------+-------------------------+----------------+-----------+-------------------------------------------+

Indexes of primary.users:
+-----------------------------------------+--------------+----------+
| Name:                                   | Type:        | Columns: |
+-----------------------------------------+--------------+----------+
| primary_users_index_email_55f8450374ca9 | UNIQUE INDEX | email    |
+-----------------------------------------+--------------+----------+
```

#### Active vs Passive Schemas
In a given we created Record which will ensure that table exists in database and has valid set of columns and indexes.ORM component does not require you to declare every table column, even more you can associate your model with existed table without even declaring schema propery. In this case model will fetch fields list and available columns with their types from table directly, in this case our model may look like:

```php
class UserPassive extends Record
{
    protected $table = 'users';
}
```

However, in a next guide section, dedicated to relations, you'll find out that relations have ability to modify associated model schemas to declare needed columns and foreign keys. This is fine if we generating our database using Spiral Models, however in some cases you might want to connect Records to existed database. In this case, you can configure relations in a way to make them follow already existed structures. But, to be absolutelly sure that no relation or recor can modify our table we can delcare record constant ACTIVE_SCHEMA and set if value to false.

```php
class SomeRecord extends Record
{
    const ACTIVE_SCHEMA = false;
}
```

Such constant will forbid any modifications for associated table and ensure that spiral can not touch existed table. The rest of ORM functionality such as validations, selections eager loading and realtions are still preserved.

> You can also use passive models if you prefer to generate your database using migrations.

## Work with Record

#### Dirty Fields and Solid State

#### Validations

#### Atomic Number

## Timestamps Trait

## Inheritance and Abstract Records