# 数据库 - 查询构造器

首先，如果需要了解手动编写原生查询语句，可以阅读[这个文档](/zh_CN/database/access.md)。

DBAL 组件包含了一系列查询构造器，用于统一不同数据库的语法。并在应用程序的生命周期内简化在不同种类的数据库之间进行数据迁移。

## 开始之前

为了演示组件的查询构造能力，我们首先在默认数据库定义一个简单的的数据表：

```php
namespace App\Controller;

use Spiral\Database\Database;

class HomeController
{
    public function index(Database $db)
    {
        $schema = $db->table('test')->getSchema();

        $schema->primary('id');
        $schema->datetime('time_created');
        $schema->enum('status', ['active', 'disabled'])->defaultValue('active');
        $schema->string('name', 64);
        $schema->string('email');
        $schema->double('balance');
        $schema->save();
    }
}
```

> 更多有关数据库结构声明的介绍，请参阅[相关文档](/zh_CN/database/declaration.md)。

## 插入语句

我们可以用以下的代码来获得一个插入语句构造器（负责插入操作）：

```php
$insert = $db->insert('test');
```

然后给插入语句构造器添加字段的具体值用于插入数据到指定的表：

```php
$insert = $db->insert('test');

$insert->values([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]);
```

定义完成后，要正式执行插入查询，只要执行 `run()` 方法即可，该方法会返回最后插入记录的 id:

```php
dump($db->run());
```

> 你也可以使用更流利的链式语法：`$database->insert('table')->values(...)->run()`.

### 批量插入

如果需要批量插入多条记录，可以添加数据库支持一次性插入的任意数量的值：

```php
$insert->columns([
    'time_created', 
    'name', 
    'email', 
    'balance'
]);

for ($i = 0; $i < 20; $i++) {
    // 前面已经指定了字段，这里只需要提供值即可
    $insert->values([
        new \DateTime(),
        $this->faker->randomNumber(2),
        $this->faker->email,
        $this->faker->randomFloat(2)
    ]);
}

$insert->run();
```

### 快速插入

还可以跳过插入构造器的创建过程，直接与数据表对话：

```php
$table = $db->table('test');

dump($table->insertOne([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]));
```

> Table 类会自动执行查询并返回最新插入的记录的 id. 可以自行了解 Table 类的 `insertMultiple` 方法。

## select 查询构造器

Select 查询构造器实例可以通过两种非常简单的方式获取，一种是从数据库实例，一种是从数据表实例：

```php
$select = $db->table('test')->select();

// 另一种方法
$select = $db->select()->from('test');

// 另一种方法
$select = $db->test->select();
```

### 查询字段

默认情况下，Select 查询会从指定表中选择所有字段（`*`）。但我们在需要的情况下可以用 `columns` 方法修改要查询的字段集。

```php
$db->users->select()
    ->columns('name')
    ->fetchAll();
```

你可以用 select 查询作为适当的迭代器，或者直接使用 `run` 方法，后者会返回 `Spiral\Database\Statement` 的实例：

```php
foreach($select->getIterator() as $row) {
    dump($row);
}
```

如果要把所有记录作为数组返回，可以使用 `fetchAll` 方法：

```php
foreach($select->fetchAll() as $row) {
    dump($row);
}
```

任何时候，如果需要的话，可以调用 `sqlStatement` 方法来查看生成的 sql 语句：

```php
dump(
    $db->users->select()
        ->columns('name')
        ->sqlStatement()
);
```

### Where 语句

使用 `where`、`andWhere`、`orWhere` 可以添加 WHERE 条件：

#### 基础条件

我们以我们示例表中的 `status` 字段为例，添加一些简单的查询条件：

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where('status', '=', 'active');

