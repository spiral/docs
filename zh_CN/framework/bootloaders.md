# 框架 - 引导程序

引导程序（bootloader）是 Spiral 框架以及基于 Spiral 框架构建的程序的核心部分。各种引导程序负责配置[容器](/frameworks/container.md)、配置组件等。

引导程序只在应用程序初始化时执行一次。由于应用程序会在内存里保持长期运行，因此可以根据需要添加很多引导程序而不会造成性能的下降。

![应用控制阶段](https://user-images.githubusercontent.com/796136/64906478-e213ff80-d6ef-11e9-839e-95bac78ef147.png)

## 简单的引导程序

开发者可以通过扩展 `Spiral\Boot\Bootloader\Bootloader` 类快速创建一个简单的引导程序：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class MyBootloader extends Bootloader 
{

}
```

要使用的每个引导程序都必须在应用内核中激活。要激活引导程序，可以把引导程序类添加到 `App\App` 类的 `LOAD` 或者 `APP` 列表中。

```php
class App extends Kernel
{
    protected const LOAD = [
        // ...
    ];

    protected const APP = [
        RoutesBootloader::class,
        LoggingBootloader::class,
        MyBootloader::class
    ];
}
```

上面示例中的引导程序还什么功能都没有。可以给它添加一些容器绑定。

> 添加到 `APP` 列表的引导程序会在 `LOAD` 列表中的引导程序之后引导，因此通常在其中放一些业务专用的引导程序。

## 配置容器

引导程序最普遍的用途是配置依赖注入容器，比如需要把某些具体实现类与其实现的接口进行绑定，或者需要初始化某些服务。

引导程序类的 `boot` 方法可以实现上述目的。这个方法本身也支持方法注入，因此可以把需要的服务作为该方法的参数：

```php
use Spiral\Core\Container;

class MyBootloader extends Bootloader 
{
    public function boot(Container $container) 
    {
        $container->bind(MyInterface::class, MyClass::class);
        
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

引导程序类提供了 `BINDINGS` 和 `SINGLETONS` 可以简化容器绑定操作： 

```php
use Spiral\Core\Container;

class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    public function boot(Container $container) 
    {
        $container->bindSingleton(MyService::class, function(MyClass $myClass) {
            return new MyService($myClass); 
        });
    }
}
```

上面示例中创建 `MyService` 单例的闭包工厂函数（factory closure）可以用工厂方法替代（factory method）：

```php
class MyBootloader extends Bootloader 
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];

    const SINGLETONS = [
        MyService::class => [self::class, 'myService']
    ];

    public function myService(MyClass $myClass): MyService
    {
        return new MyService($myClass); 
    }
}
```

## 配置应用程序

引导程序的另一个用途是在应用程序启动之前对框架进行配置。例如为应用程序或者模块声明一个新的路由：

```php
class MyBootloader extends Bootloader 
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> 与此类似，你也可以用引导程序加载中间件、更改令牌生成器目录等。

## 引导程序间的依赖

某些框架引导程序可以作为配置系统设置的便捷手段。例如，可以通过 `Spiral\Bootloader\Http\HttpBootloader` 添加一个遵循 PSR-15 的全局中间件：


```php
class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

如果 `HttpBootloader` 必须在 `MyBootloader` 之前完成初始化，可以借助 `DEPENDENCIES` 常量来实现：

```php
class MyBootloader extends Bootloader 
{
    const DEPENDENCIES = [
        HttpBootloader::class
    ];

    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

> 注意，只有在引导程阶段才能通过引导程序对组件进行配置。因此要对一个引导程序进行配置的话，只能通过另一个引导程序来实现。Spiral 框架不允许在组件初始化之后改变组件的任何配置项。

## 层叠引导

通过 `Spiral\Boot\BootloadManager`, 引导程序还能控制其自身的引导过程：

```php
namespace App\Bootloader;


use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\BootloadManager;
use Spiral\Bootloader\DebugBootloader;

class AppBootloader extends Bootloader
{
    public function boot(BootloadManager $bootloadManager)
    {
        if (env('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class
            ]);
        }
    }
}
```
