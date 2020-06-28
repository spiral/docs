# 过滤器对象

过滤器对象用于对 PSR-7 或者任何其它的输入数据进行复杂的数据验证和筛选过滤。你可以手动创建过滤器，也可以使用脚手架提供的命令 `php app.php create:filter my`:

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA    = [
    ];

    protected const VALIDATES = [
    ];

    protected const SETTERS   = [
    ];
}
```

有了过滤器类之后，要创建具体的实例，可以借助 `Spiral\Filter\FilterProviderInterface` 接口的 `createFilter` 方法，或者采用最简单的手段——方法注入：

```php
use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter)
    {
        dump($filter);
    }
}
```

## 过滤器结构

任何一个过滤器对象的核心构成是 `SCHEMA`；这个常量定义了由输入对象提供的字段名和字段值之间的映射关系。
每个键值对都需要定义为 `field` => `source:origin` 或者 `filter` => `source` 的形式。**source** 用户输入数据的子集。在 HTTP 范围内，**source** 可以是 `cookie`、`data`、`query`、`input`（等于 `data` 加 `query`）、`header`、`file`、`server`. **origin** 是字段名称（支持点表示法）。

> 你可以使用 [InputManager](/zh_CN/http/request-response.md) 中的任何一个 input bag 作为数据源。

比如我们可以告诉过滤器把 `login` 字段指向查询参数 `username`：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'login' => 'query:username'
    ];
}
```

在过滤器中可以混合使用多个数据源：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'redirectTo'   => 'query:redirectURL',
        'memberCookie' => 'cookie:memberCookie',
        'username'     => 'data:username',
        'password'     => 'data:password',
        'rememberMe'   => 'data:rememberMe'
    ];
}
```

> 最常用的数据源是 `data`（指向 PSR-7 解析的主体），使用这个数据源可以从传入的 JSON 数据中取值。

### 点表示法

数据的 **origin** 可以用点表示法来指向嵌套结构内部的字段。例如：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'firstName' => 'data:name.first'
    ];

    protected const VALIDATES = [
        'firstName' => [
            ['notEmpty']
        ]
    ];

    protected const SETTERS = [
        'firstName' => 'strval'
    ];
}
```

上面的例子可以接受和验证下面的数据结构：

```json
{
  "names": {
    "first": "Antony"
  }
}
```

> 错误信息会正确地挂载到对应的字段位置。当然你可以混合使用不同的过滤器来实现更复杂的用例。

### 其它数据源

按照设计，你可以使用 `[InputManager](/zh_CN/http/request-response.md)` 的任何方法作为数据源，其中 **origin** 是传递参数。以下的数据源都是可用的：

| 数据源         | 说明                                                    |
| -------------- | ------------------------------------------------------- |
| uri            | 以 `Psr\Http\Message\UriInterface` 表示的当前请求的 URI |
| path           | 当前请求的路径部分                                      |
| method         | HTTP 方法 (GET, POST, ...)                              |
| isSecure       | 如果使用了 https                                        |
| isAjax         | 如果 `X-Requested-With` 设为 `xmlhttprequest`           |
| isJsonExpected | 如果客户端的 `Accept` 设为 `application/json`           |
| remoteAddress  | 用户的 IP 地址                                          |

> 请阅读 [InputManager 的文档](/zh_CN/http/request-response.md) 了解更多。

举个例子，如果需要检查用户请求是否通过 https 协议发起：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'httpsRequest' => 'isSecure'
    ];

    protected const VALIDATES = [
        'httpsRequest' => [
            ['notEmpty', 'error' => '不是安全连接']
        ]
    ];
}
```

> 有关于验证规则，下面的文档会进一步介绍。

### 路由参数

每个路由都会把匹配的参数写入到 `ServerRequestInterface` 的 `matches` 属性，因此在过滤器对象中也可以使用 `attribute:matches.{name}` 的表示法来访问路由参数：

```php
$router->setRoute(
    'sample',
    new Route('/action/<id>.html', new Controller(HomeController::class))
);
```

对于上面的路由，可以这样定义过滤器：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'routeID' => 'attribute:matches.id'
    ];
}
```

### 设置器（Setters）

