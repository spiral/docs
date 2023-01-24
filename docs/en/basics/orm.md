# The Basics — Database and ORM

The spiral framework provides a component `spiral/cycle-bridge` to work with the ORM and database in your application.

## Installation

The package is included in `spiral/app` skeleton and will be suggested to install when you create a new project, but if
you want to install it in your existing project, you can do it with composer.

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

## Cycle ORM

CycleORM is a powerful and flexible Object-Relational Mapping (ORM) tool for PHP that allows developers to interact with
databases in an object-oriented way. It provides a range of features that make it easy to work with data including a
flexible configuration options, a powerful query builder and support for dynamic mapping of schemas.

It supports a variety of popular relational databases such as MySQL, MariaDB, PostgresSQL, SQLServer, and SQLite.

> **Note**
> Full documentation is available on the official site [CycleORM](https://cycle-orm.dev/docs).

### Configuration

The configuration for spiral framework's ORM services is located in your application's `app/config/cycle.php`

```php app/config/cycle.php
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

### ORM

You can access the ORM instance from the container by using the `Cycle\ORM\ORMInterface` interface.

### Repositories

Let's imagine that we have a `User` entity 

```php
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
class User
{
    // ...
}
```

with a `UserRepository` repository.

```php 
class UserRepository extends \Cycle\ORM\Select\Repository
{
    public function findByEmail(string $email): ?User
    {
        return $this->findOne(['email' => $email]);
    }
}
```

You can request a repository from the ORM instance by yourself, by providing the entity or role name.

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\RepositoryInterface;

class UserService
{   
    private readonly RepositoryInterface $repository;

    public function __construct(
        Cycle\ORM\ORMInterface $orm
    ) {
        $this->repository = $orm->getRepository(User::class);
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findOne(['email' => $email]);
        // ...
    }
}
```

Yu can also request a repository from the container. The framework uses [IoC injections](../advanced/injectors.md) to 
inject repositories into your code that implement `Cycle\ORM\RepositoryInterface`.


```php
class UserService
{
    public function __construct(
        private readonly UserRepository $repository
    ) {
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findByEmail($email);
        // ...
    }
}
```

When you request a repository from the container, the Spiral Framework will automatically request the repository from
the ORM and associate it with the correct Entity.

### Transactions

To persist entity changes, your application services and controllers will require `Cycle\ORM\EntityManagerInterface`.

By default, the framework will automatically create a transaction on-demand from the container. Considering that
transactions always clean after the operation `run`, you can request it as a constructor parameter.

> **Note:**
> You can read more about transactions in
> the [CycleORM documentation](https://cycle-orm.dev/docs/advanced-entity-manager).

Here is an example of a service that uses the `EntityManagerInterface`:

```php
use Cycle\ORM\EntityManagerInterface;

class UserService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        $user = new User($name, $email);
        
        $this->entityManager->persist($user);
        $this->entityManager->run();
        
        return $user;
    }
}
```

> **Note:**
> Make sure that the `persist/delete` and run methods are always called within the same method scope while using
> service-specific transactions.

### Console Commands

The Cycle ORM integration provides multiple commands for easier control. You can get help for any of the commands using

```bash
php app.php help cycle...
```

> **Note**
> Make sure to enable `Spiral\Cycle\Bootloader\CommandBootloader` after the cycle bootloaders to activate helper
> commands.

#### Migrations

| Command            | Description                                                                                       |
|--------------------|---------------------------------------------------------------------------------------------------|
| `migrate`          | Performs one or all the outstanding migrations.<br/>`--one` Execute only one (first) migration.   |
| `migrate:replay`   | Replays (down, up) one or multiple migrations.<br/>`--all` Replays all the migrations.            |
| `migrate:rollback` | Rolls back  (default) or multiple migrations.<br/>`--all` Rolls back all the executed migrations. |
| `migrate:init`     | Initiates the migrations component (create a migrations table).                                   |
| `migrate:status`   | Gets a list of all available migrations and their statuses.                                       |

#### Database

| Command            | Description                                                                                                               |
|--------------------|---------------------------------------------------------------------------------------------------------------------------|
| `db:list [db]`     | Gets a list of available databases, their tables and records count.<br/>`db` database name.                               |
| `db:table <table>` | Describes a table schema of a specific database.<br/>`table` A table name (required).<br/>`--database` A source database. |

#### ORM and Schema

| Command         | Description                                                                                |
|-----------------|--------------------------------------------------------------------------------------------|
| `cycle`         | Updates (init) the cycle schema from the database and annotated classes.                   |
| `cycle:migrate` | Generates the ORM schema migrations.<br/>`--run` Automatically runs a generated migration. |
| `cycle:render`  | Renders the available CycleORM schemas.<br/>`--no-color` Displays output without colors.   |

> **Note**
> You can run any cycle command with the `-vv` flag to see a list of modified tables.

<hr>

## Database

### Configuration

The configuration for spiral framework's database services is located in your application's `app/config/database.php`
configuration file. In this file, you may define all of your database connections, as well as specify which connection
should be used by default. Most of the configuration options within this file are driven by the values of your
application's environment variables.

Here is an example configuration file that defines a database connection:

```php app/config/database.php
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