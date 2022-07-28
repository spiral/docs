# Database - Installation and Configuration

The `cycle/database` component is included by default in Web and GRPC builds. The DBAL focuses mainly on unifying
database access rather than trying to get 100% of the specific DBMS feature set.

However, you can always use direct queries to bypass the spiral abstractions.

> **Note**
> Spiral DBAL supports MySQL, MariaDB, SQLite, PostgresSQL, and SQLServer (Windows) databases.

## Installation

To install the component in alternative bundles or as a standalone library:

```bash
composer require cycle/cycle-bridge
```

Activate the bootloader `Spiral\Cycle\Bootloader\DatabaseBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\Cycle\Bootloader\DatabaseBootloader::class,
    // ...
];
```

To enable migrations component:

```php
protected const LOAD = [
    // ...
    Spiral\Cycle\Bootloader\MigrationsBootloader::class,
    // ...
];
```

## Configuration

By default, the database configuration located in `app/config/database.php` file. The configuration includes a set of
options for each database driver, database-driver association, and database aliases.

```php
<?php

declare(strict_types=1);

use Cycle\Database\Config;

return [
    /**
     * Database logger configuration
     */
    'logger' => [
        'default' => null, // Default log channel for all drivers (The lowest priority)
        'drivers' => [
            // By driver name (The highest priority)
            // See https://spiral.dev/docs/extension-monolog
            'runtime' => 'sql_logs',

            // By driver class (Medium priority)
            \Cycle\Database\Driver\MySQL\MySQLDriver::class => 'console',
        ],
    ],

    /**
     * Default database connection
     */
    'default' => env('DB_DEFAULT', 'default'),

    /**
     * The Cycle/Database module provides support to manage multiple databases
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

### Declare Connection

To create new database connection add a new section or alter existed options of `drivers` section of your configuration,
you can use `env` function to keep your passwords and usernames separately.

```php
<?php

declare(strict_types=1);

use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => [
            'driver' => env('DATABASE_DRIVER', 'mysql')
        ],
    ],
    'drivers'   => [
        'mysql' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: env('DB_NAME'),
                host: env('DB_HOST', '127.0.0.1'),
                port: env('DB_PORT', 3306),
                user: env('DB_USERNAME'),
                password: env('DB_PASSWORD'),
            ),
            queryCache: true
        ),
        'postgres' => new Config\PostgresDriverConfig(
            connection: new Config\Postgres\TcpConnectionConfig(
                database: env('DB_NAME'),
                host: env('DB_HOST', '127.0.0.1'),
                port: env('DB_PORT', 5432),
                user: env('DB_USERNAME'),
                password: env('DB_PASSWORD'),
            ),
            schema: 'public',
            queryCache: true,
        ),
        'runtime' => new Config\SQLiteDriverConfig(
            connection: new Config\SQLite\FileConnectionConfig(
                database: directory('runtime') . 'runtime.db'
            ),
            queryCache: true,
        ),
        'sqlServer' => new Config\SQLServerDriverConfig(
            connection: new Config\SQLServer\TcpConnectionConfig(
                database: env('DB_NAME'),
                host: env('DB_HOST', '127.0.0.1'),
                port: env('DB_PORT', 1433),
                user: env('DB_USERNAME'),
                password: env('DB_PASSWORD'),
            ),
            queryCache: true,
        ),
    ]
];
```

> **Note**
> Use connection option `options` to set PDO specific attributes.

### Declare Database

In order to access connected database we have to add it into `databases` section first:

```php
<?php

declare(strict_types=1);

// ...

return [
    'default'   => 'primary',
    'databases' => [
        'primary' => [
           'driver'  => 'mysql',
           'prefix'  => 'primary_'
        ],
        'secondary'=> [
            'driver'  => 'mysql',
            'prefix'  => 'secondary_'
        ]       
    ],
    'drivers'   => [
        // ...
    ]
];
```

### Aliases

Your application and modules can access the database in multiple different ways. Database aliasing allows you to use
separate databases with relation to one physical database.

> **Note**
> Use aliases to configure IoC auto wiring.

Example controller constructor:

```
public function __construct(Database $db, Database $other)
{
}
```

To point `db` and `other` to specific database instance:

```php
<?php

declare(strict_types=1);

// ...

return [
    'default'   => 'primary',
    'aliases'   => [
        'db'    => 'primary',
        'other' => 'secondary'
    ],
    'databases' => [
        'primary' => [
           'driver'  => 'mysql',
           'prefix'  => 'primary_'
        ],
        'secondary'=> [
            'driver'  => 'mysql',
            'prefix'  => 'secondary_'
        ]       
    ],
    'drivers'   => [
        // ...
    ]
];
```

## Console Commands

The default Web and GRPC bundles include a set of console commands to view the database schema.

Activate the bootloader `Spiral\Cycle\Bootloader\CommandBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\Cycle\Bootloader\CommandBootloader::class,
    // ...
];
```

### View available drivers and tables

To view available databases, drivers, and tables:

```bash
php app.php db:list
```

The output:

```bash
+------------+------------+---------+---------+-----------+---------+----------------+
| Name (ID): | Database:  | Driver: | Prefix: | Status:   | Tables: | Count Records: |
+------------+------------+---------+---------+-----------+---------+----------------+
| default    | runtime.db | SQLite  | ---     | connected | users   | 0              |
|            |            |         |         |           | posts   | 0              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

### View table schema

To view the details about a particular table:

```bash
php app.php db:table posts
```

The output:

```bash
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

> **Note**
> Read how to configure database [here](https://cycle-orm.dev/docs/database-configuration/2.x/en).
