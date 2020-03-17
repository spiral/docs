# Database - Logical Isolation
The spiral/database component provides the ability to use a single connection for multiple, logically isolated databases.
The isolation performed using table prefix, which automatically added to every SQL identifier.

## Configuration
To enable database prefix use option `prefix` of your database in `app/config/database.php`:

```php
<?php

declare(strict_types=1);

use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        'default' => [
            'driver' => 'runtime',
            'prefix' => 'my_prefix_'
        ],
    ],
    'drivers'   => [
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            'profiling'  => true,
        ],
    ]
];
```

## Runtime configuration
You can isolate database in runtime using `withPrefix` method:

```php
public function index(Database $db)
{
    dump($db->withPrefix('my_db_prefix')->getTables());
}
```

> The schema introspection and declaration will work with the isolated database by automatically adding a prefix to all declared
> tables.
