# The Basics — Working with Databases and ORM

CycleORM is an ORM (Object-Relational Mapping) and Data Modelling engine, which means it allows you to interact with 
your database using plain PHP objects, rather than writing raw SQL. It has a lot of features like support for different
types of relations between tables, lazy and eager loading, and the ability to work with different types of PHP objects. It also has a flexible configuration options and
supports different types of databases like MySQL, PostgresSQL, and SQLite. Overall, it's designed to make working with
databases in PHP easier and more efficient.

The spiral framework provides a component `spiral/cycle-bridge` to work with the ORM and database in your application.
The package is included in `spiral/app` skeleton and will be suggested to install when you create a new project.

## Installation

If you want to install the component in you application follow the next steps:

Make sure that your server is configured with following PHP version and extensions:

- PHP 8.1+
- PDO Extension with desired database drivers

Run the following command to install the package:

```terminal
composer require spiral/cycle-bridge
```

After package install you need to add bootloader `Spiral\Cycle\Bootloader\BridgeBootloader` from the package in your
application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
];
```

You can exclude `Spiral\Cycle\Bootloader\BridgeBootloader` bootloader and select only needed bootloaders by writing them
separately.

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // Database
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,

    // Close the database connection after every request automatically (Optional)
    // CycleBridge\DisconnectsBootloader::class,

    // ORM
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,
    CycleBridge\CommandBootloader::class,

    // Validation (Optional)
    CycleBridge\ValidationBootloader::class,

    // DataGrid (Optional)
    CycleBridge\DataGridBootloader::class,

    // Database Token Storage (Optional)
    CycleBridge\AuthTokensBootloader::class,

    // Migrations and Cycle Scaffolders (Optional)
    CycleBridge\ScaffolderBootloader::class,
    
    // Prototyping (Optional)
    CycleBridge\PrototypeBootloader::class,
];
```

## Database

### Configuration

The configuration for spiral framework's database services is located in your application's `app/config/database.php`
configuration file. In this file, you may define all of your database connections, as well as specify which connection
should be used by default. Most of the configuration options within this file are driven by the values of your
application's environment variables.

Here is an example configuration file that defines a database connection:

```php
use Cycle\Database\Config;

return [
    /**
     * Database logger configuration
     */
    'logger' => [
        'default' => null, // Default log channel for all drivers (The lowest priority)
    ],

    'default' => env('DB_DEFAULT', 'default'),

    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(
            connection: new Config\SQLite\FileConnectionConfig(
                database: env('DB_DATABASE', directory('root') . 'runtime/app.db')
            ),
            queryCache: true,
        ),
        // ...
    ],
];
```

