# 速查手册 - 注解式路由

Spiral 框架当前的版本暂时没有默认支持类似 Symfony 那样的注解式路由，但 Spiral 提供了相关的组件。如果开发者希望使用这一特性，可以在项目中引入 `spiral/annotated` 和 `spiral/router` 组件，即可构建特定于业务的路由逻辑。默认的 Web 项目模板中已经包含了 `spiral/router` 依赖，但还需要安装 `spiral/annotated`:

```bash
$ composer require spiral/annotated-routes
```

安装完成以后，在 `App.php` 中激活 `Spiral\Router\Bootloader\AnnotatedRoutesBootloader` 引导程序（可以禁用默认的应用级 `RoutesBootloader`）：

```php
protected const LOAD = [
    // 其它引导程序...
    Spiral\Router\Bootloader\AnnotatedRoutesBootloader::class
];
```

这样就能在项目中使用注解式路由扩展了。

## 定义路由

要定义一个路由定义，只需要在控制器方法上添加 `Spiral\Router\Annotation\Route` 注解：

```php
namespace App\Controller;

use Spiral\Router\Annotation\Route;

class HomeController
{
    /**
     * @Route(route="/", name="index", methods="GET")
     */
    public function index()
    {
        return 'hello world';
    }
}
```

以下是在路由注解中可用的属性：

属性 | 类型 | 说明
--- | --- | ---
route | string | 路由匹配，与 [Router](/zh_CN/http/routing.md) 的规则一致。必须提供
name | string | 路由名称，必须提供
methods | array/string | 支持的 HTTP 方法，默认是所有方法
defaults | array | 路由匹配中参数的默认值
group | string | 路由所属的组，默认是 `default`
middleware | array | 该路由要使用的中间件

## 路由组

注解式路由可以以路由分组的方式实现共享的规则、中间件、[领域核心](/zh_CN/cookbook/domain-core.md)、路径前缀等。如果要对路由分组，需要创建和注册引导程序来实现：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\GroupRegistry;

class APIRoutes extends Bootloader
{
    public function boot(GroupRegistry $groups)
    {
        $groups->getGroup('api')
               ->setPrefix('/api/v1')
               ->addMiddleware(SomeMiddelware::class);
    }
}
```

> 请确保这个引导程序的顺序要在 `AnnotatedRoutesBootloader` 之后。可以使用 `default` 路由组来配置所有路由。

完成上述工作之后，就可以对路由的注解使用 `group` 属性来进行分组了。

```php
/**
 * @Route(route="/", name="index", methods="GET", group="api")
 */
public function index(){
    // ...    
}
```

> 在分组时已经指定的中间件、前缀和领域核心会自动添加。

## 路由缓存

默认情况下，在 `DEBUG` 选项关闭时所有的注解式路由都会被缓存。如果要让路由缓存独立于 `DEBUG` 配置进行设定，可以使用 `ROUTE_CACHE` 环境变量：

```dotenv
DEBUG = true
ROUTE_CACHE=true
```

执行控制台命令 `route:reset` 可以重置路由缓存：

```bash
$ php app.php route:reset
```
