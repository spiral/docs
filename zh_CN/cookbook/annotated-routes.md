# 速查手册 - 注解式路由

Spiral 框架当前的版本暂时没有默认支持类似 Symfony 那样的注解式路由，但 Spiral 提供了相关的组件。如果开发者希望使用这一特性，可以在项目中引入 `spiral/annotated` 和 `spiral/router` 组件，即可构建特定于业务的路由逻辑。

> 注解式路由在应用的引导阶段会占用一点时间，但不会影响运行时的性能（得益于混合运行时提供的核心组件常驻内存）

## 路由注解
首先需要创建简单的注解，稍后将把该注解附加到控制器的 `public` 方法上：

```php
namespace App\Annotation;

use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation()
 * @Annotation\Target({"METHOD"})
 * @Annotation\Attributes({
 *      @Annotation\Attribute("action", type="string", required=true),
 *      @Annotation\Attribute("verbs", type="array"),
 * })
 */
class Route
{
    /** @var string */
    public $action;

    /** @var string[]|null */
    public $verbs;
}
```

> 官方的 Web 应用项目模板默认已经启用了注解支持。

定义好注解之后，就可以在控制器中如下面的示例这样使用该注解了：

```php
namespace App\Controller;

use App\Annotation\Route;

class DemoController
{
    /**
     * @Route(action="/hello", verbs={"GET"})
     */
    public function hello()
    {
        return 'hello world';
    }
}
```

> 此处的 action 参数支持使用路由匹配模式和路由参数，与[路由文档](/zh_CN/http/routing.md)中一致。

## 引导程序

为了让应用能够识别注解式路由，需要在 `RoutesBootloader` 引导程序中把路由注解转换为路由定义。可以通过 `Spiral\Annotations\AnnotationLocator` 这个类来查找项目代码中的可用注解：

```php
namespace App\Bootloader;

use Spiral\Annotations\AnnotationLocator;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(AnnotationLocator $annotationLocator, RouterInterface $router): void
    {
        $methods = $annotationLocator->findMethods(\App\Annotation\Route::class);

        foreach ($methods as $method) {
            $name = sprintf(
                "%s.%s",
                $method->getClass()->getShortName(),
                $method->getMethod()->getShortName()
            );

            $route = new Route(
                $method->getAnnotation()->action,
                new Action(
                    $method->getClass()->getName(),
                    $method->getMethod()->getName()
                )
            );

            $route = $route->withVerbs(...(array)$method->getAnnotation()->verbs);

            $router->setRoute($name, $route);
        }
    }
}
```

通过控制台命令的 `route:list` 可以列出所有可用的路由，来看定义的注解式路由是否生效：

```bash
$ php app.php route:list
```

> 可以使用额外的路由参数来配置中间件、通用前缀等等。