> **Note**
> Read more about the configuration of the database in
> the [Database - Installation and Configuration](https://cycle-orm.dev/docs/database-configuration) on the official
> site.

### Access database

You can access your databases in controllers and services in several ways:

#### Using Database provider

```php
use Cycle\Database\DatabaseProviderInterface;

final class SomeService 
{
    public function __construct(
        private readonly DatabaseProviderInterface $dbal
    ) {}
    
    public function store(): void
    {
        // Default database
        dump($this->dbal->database());
    
        // Using alias default which points to primary database
        dump($this->dbal->database('default'));
    
        // Secondary
        dump($this->dbal->database('slave'));
    }
}
```

#### Using method and Constructor Injections

The DBAL component fully supports [IoC injections](../advanced/injectors.md) based on the database name and their
aliases:

```php
use Cycle\Database\DatabaseInterface;

public function store(
    DatabaseInterface $database, 
    DatabaseInterface $primary,
    DatabaseInterface $slave
): void {
    // Database is an alias for "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

#### Using prototype

Access `Cycle\Database\DatabaseProviderInterface` and default database instance using `PrototypeTrait`:

```php
final class SomeService 
{
    use PrototypeTrait;
    
    public function store(): void
    {
        dump($this->dbal);
        dump($this->db); // default db
    }
}
```

### Run Queries

To run the database query, use the method `query`:

```php
dump(
    $db->query('SELECT * FROM users WHERE id > ?', [
        1
    ])->fetchAll()
);
```

To execute an update or delete statement, use the alternative method `execute`:

```php
dump(
    $db->execute('DELETE FROM users WHERE id > ?', [
        1,
    ]) // number of affected rows 
);
```

> **Note**
> Read how to use query builders [here](https://cycle-orm.dev/docs/database-query-builders).

### Console Commands

The default Web and GRPC bundles include a set of console commands to view the database schema.

Activate the bootloader `Spiral\Cycle\Bootloader\CommandBootloader` in your application:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cycle\Bootloader\CommandBootloader::class,
    // ...
];
```

#### View available drivers and tables

To view available databases, drivers, and tables:

```terminal
php app.php db:list
```

The output:

```output
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mName (ID):[39m | [32mDatabase:[39m  | [32mDriver:[39m | [32mPrefix:[39m | [32mStatus:[39m   | [32mTables:[39m | [32mCount Records:[39m |
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mdefault[39m    | runtime.db | SQLite  | ---     | connected | users   | [33m0[39m              |
|            |            |         |         |           | posts   | [33m0[39m              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

#### View table schema

To view the details about a particular table:

```terminal
php app.php db:table posts
```

The output:

```output
Columns of default.posts:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int            | primary        | int       | ---            |
| title   | string (255)   | text           | string    | ---            |
| user_id | int            | integer        | int       | ---            |
+---------+----------------+----------------+-----------+----------------+

Indexes of default.posts:
+-----------------------------------+-------+----------+
| Name:                             | Type: | Columns: |
+-----------------------------------+-------+----------+
| posts_index_user_id_5e32b9642a0ff | INDEX | user_id  |
+-----------------------------------+-------+----------+

Foreign Keys of default.posts:
+------------------+---------+----------------+-----------------+------------+------------+
| Name:            | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+------------------+---------+----------------+-----------------+------------+------------+
| posts_user_id_fk | user_id | users          | id              | CASCADE    | CASCADE    |
+------------------+---------+----------------+-----------------+------------+------------+
```

## ORM

### Configuration

The configuration for spiral framework's ORM services is located in your application's `app/config/cycle.php`

```php
use Cycle\ORM\SchemaInterface;

return [
    'schema' => [
        /**
         * true (Default) - Schema will be stored in a cache after compilation.
         * It won't be changed after entity modification. Use `php app.php cycle` to update schema.
         *
         * false - Schema won't be stored in a cache after compilation.
         * It will be automatically changed after entity modification. (Development mode)
         */
        'cache' => false,

        /**
         * The CycleORM provides the ability to manage default settings for
         * every schema with not defined segments
         */
        'defaults' => [
            SchemaInterface::MAPPER => \Cycle\ORM\Mapper\Mapper::class,
            SchemaInterface::REPOSITORY => \Cycle\ORM\Select\Repository::class,
            SchemaInterface::SCOPE => null,
            SchemaInterface::TYPECAST_HANDLER => [
                \Cycle\ORM\Parser\Typecast::class
            ],
        ],

        'collections' => [
            'default' => 'array',
            'factories' => [
                'array' => new \Cycle\ORM\Collection\ArrayCollectionFactory(),
                // 'doctrine' => new \Cycle\ORM\Collection\DoctrineCollectionFactory(),
                // 'illuminate' => new \Cycle\ORM\Collection\IlluminateCollectionFactory(),
            ],
        ],

        /**
         * Schema generators (Optional)
         * null (default) - Will be used schema generators defined in bootloaders
         */
        'generators' => null,

        // 'generators' => [
        //        \Cycle\Schema\Generator\ResetTables::class,
        //        \Cycle\Annotated\Embeddings::class,
        //        \Cycle\Annotated\Entities::class,
        //        \Cycle\Annotated\TableInheritance::class,
        //        \Cycle\Annotated\MergeColumns::class,
        //        \Cycle\Schema\Generator\GenerateRelations::class,
        //        \Cycle\Schema\Generator\GenerateModifiers::class,
        //        \Cycle\Schema\Generator\ValidateEntities::class,
        //        \Cycle\Schema\Generator\RenderTables::class,
        //        \Cycle\Schema\Generator\RenderRelations::class,
        //        \Cycle\Schema\Generator\RenderModifiers::class,
        //        \Cycle\Annotated\MergeIndexes::class,
        //        \Cycle\Schema\Generator\GenerateTypecast::class,
        // ],
    ],

    /**
     * Prepare all internal ORM services (mappers, repositories, typecasters...)
     */
    'warmup' => false,
];
```