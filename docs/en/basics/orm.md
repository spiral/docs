# The Basics — Database and ORM

To utilize the ORM and database functionality in your application, Spiral offers
the [spiral/cycle-bridge](https://github.com/spiral/cycle-bridge) component.

## Installation

This component is automatically included in the `spiral/app` and can also be installed into existing projects through
the use of Composer by executing the following command:

```terminal
composer require spiral/cycle-bridge
```

After successful installation of the package, it is necessary to add the `Spiral\Cycle\Bootloader\BridgeBootloader`
bootloader to the Kernel:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
];
```

Alternatively, for more granular control, the `BridgeBootloader` can be excluded and selected the only needed
bootloaders.

The relevant code for this in Kernel would look like this:

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
    // CycleBridge\ValidationBootloader::class,

    // DataGrid (Optional)
    // CycleBridge\DataGridBootloader::class,

    // Database Token Storage (Optional)
    CycleBridge\AuthTokensBootloader::class,

    // Migrations and Cycle Scaffolders (Optional)
    CycleBridge\ScaffolderBootloader::class,
    
    // Prototyping (Optional)
    CycleBridge\PrototypeBootloader::class,
];
```

#### Disconnects bootloader

The bootloader serves the purpose of automatically closing the database connection after every request in long-running
applications. This is an optional bootloader and can be included or excluded as per the requirements of the specific
application.

## Configuration

### Database

The configuration for database services is located in `app/config/database.php` configuration file. In this file, you
may define all of your database connections, as well as specify which connection should be used by default. Most of the
configuration options within this file are driven by the values of your
application's environment variables.

Here is an example configuration file that defines a database connection:

```php app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DEFAULT'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

    'default' => env('DB_DEFAULT', 'default'),

    /**
     * The Spiral/Database module provides support to manage multiple databases
     * in one application, use read/write connections and logically separate
     * multiple databases within one connection using prefixes.
     *
     * To register a new database simply add a new one into
     * "databases" section below.
     */
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    /**
     * Each database instance must have an associated connection object.
     * Connections used to provide low-level functionality and wrap different
     * database drivers. To register a new connection you have to specify
     * the driver class and its connection options.
     */
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

> **See more**
> Read more about the configuration of the database in
> the [Database - Installation and Configuration](https://cycle-orm.dev/docs/database-configuration) on the official
> site.

### ORM

The configuration for Spiral framework's ORM services is located in your application's `app/config/cycle.php`

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

## Cycle ORM

CycleORM is a powerful and flexible Object-Relational Mapping (ORM) tool for PHP that allows developers to interact with
databases in an object-oriented way. It provides a range of features that make it easy to work with data including a
flexible configuration options, a powerful query builder and support for dynamic mapping of schemas.

It supports a variety of popular relational databases such as MySQL, MariaDB, PostgresSQL, SQLServer, and SQLite.

> **Note**
> Full documentation is available on the official site [CycleORM](https://cycle-orm.dev/docs).

### ORM instance

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

When you request a repository from the container, Spiral will automatically request the repository from
the ORM and associate it with the correct Entity.

### Transactions

To persist entity changes, your application services and controllers will require `Cycle\ORM\EntityManagerInterface`.

By default, the framework will automatically create a transaction on-demand from the container. Considering that
transactions always clean after the operation `run`, you can request it as a constructor parameter.

> **See more**
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

### Entity validation

The Cycle bridge provides a CycleBridge\ValidationBootloader bootloader that registers additional checkers for
the [spiral/validator](../validation/spiral.md) package. This bootloader includes two additional validation rules, which
enhance the functionality of the validator and allow for more efficient and effective data validation within the
application.

#### exists

Check if an entity with a given role and primary key exists.

By default, a rule will check if an entity exists by the primary key.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    #[Setter(filter: 'intval')]
    public int $id;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class // Entity role
                ] 
            ]       
        ]);
    }
}
```

You can also specify the field name and the value, which will be used to check if the entity exists.

```php
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class UpdateUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class, // Entity role
                    'username', // Field name
                ], 
            ],       
        ]);
    }
}
```

#### unique

Check if an entity with a given role is unique.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::unique', 
                    \App\Entity\User::class, // Entity role
                    'username', // Field name
                ] 
            ]       
        ]);
    }
}
```

### Entity behaviors

If you want to use [cycle/entity-behavior](https://cycle-orm.dev/docs/entity-behaviors-install/) package in your 
application, you need to install it first:

```bash
composer require cycle/entity-behavior
```

After that, you need to bind `Cycle\ORM\Transaction\CommandGeneratorInterface` 
with `\Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator` in the application container:

```php app/src/Application/Bootloader/EntityBehaviorBootloader.php
namespace App\Application\Bootloader;

use Cycle\ORM\Transaction\CommandGeneratorInterface;
use Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator;
use Spiral\Boot\Bootloader\Bootloader;

final class EntityBehaviorBootloader extends Bootloader
{
    protected const BINDINGS = [
        CommandGeneratorInterface::class => \Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator::class,
    ];
}
```

And finally, you need to register the `App\Application\Bootloader\EntityBehaviorBootloader` in the application kernel:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \App\Application\Bootloader\EntityBehaviorBootloader::class,
    ...
],
```

That's it! Now you can use entity behaviors in your application.

### Interceptors

#### Cycle Entity Resolution

The Cycle ORM integration provides a `Spiral\Cycle\Interceptor\CycleInterceptor` interceptor that allows you to
automatically resolve entities in controller methods by their primary key.

