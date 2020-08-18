# 数据库 - 逻辑隔离

`spiral/database` 组件提供了使用单一连接连接多个逻辑隔离的数据库的能力。
这样的隔离是用表前缀来实现的，表前缀会自动添加到每个 SQL 标识符前面。

## 配置

要启用数据库的表前缀，需要在数据库配置文件 `app/config/database.php` 中使用 `prefix` 选项：

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

## 运行时配置

你还可以在运行时使用 `withPrefix` 来访问特定前缀的表：

```php
public function index(Database $db)
{
    dump($db->withPrefix('my_db_prefix')->getTables());
}
```

> 数据库结构反射和结构声明会通过自动为所有的表定义语句添加前缀，从而访问逻辑隔离的数据库。
