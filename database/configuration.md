# Database - Installation and Configuration
The `spiral/database` component is included by default in Web and GRPC builds. The DBAL focuses mainly on unifying database 
access rather than trying to get 100% of the specific DBMS feature set.
 
However, you can always use direct queries to bypass the spiral abstractions.

> Spiral DBAL supports MySQL, MariaDB, SQLite, PostgresSQL and SQLServer (Windows) databases.

## Installation
To install the component in alternative bundles or as standalone library: 

```bash
$ composer require spiral/database
```

Activate the bootloader `Spiral\Bootloader\Database\DatabaseBootloader` in your application:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Database\DatabaseBootloader::class,
    // ...
];
```

To enable migrations component:

```bash
$ composer require spiral/migrations
```

And corresponding bootloader:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Database\MigrationsBootloader::class,
    // ...
];
```

## Configuration
By default, the database configuration is located in `app/config/database.php` file. Configuration include set of options 
for each database driver, database-driver association and database aliases.

```php
<?php

declare(strict_types=1);

use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'runtime'],
    ],
    'drivers'   => [
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
        ],
    ]
];
```

### Declare Connection
To create new database connection add new section or alter existed options of `drivers` section of your configuration, 
you are able to use `env` function to keep your passwords and usernames separately.

```php
<?php

declare(strict_types=1);

use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'mysql'],
    ],
    'drivers'   => [
        'mysql'     => [
            'driver'     => Driver\MySQL\MySQLDriver::class,
            'connection' => 'mysql:host=127.0.0.1;dbname=' . env('DB_NAME'),
            'username'   => env('DB_USERNAME'),
            'password'   => env('DB_PASSWORD'),
            'options'    => []
        ],
        'postgres'  => [
            'driver'     => Driver\Postgres\PostgresDriver::class,
            'connection' => 'pgsql:host=127.0.0.1;dbname=' . env('DB_NAME'),
            'username'   => env('DB_USERNAME'),
            'password'   => env('DB_PASSWORD'),
            'options'    => []
        ],
        'runtime'   => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            'username'   => 'sqlite',
            'password'   => '',
            'options'    => []
        ],
        'sqlServer' => [
            'driver'     => Driver\SQLServer\SQLServerDriver::class,
            'connection' => 'sqlsrv:Server=MY-PC;Database=' . env('DB_NAME'),
            'username'   => env('DB_USERNAME'),
            'password'   => env('DB_PASSWORD'),
            'options'    => []
        ]
    ]
];
```

> Use connection option `options` to set PDO specific attributes.

### Declare Database
In order to access connected database we have to add it into `databases` section first:

```php
<?php

declare(strict_types=1);

use Spiral\Database\Driver;

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
You application and modules can access database multiple different ways. Database aliasing allows you to use separate 
databases with relation to one physical database.

> Use aliases to configure IoC autowiring.

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

use Spiral\Database\Driver;

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

### View available drivers and tables
To view available databases, drivers and tables:

```bash
$ php app.php db:list
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
To view the details about particular table:

```bash
$ php app.php db:table posts
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

## Standalone Usage
You can initiate DBAL component as standalone library. The configuration can be provided in a form of array:

```php
use Spiral\Database;

$dbConfig = new Database\Config\DatabaseConfig([
    'default'     => 'default',
    'databases'   => [
        'default' => [
            'connection' => 'sqlite'
        ]
    ],
    'connections' => [
        'sqlite' => [
            'driver'  => Database\Driver\SQLite\SQLiteDriver::class,
            'options' => [
                'connection' => 'sqlite:database.db',
                'username'   => '',
                'password'   => '',
            ]
        ]
    ]
]);

$dbal = new Database\DatabaseManager($dbConfig);
```

To create Database instance manually (without the DatabaseManager):

```php
use Spiral\Database;

$driver = new Database\Driver\SQLite\SQLiteDriver(
   [
       'connection' => 'sqlite:db.db'
   ]
);
       
$db = new Database\Database(
   'name',
   '',
   $driver,
   $driver // read only driver (optional)
);

dump($db->getTables());
```
