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

## View table schema
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