在把输入的值传递给验证器之前，可以使用设置器对输入的值进行类型转换。如果在转换时抛出了类型转换错误，那么筛选器会把该字段的值设定为 `null`：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'number' => 'query:number'
    ];

    protected const VALIDATES = [
        'number' => [
            ['notEmpty'],
            ['number::higher', 5]
        ]
    ];

    protected const SETTERS = [
        'number' => 'intval'
    ];
}
```

> 可以使用任何 PHP 内置函数例如 `intval`、`strval` 等。

```php
namespace App\Controller;

use App\Filter\MyFilter;

class HomeController
{
    public function index(MyFilter $filter)
    {
        dump($filter->number); // 总是数字
    }
}
```

> 译注：验证用户输入时，请谨慎使用设置器，这会使输入验证规则失效，且筛选后的数据与用户输入不一致

## 验证

验证规则的定义与 [数据验证](/zh_CN/security/validation.md) 组件中的方法和规则一致。

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'name' => 'data:name'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty']
        ]
    ];
}
```

可以使用所有的检查器、条件和规则。

### 自定义错误

与安全组件中的数据验证一样，你可以给任何一条验证规则定义自己的错误信息。

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'name' => 'data:name'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty', 'error' => 'Name 不能为空']
        ]
    ];
}
```

如果想要在定义之后对错误消息进行本地化，可以把错误信息文字用 `[[]]` 包围起来，可以自动生成本地化翻译的索引并在显示时自动替换为对应的语言（在语言包中定义）：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'name' => 'data:name'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty', 'error' => '[[Name must not be empty]]']
        ]
    ];
}
```

## 使用

一旦配置好过滤器，就可以直接访问它的字段（过滤之后的数据），检查数据合法性并在发生错误时返回一组错误信息。

> 建议在数据到达控制器之前使用[领域核心拦截](/zh_CN/cookbook/domain-core.md)对过滤器的数据进行必要的验证。

### 获取字段

要访问经过过滤的数据字段，使用 `getField` 和 `getFileds` 方法。举个例子，假设有如下的过滤器：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'name'  => 'data:name',
        'email' => 'data:email'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty']
        ]
    ];

    protected const SETTERS = [

    ];
}
```

在数据存在且通过验证的情况下，以下字段可以访问：

```php
public function index(MyFilter $filter)
{
    dump($filter->getFields()); // {name: ..., email: ...}

    dump($filter->getField('name'));

    // 同上
    dump($filter->email);
}
```

### 获取错误

要检查过滤器的数据是否有效，可以使用 `isValid` 方法，无效字段的错误可以通过 `getErrors` 获取：

```php
public function index(MyFilter $filter)
{
    if (!$filter->isValid()) {
        dump($filter->getErrors());
    }
}
```

错误会自动映射到正确的原始属性名称，例如：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    protected const SCHEMA = [
        'name' => 'data:names.name.0'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty']
        ]
    ];

    protected const SETTERS = [

    ];
}
```

在 `name` 字段无效时，上面的过滤器会响应下面的错误：

```json
{
  "status": 400,
  "errors": {
    "names": {
      "name": "This value is required."
    }
  }
}
```

> 错误的格式和[数据验证文档](/zh_CN/security/validation.md)中描述的一致。

## 继承

你可以从一个过滤器扩展出另一个过滤器，`SCHEMA`、`VALIDATES` 和 `SETTERS` 常量会被继承：

```php
namespace App\Filter;

class MyFilter extends BaseFilter
{
    protected const SCHEMA = [
        'name' => 'data:name'
    ];

    protected const VALIDATES = [
        'name' => [
            ['notEmpty']
        ]
    ];
}
```

上面例子中的 `BaseFilter` 代码如下：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class BaseFilter extends Filter
{
    protected const SCHEMA = [
        'token' => 'data:token'
    ];

    protected const VALIDATES = [
        'token' => [
            ['notEmpty']
        ]
    ];
}
```

在上例中，`MyFilter` 同样要求 `token` 字段必须填写：

```json
{
  "status": 400,
  "errors": {
    "token": "This value is required.",
    "name": "This value is required."
  }
}
```