foreach ($select as $row) {
    dump($row);
}
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `status` = 'active'        
```

> 请注意底层代码会使用 pdo 的 prepared 语句。

简单的 `=` 操作符可以省略：

```php
$select->where('status', 'active');
```

#### Where 操作符

第二个参数可以用来指定操作符（如果提供了三个参数）：

```php
$select->where('id', '>', 10);
$select->where('status', 'like', 'active');
```

如果使用 between 和 not between 操作，需要给 where 方法提供第四个参数：

```php
$select->where('id', 'between', 10, 20);
```

生成的 SQL 类似这样：

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` BETWEEN 10 AND 20  
```

#### 多个 Where 条件

可以直接链式调用多个 where 方法：

```php
$select
    ->where('id', 1)
    ->where('status', 'active');
```

`andWhere` 方法是 `where` 的别名，所以上面的查询也可以写为下面的形式，以提升可读性：

```php
$select
    ->where('id', 1)
    ->andWhere('status', 'active');
```

生成的 SQL 类似这样：

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` = 1 AND `status` = 'active'
```

Select 查询会根据你的运算符顺序和布尔运算符生成 SQL:

```php
$select
    ->where('id', 1)
    ->orWhere('id', 2)
    ->orWhere('status', 'active');
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` = 1 OR `id` = 2 OR `status` = 'active'
```

#### 复杂/分组 Where 条件

如果需要在使用多个 where 条件时，对它们进行分组，可以用匿名函数作为第一个参数：

```php
$select
    ->where('id', 1)
    ->where(
        static function (SelectQuery $select) {
            $select
                ->where('status', 'active')
                ->orWhere('id', 10);
        }
    );
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` = 1 AND (`status` = 'active' OR `id` = 10)
```

布尔操作符也是支持的：

```php
$select
    ->where('id', 1)
    ->orWhere(
        static function (SelectQuery $select) {
            $select
                ->where('status', 'active')
                ->andWhere('id', 10);
        }
    );
```

生成的 SQL 语句：

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` = 1 OR (`status` = 'active' AND `id` = 10)     
```

> 你可以根据需要嵌套任意数量的查询条件。

#### 简化/数组 Where 条件

还可以使用 [MongoDB 风格](https://docs.mongodb.org/manual/reference/operator/query/) 的 where 条件构造：

```php
$select->where([
    'id'     => 1,
    'status' => 'active'
]);
```

这样的代码会被识别为两个 where 方法调用，并生成以下的 SQL:

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE (`id` = 1 AND `status` = 'active')
```

当然也可以用嵌套的数组来指定比较运算符：

```php
$select->where([
    'id'     => ['in' => new Parameter([1, 2, 3])],
    'status' => ['like' => 'active']
]);
```

> 请注意，必须使用 Parameter 类封装所有的数组参数，标量参数会自动封装。

生成的 SQL:

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE (`id` IN (1, 2, 3) AND `status` LIKE 'active')
```

对单个字段使用多个条件也是支持的：

```php
$select->where([
    'id' => [
        'in' => [1, 2, 3],
        '<'  => 100
    ]
]);
```

在创建 where 分组的时候，可以使用 `@or` 和 `@and` 来简化分组：

```php
$select
    ->where(
        static function (SelectQuery $select) {
            $select
                ->where('id', 'between', 10, 100)
                ->andWhere('name', 'Anton');
        }
    )
    ->orWhere('status', 'disabled');
```

上面的代码可以简化为：

```php
$select->where([
    '@or' => [
        [
            'id'   => ['between' => [10, 100]],
            'name' => 'Anton'
        ],
        ['status' => 'disabled']
    ]
]);
```

以上两种形式的代码生成的 SQL 都一样：

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE ((`id` BETWEEN 10 AND 100 AND `name` = 'Anton') OR `status` = 'disabled')
```

你可以尝试分别用两种方法来定义 where 条件，然后选择更喜欢的一种。

#### 参数

Spiral 框架内部用 `Parameter` 类模拟所有的给定值，在某些情况下（数组）你可能需要直接传递 `Parameter` 类型。你可以执行 `run` 方法之前的任何时候改变参数的值：

```php
use Spiral\Database\Injection\Parameter;
// ...

$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where('id', $id = new Parameter(null));

// 绑定新的参数值
$id->setValue(15);

foreach ($select as $row) {
    dump($row);
}
```

