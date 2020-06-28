# 数据网格

使用 `spiral/data-grid` 和 `spiral/data-grid-bridge` 可以根据用户请求为 Cycle 和 DBAL 自动生成查询：

## 安装

首先安装组件：

```bash
$ composer require spiral/data-grid-bridge
```

然后在引用中启用 `Spiral\DataGrid\Boot\Loader\GridBootloader` 引导程序，注意这个引导程序要在 Database 和 Cycle 的引导程序之后。

```php
protected const LOAD = [
    // ...
    Spiral\DataGrid\Bootloader\GridBootloader::class
];
```

## 使用

要使用数据网格功能，需要使用两个基础抽象——网格工厂（grid factory）和网格模式（grid schema）。

### 网格模式

网格模式是用于描述如何根据用户输入来配置数据库选择器的对象。可以通过 `Spiral\DataGrid\GridSchema` 类来创建：

```php
use Spiral\DataGrid;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new DataGrid\GridSchema();

// 允许用户按照每页 10 个结果来进行分页
$schema->setPaginator(new PagePaginator(10));

// 允许用户根据 id 进行数据排序
$schema->addSorter('id', new Sorter('id'));

// 允许基于 name 筛选数据，值由用户提供
$schema->addFilter('name', new Like('name', new StringValue()));
```

> 开发者可以扩展 GridSchema 并且在扩展类的构造函数中初始化所有条件。

### 网格工厂

为了使用定义好的网格模式，必须先获取支持的数据源的实例。默认支持 Cycle 读取查询和 Database 读取查询。

```php
namespace App\Controller;

use App\Database\UserRepository;
use Spiral\DataGrid\GridFactory;
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

class HomeController
{
    public function index(UserRepository $users, GridFactory $factory)
    {
        $schema = new GridSchema();
        $schema->setPaginator(new PagePaginator(10));
        $schema->addSorter('id', new Sorter('id'));
        $schema->addFilter('name', new Like('name', new StringValue()));

        $result = $factory->create($users->select(), $schema);

        dump(iterator_to_array($result));
    }
}
```

这样就可以通过调用控制器来使用数据网格了。默认情况下，第一个访问的用户会从数据库查询（之后的用户会从缓存直接取数据）。

如果要获取第二页的数据，可以在 GET 或者 POST 的请求中传递需要的参数：`?paginate[page]=2`.

要对特定字段进行搜索，可以传递参数：`?filter[name]=antony`.

要按照 id 进行正序或倒序排列，可以传递参数：`?sort[id]=desc`.

## 可用的条件

数据网格有大量可用的条件：

### 查询条件

这个条件表示一组可用的表达式，在输入参数中传入值可以对结果应用单个或多个查询条件。

> 可以传入单个 key 或者多个 key. 但不要声明 ValueInterface.

单条件示例：

```php

use Spiral\DataGrid\Specification\Filter;

// 注意，这里使用的是整数型参数
$select = new Filter\Select([
    new Filter\Equals('name', 'value'),
    new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    new Filter\Equals('email', 'email@example.com'),
]);

$filter = $select->withValue(1); // 第二个筛选器
```
> 这种情况下筛选器等同于 `Filter\Any` 查询。

多条件示例：

```php

use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

$filter = $select->withValue(['one', 'two']);
```
> 这种情况下筛选器会同时包含两个查询条件，类似 `Filter\All` 查询。


传入未知值的示例：

```php

use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

$filter = $select->withValue('four');
```

> 这种情况下筛选器等同于 null.
