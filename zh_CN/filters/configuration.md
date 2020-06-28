# 过滤器对象

Spiral 框架的 `spiral/filters` 组件为应用提供了请求验证、复合验证、错误信息映射和定位等能力。

> 此组件依赖 [安全/数据验证](/zh_CN/security/validation.md) 库，如果还未阅读该文档，请先阅读。

## 安装

筛选组件不需要进行任何配置，但需要通过 `Spiral\Bootloader\Security\FiltersBootloader` 引导程序来启用。

```php
[
    // ...
    Spiral\\Bootloader\Security\FiltersBootloader::class
    // ...
]
```

## 输入绑定

筛选组件使用 `Spiral\Filter\InputInterface` 作为主要数据源：

```php
interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

默认情况下，这个接口是绑定到 [InputManager](/zh_CN/http/request-response.md) 对象的，这让它可以访问任何使用 **source** 和 **origin** 对的请求属性，并支持点符号。例如：

```php
namespace App\Controller;

use Spiral\Filters\InputInterface;

class HomeController
{
    public function index(InputInterface $input)
    {
        dump($input->getValue('query', 'abc')); // ?abc=1

        // 点符号
        dump($input->getValue('query', 'a.b.c')); // ?a[b][c]=2

        // 同上
        dump($input->withPrefix('a')->getValue('query', 'b.c')); // ?a[b][c]=2
    }
}
```

输入绑定是把数据传递给筛选对象的主要方法。

## 创建过滤器

过滤器对象的具体实现可能因为使用包不同而有所差异。默认的实现提供了一个抽象类 `Spiral\Filter\Filter`。如果要创建一个自定义的过滤器来验证查询参数：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class MyFilter extends Filter
{
    public const SCHEMA = [
        'abc' => 'query:abc'
    ];

    public const VALIDATES = [
        'abc' => [
            'notEmpty',
            'string'
        ]
    ];
}
```

> 或者使用脚手架提供的控制台命令 `php app.php create:filter my -f "abc:string(query)"`，效果同上。

一旦创建，可以借助方法注入来获得过滤器的实例（该实例会自动与当前 http 输入绑定）：

```php
namespace App\Controller;

use App\Filter;

class HomeController
{
    public function index(Filter\MyFilter $f)
    {
        dump($f->isValid());
        dump($f->getErrors());
        dump($f->getFields());
    }
}
```

> 可以在 URL 后加上 `?abc=1` 来尝试上面的例子。

## 扩展

在你的[领域核心](/zh_CN/cookbook/domain-core.md)里启用 `Spiral\Domain\FilterInterceptor` 可以在把请求传递给控制器之前自动预验证。