> 你也可以传递第二个参数用于指定该参数值的 PDO 参数类型： `new Parameter(1, PDO::PARAM_INT)`.
> 在框架内部，每一个传递到 `where` 方法的值默认都会使用 Parameter 类进行封装。

如果你想自行定义你的参数包装逻辑，可以实现 `ParameterInterface` 接口。

#### SQL 片段和表达式

查询构造器允许你使用自定义的 SQL 代码或者表达式替换一些 where 语句。`Spiral\Database\Injections\Fragment` 和 `Spiral\Database\Injections\Expression` 就是用于这种需求的。

例如，用片段（Fragment）将原生SQL代码绕过转义添加到查询中：

```php
use Spiral\Database\Injection\Fragment;

// 255
$select->where('id', '=', new Fragment("DAYOFYEAR('2015-09-12')"));
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` = DAYOFYEAR('2015-09-12')
```

如果希望将复杂的值与用户参数进行比较，则可以用表达式（Expression）替换 where 参数中的字段：

```php
use Spiral\Database\Injection\Expression;

$select->where(
    new Expression("DAYOFYEAR(concat('2015-09-', id))"), 
    '=',
    255
);
```

```sql
SELECT
*
FROM `x_users`
WHERE DAYOFYEAR(concat('2015-09-', `id`)) = 255
```

> 请注意，表示式中所有的字段标识都会被自动加引号。

连接多个字段的方法也是一样：

```php
$select->where(new Expression("CONCAT(id, '-', status)"), '1-active');
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE CONCAT(`id`, '-', `status`) = '1-active'
```

如果数据库连接指定了表前缀，表达式中的表名称会被自动处理：

```php
$select->where(
    new Expression("CONCAT(test.id, '-', test.status)"), 
    '1-active'
);
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE CONCAT(`primary_test`.`id`, '-', `primary_test`.`status`) = '1-active'
```

你也可以在插入和更新语句中使用表达式和片段作为字段值。

> 安全提示：请尽可能不要在片段或者表达式中使用客户输入数据。

### 表和字段别名

查询构造器支持用户定义的表和字段别名：

```php
$select = $db->select()
    ->from('test as abc')
    ->columns([
        'id',
        'status',
        'name'
    ]);

$select->where('abc.id', '>', 10);

foreach ($select as $row) {
    dump($row);
}
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test` as `abc`
WHERE `abc`.`id` > 10
```

字段别名：

```php
$select = $db->select()
    ->from('test')
    ->columns([
        'id',
        'status as st',
        'name',
        "CONCAT(test.name, ' ', test.status) as name_and_status"
    ]);

foreach ($select as $row) {
    dump($row);
}
```

生成的 SQL:

```sql
SELECT
    `id`, 
    `status` as `st`, 
    `name`, 
    CONCAT(`primary_test`.`name`, ' ',`primary_test`.`status`) as `name_and_status`
FROM `primary_test`
```

#### 子查询和嵌套查询

Spiral 的每个查询构造器都是实现了 `FragmentInterface` 接口的实例，因此在你需要复杂的嵌套查询时，有能力使用它们：

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where(
    'id',
    'IN',
    $database->select('id')
        ->from('test')
        ->where('id', 'BETWEEN', 10, 100)
);

foreach ($select as $row) {
    dump($row);
}
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE `id` IN  (SELECT
`id`
FROM `primary_test`
WHERE `id` BETWEEN 10 AND 100)  
```

你还可以在 where 语句中使用嵌套查询的返回值：

```php
$select->where(
    $db->select('COUNT(*)')
        ->from('test')
        ->where('id', 'BETWEEN', 10, 100), 
    '>',
    1
);
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE (SELECT
COUNT(*)
FROM `primary_test`
WHERE `id` BETWEEN 10 AND 100) > 1   
```

你也可以使用 `Expression` 类在父查询和子查询中交换字段标识：

```php
$select = $db->select()
    ->from('test')
    ->columns(['id', 'status', 'name']);

