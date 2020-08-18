# 数据库 - 安装和配置

`spiral/database` 组件默认包含在 Web 和 GRPC 应用模板中。DBAL 主要侧重于统一的数据库访问，而不是试图百分之百地实现特定类型的 DBMS 完整功能。
 
当然，实际使用中你可以通过直接查询来绕过 Spiral 的抽象层。

> Spiral DBAL 支持 MySQL、MariaDB、SQLite、PostgresSQL 和 SQLServer(Windows) 数据库。

## 安装

如果需要在自己构建的项目中独立安装数据库组件，可以执行：

```bash
$ composer require spiral/database
```

然后在应用中激活 `Spiral\Bootloader\Database\DatabaseBootloader` 引导程序：

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Database\DatabaseBootloader::class,
    // ...
];
```

如果需要使用数据库迁移（migration）组件的话，可以安装：

```bash
$ composer require spiral/migrations
```

激活对应的引导程序：

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\Database\MigrationsBootloader::class,
    // ...
];
```

## 配置

默认情况下，数据库配置存放在 `app/config/database.php` 文件中。数据库相关的配置包含了每个数据库驱动、数据库连接和数据库别名的一系列选项。

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
            'options'    => [
                'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            ]
        ],
    ]
];
```

> 上面的 `default` 指默认要使用的连接（database），而 `databases` 指启用的数据库连接，每个连接可以关联到下面定义的一种数据库驱动（drivers）。

### 定义数据库驱动

To create new database connection add a new section or alter existed options of `drivers` section of your configuration, 
you can use `env` function to keep your passwords and usernames separately.

要创建新的数据库连接，请先在配置中的 `drivers` 部分中新增或者修改一个现有的驱动选项，可以使用 `env` 从环境变量中读取用户名和密码以避免敏感信息泄露。

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
            'options'    => [
                'connection' => 'mysql:host=127.0.0.1;dbname=' . env('DB_NAME'),
                'username'   => env('DB_USERNAME'),
                'password'   => env('DB_PASSWORD'),
            ]
        ],
        'postgres'  => [
            'driver'     => Driver\Postgres\PostgresDriver::class,
            'options'    => [
                'connection' => 'pgsql:host=127.0.0.1;dbname=' . env('DB_NAME'),
                'username'   => env('DB_USERNAME'),
                'password'   => env('DB_PASSWORD'),
            ]
        ],
        'runtime'   => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'options'    => [
                'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
                'username'   => 'sqlite',
                'password'   => '',
            ]
        ],
        'sqlServer' => [
            'driver'     => Driver\SQLServer\SQLServerDriver::class,
            'options'    => [
                'connection' => 'sqlsrv:Server=MY-PC;Database=' . env('DB_NAME'),
                'username'   => env('DB_USERNAME'),
                'password'   => env('DB_PASSWORD'),
            ]
        ]
    ]
];
```

> 可以使用连接的 `options` 选项来设置 PDO 特定属性。

### 定义数据库连接

要在代码中访问到某个数据库连接，需要首先在 `databases` 节中添加对应的连接：

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

> 定义连接时，支持多个连接指向同一个数据库驱动。

### 定义数据库别名

应用程序和模块可以以多种不同的方式访问数据库。数据库别名允许你使用不同的数据库连接来关联同一个物理数据库。

例如使用别名来配置 IoC 的自动匹配。

比如有一个控制器的构造函数是这样的：

```php
public function __construct(Database $db, Database $other)
{
}
```

为了把 `db` 和 `other` 指向具体的数据库实例，但是两个连接使用不同的表前缀：

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

## 控制台命令


默认的 Web 和 GRPC 应用模板中包含了一系列控制台命令，用于查看数据库结构：

### 查看可用的数据库连接和表

要查看可用的数据库连接、驱动和数据表，可以执行：

```bash
$ php app.php db:list
```

输出如下：

```bash
+------------+------------+---------+---------+-----------+---------+----------------+
| Name (ID): | Database:  | Driver: | Prefix: | Status:   | Tables: | Count Records: |
+------------+------------+---------+---------+-----------+---------+----------------+
| default    | runtime.db | SQLite  | ---     | connected | users   | 0              |
|            |            |         |         |           | posts   | 0              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

### 查看表结构

要查看一个具体的数据表的细节：

```bash
$ php app.php db:table posts
```

输出如下：

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

## 独立使用

你也可以把 DBAL 组件作为一个单独的库来初始化。需要以数组形式提供配置：

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

如果不使用 DatabaseManager, 也可以手动创建数据库连接实例：

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
   $driver // 只读数据库驱动 (可选)
);

dump($db->getTables());
```
