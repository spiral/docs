# 数据库 - 访问数据

首先按照[这里](/zh_CN/database/configuration.md)的配置指引进行数据库配置。

## 访问数据

一旦正确配置好 DBAL 组件，就可以在控制器或者服务中以多种方式进行数据访问了。

```php
namespace App\Controller;

use Spiral\Database\DatabaseManager;

class HomeController 
{
    public function index(DatabaseManager $dbal)
    {
        // 默认数据库
        dump($dbal->database());
        
        // 通过快捷属性访问数据库
        dump($this->db);
    
        // 使用指向主数据库的 default 别名
        dump($dbal->database('default'));
    
        // 从数据库
        dump($dbal->database('slave'));
    
        // 快捷属性 + 数据库名称
        dump($this->dbal->database('secondary'));
    }
}
```

### 方法和构造函数注入

DBAL 组件完全支持基于数据库连接名称和别名的 [IoC 注入](/zh_CN/framework/container.md)。

```php
public function index(Database $database, Database $primary, Database $slave)
{
    // 不绑定名称的连接是 `primary` 的别名
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

### 开发原型


可以使用 `PrototypeTrait` 访问 `Spiral\Database\DatabaseProviderInterface` 和默认的数据库连接实例：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->dbal);
        dump($this->db); // 默认数据库连接
    }
}
```

## 执行查询

要执行数据库查询，可以使用 `query` 方法：

```php
dump(
    $db->query(
        'SELECT * FROM users WHERE id > ?',
        [
            1
        ]
    )->fetchAll()
);
```

要执行 update 或者 delete 语句，可以使用替代方法 `execute`:

```php
dump(
    $db->execute(
        'DELETE FROM users WHERE id > ?',
        [
            1
        ]
    ) // 返回受影响的行数
);
```

> 要了解更多有关查询构造的资料，请阅读[相关文档](/zh_CN/database/query-builders.md)。