$select->where(
    $db->select('name')
        ->from('users')
        ->where('id', '=', new Expression('test.id'))
        ->where('id', '!=', 100),
    'Anton'
);
```

```sql
SELECT
`id`, `status`, `name`
FROM `primary_test`
WHERE (SELECT
`name`
FROM `primary_users`
WHERE `id` = `primary_test`.`id` AND `id` != 100) = 'Anton'
```

> 嵌套查询只有在嵌套查询与主建库属于同一个数据库时才可用。

### Having
使用 `having`、`orhaving` 和 `andHaving` 方法来定义 HAVING 条件。使用方法与 WHERE 语句基本一致。

### 联合查询

你可以使用 `leftJoin`、`join`、`rightJoin`、`fullJoin` 和 `innerJoin` 方法将任何需要的数据表加入到查询中：

```php
$select = $db->table('test')
    ->select(['test.*', 'u.name as u']);

$select->leftJoin('users', 'u')
    ->on('users.id', 'test.id');
```

```sql
 SELECT
`x_test`.*, `u`.`name` AS `u`
FROM `x_test` 
LEFT JOIN `x_users` AS `u`
    ON `x_users`.`id` = `x_test`.`id`
```

联合查询中的 `on` 方法工作原理与 `where` 完全相同，只是将提供的值作为标识符而不是参数值。把 `on`、`andOn` 和 `orOn` 方法串联起来可以创建更加复杂的连接。

```php
$select->leftJoin('users')
    ->on('users.id', 'test.id')
    ->orOn('users.id', 'test.balance');
```

数组形式的 where 条件也同样支持：

```sql
$select->leftJoin('users', 'u')->on([
    '@or' => [
        ['u.id' => 'test.id'],
        ['u.id' => 'test.balance']
    ]
]);
```

生成的 SQL:

```sql
SELECT
`primary_test`.*, `primary_users`.`name` as `user_name`
FROM `primary_test`  
LEFT JOIN `primary_users`
    ON (`primary_users`.`id` = `primary_test`.`id` OR `primary_users`.`id` = `primary_test`.`balance`)    
```

#### On Where 语句

要在 ON 语句中包含用户输入值，可以使用 `onWhere`, `orOnWhere` 和 `andOnWhere`:

```php
$select
    ->innerJoin('users')
    ->on(['users.id' => 'test.id'])
    ->onWhere('users.name', 'Anton');
```

```sql
SELECT
`primary_test`.*, `primary_users`.`name` as `user_name`
FROM `primary_test`  
INNER JOIN `primary_users`
    ON `primary_users`.`id` = `primary_test`.`id` AND `primary_users`.`name` = 'Anton'
```

#### 别名

联合查询方法中的第二个参数用于定义表的别名，可以在查询的 `on` 和 `where` 语句中使用：
```php
$select = $db->table('test')
    ->select(['test.*', 'uu.name as user_name'])
    ->innerJoin('users', 'uu')
    ->onWhere('uu.name', 'Anton');
```

第二种写法：

```php
$select = $db->table('test')
    ->select(['test.*', 'uu.name as user_name'])
    ->innerJoin('users as uu')
    ->onWhere('uu.name', 'Anton');
```

```sql
SELECT
`primary_test`.*, `uu`.`name` as `user_name`
FROM `primary_test`  
INNER JOIN `primary_users` as `uu`
    ON `uu`.`id` = `primary_test`.`id` AND `uu`.`name` = 'Anton'       
```

### 排序

使用 `orderBy` 来指定结果排序：

```php
// 我们的查询中有联合查询，因此表名是必须的。
$select
    ->orderBy('test.id', SelectQuery::SORT_DESC);
```

组合使用多个 `orderBy` 是可以的：

```php
$select
    ->orderBy(
        'test.name', SelectQuery::SORT_DESC
    )->orderBy(
        'test.id', SelectQuery::SORT_ASC
    );
```

另一种写法：

```php
$select
    ->orderBy([
        'test.name' => SelectQuery::SORT_DESC,
        'test.id'   => SelectQuery::SORT_ASC
    ]); 