> **Note:**
> Read more about using interceptors in the [HTTP — Interceptors](../http/interceptors.md) section.

To activate interceptor:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        // ...
    ];
}
```

After that you can use a cycle entity injection in your controller methods:

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

use App\Entity\User;
use Spiral\Router\Annotation\Route;

final class HomeController
{
    #[Route('/users/<user>')]
    public function index(User $user)
    {
        dump($user);
    }
}
```

> **Note:**
> If an entity can't be found the 404 exception will be thrown.

### Long-Running

Cycle ORM aims to make the usage of the library in daemonized applications, such as PHP workers running under RoadRunner
or Swoole, simpler. The ORM provides multiple options to avoid memory leaks, which can also be applied to batch
operations. This helps ensure the stability and efficiency of the application when executing long-running processes.

The package will automatically clean the heap after each request. If you need to clean the heap manually, you can use
the following methods:

```php
use Cycle\ORM\ORMInterface;

class UserService
{
    public function __construct(
        private readonly ORMInterface $orm
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        // Create a new user
        
        $this->orm->getHeap()->clean();
    }
}
```

### Console Commands

The Cycle ORM integration provides multiple commands for easier control. You can get help for any of the commands using

```bash
php app.php help cycle...
```

> **Note**
> Make sure to enable `Spiral\Cycle\Bootloader\CommandBootloader` after the cycle bootloaders to activate helper
> commands.

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

### Logging

Spiral provides the capability to log database queries through the use of the `spiral/logger` component.
This component uses Monolog as its default logging driver.

> **See more**
> Read more about the logger in the [The Basics — Logging](../basics/logging.md) section.

The database logging drivers can be configured in the `logger` section of `app/config/database.php` configuration file.

```php app/config/database.php
return [
    'logger' => [
        'default' => null,
        'drivers' => [],
    ],

    // ...
];
```

If no logger driver is defined, the channel with the current database driver name will be used. If the SQLite driver is
used to execute queries, the monolog channel name will automatically be set
to `Cycle\Database\Driver\SQLite\SQLiteDriver`.

In this case, you can configure a monolog handler for the channel `Cycle\Database\Driver\SQLite\SQLiteDriver` in order
to log all the database queries.

This can be done by adding the following code in the `app/config/monolog.php` file:

```php app/config/monolog.php
return [
    'handlers' => [
        // ...

        \Cycle\Database\Driver\SQLite\SQLiteDriver::class => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/db.log',
                    'level' => Logger::DEBUG,
                ],
            ],
        ],
    ],
    
    // ...
];
```

The `drivers` section of the `logger` is used to specify which database driver should use the log channel specified
by the key.

Let's consider the following database configuration:

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            'runtime' => 'console'
        ],
    ],
    
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    
    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(...),
        // ...
    ],
];
```

We can use the logger configuration array to map the `runtime` database driver to the `console` log channel.

And the following monolog config:

```php app/config/monolog.php
return [
    'handlers' => [
        //...
        'console' => [
            \Monolog\Handler\ErrorLogHandler::class,
        ],
    ],
];
```

With these configurations, every time you use the `runtime` database driver, its logs will be sent to
the `console` channel.

You can also point a specific database driver to a specific log channel. For example, you can have a separate log
channel for your SQLite database and another for your MySQL database. This way, you can monitor each database's logs
individually and troubleshoot any issues specific to each database. This helps you identify and fix issues quickly,
without having to sift through a large, complex log file.

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            \Cycle\Database\Driver\MySQL\MySQLDriver::class => 'db_logs',
            \Cycle\Database\Driver\SQLite\SQLiteDriver::class => 'console'
        ],
    ],
];
```

In this case every time when you will use `SQLiteDriver` database driver it will send logs to the `console` log channel.

You can also set the `default` key to a specific log channel. This will be used as the default log channel for all
queries
executed by the database drivers that are not specified in the `drivers` section.

```php app/config/database.php
return [
    'logger' => [
        'default' => 'console',
    ],
];
```

By setting a default log channel, it ensures that even if no specific log channel has been defined for a particular
database driver, there is still a channel available for logging.

For example, if a developer sets the default log channel as `console`, any database driver without a specific log 
channel assigned to it will have its logs directed to the `console` channel. This provides a fallback for situations 
where a log channel has not been defined, ensuring that all logs are captured.

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

<hr>

## Migrations

You can create config file `app/config/migration.php` if you want to configure Cycle ORM Migrations:

```php app/config/migration.php
return [
    /**
     * Directory to store migration files
     */
    'directory' => directory('app').'migrations/',

    /**
     * Table name to store information about migrations status (per database)
     */
    'table' => 'migrations',

    /**
     * When set to true no confirmation will be requested on migration run.
     */
    'safe' => env('APP_ENV') === 'production',
];
```

### Console Commands

| Command            | Description                                                                                       |
|--------------------|---------------------------------------------------------------------------------------------------|
| `migrate`          | Performs one or all the outstanding migrations.<br/>`--one` Execute only one (first) migration.   |
| `migrate:replay`   | Replays (down, up) one or multiple migrations.<br/>`--all` Replays all the migrations.            |
| `migrate:rollback` | Rolls back  (default) or multiple migrations.<br/>`--all` Rolls back all the executed migrations. |
| `migrate:init`     | Initiates the migrations component (create a migrations table).                                   |
| `migrate:status`   | Gets a list of all available migrations and their statuses.                                       |