```

两种写法生成的 SQL 都一样：

```sql
SELECT
`primary_test`.*, `uu`.`name` as `user_name`
FROM `primary_test`  
INNER JOIN `primary_users` as `uu`
    ON `uu`.`id` = `primary_test`.`id` AND `uu`.`name` = 'Anton' 
ORDER BY `primary_test`.`name` DESC, `primary_test`.`id` ASC
```

> 你也可以使用片段（Fragment）代替排序标识符（默认情况下排序标识符会作为字段名或者表达式来处理）。

### 分组和去重

如果你希望对查询结果进行去重，可以使用 `distinct` 方法（在 ORM 中加载 HAS_MANY/MANY_TO_MANY 关系时总是使用 `distinct`）。

```php
$select->distinct();
```

结果分组的操作通过 `groupBy` 方法来实现：

```php
$select = $db->table('test')
    ->select(['status', 'count(*) as count'])
    ->groupBy('status');
```

上面的代码会生成以下 SQL 语句：

```sql
SELECT
`status`, count(*) as `count`
FROM `primary_test`
GROUP BY `status`
```

### 聚合和计数
由于可以用 COUNT 和其它聚合函数来操作查询的字段，所以你构造的查询看起来可能会像这样：

```php
$select = $db->table('test')->select(['COUNT(*)']);
```

然而，在很多情况下你可能想直接获得计数或者汇总结果但不要进行列操作。这种情况下可以使用 `count`、`avg`、`sum`、`max` 和 `min` 方法：

```php
$select = $db->table('test')
    ->select(['id', 'name', 'status']);

dump($select->count());
dump($select->sum('balance'));
```

```sql
SELECT
COUNT(*)
FROM `primary_test`;

SELECT
SUM(`balance`)
FROM `primary_test`;
```

### 分页

要对查询结果进行分页，可以使用 `limit` 和 `offset` 方法：

```php
$select = $db->table('test')
    ->select(['id', 'name', 'status'])
    ->limit(10)
    ->offset(1);

foreach ($select as $row) {
    dump($row);
}
```

## 更新查询构造器

使用数据库连接实例或者表实例的 `update` 方法来获得更新查询构造器，然后调用该构造器的 `run` 方法对查询结果进行更新操作：


```php
$update = $db->table('test')
    ->update(['name' => 'Abc'])
    ->where('id', '<', 10)
    ->run();
```

```sql
UPDATE `primary_test`
SET `name` = 'Abc'
WHERE `id` < 10
```

可以使用 `Expression` 和 `Fragment` 的实例作为更新的值：

```php
$update = $db->table('test')
    ->update([
        'name' => new Expression('UPPER(test.name)')
    ])
    ->where('id', '<', 10)
    ->run();
```

```sql
UPDATE `primary_test`
SET `name` = UPPER(`primary_test`.`name`)
WHERE `id` < 10
```

嵌套查询在更新操作中同样支持：

```php
$update = $db->table('test')
    ->update([
        'name' => $database
            ->table('users')
            ->select('name')
            ->where('id', 1)
    ])
    ->where('id', '<', 10)->run();
```

```sql
UPDATE `primary_test`
SET `name` = (SELECT
`name`
FROM `primary_users`
WHERE `id` = 1)
WHERE `id` < 10 
```

> 更新查询中的 where 方法与 Select 查询中的 where 方法工作原理完全一致。

## 删除查询构造器

删除查询通过 `delete` 方法获取：

```php
$db->table('test')
    ->delete()
    ->where('id', '<', 1000)
    ->run();
```

你也可以在数据表实例的 `delete` 方法中直接以数组形式提供 where 条件：

```php
$db->table('test')
    ->delete([
        'id' => ['>' => 1000]
    ])
    ->run();
```

## 复杂表达式

你可以使用 Expression 对象来创建复杂的、特定于某种数据库的、包含参数的 SQL 语句注入查询中：

```php
$db->table('test')
    ->select()
    ->where(new Expression('SUM(column) = ?', $value))
    ->run();
```
