# 框架 - 容器和工厂

Spiral 框架包含一系列的接口和组件，旨在简化应用程序中的依赖注入管理和对象构造。

> 容器的实现 与 [PSR-11 容器规范](https://github.com/php-fig/container) 完全兼容。

## PSR-11 容器

在程序代码的任意地方，你随时可以通过 `Psr\Container\ContainerInterface` 的依赖来访问容器，比如在控制器中：

```php
use Psr\Container\ContainerInterface;

class HomeContoller
{
    public function index(ContainerInterface $container)
    {
        dump($container->get(App\App::class));
    }
}
```

## 依赖注入

Spiral 支持在类构造函数以及其它方法实现自动注入依赖项：

```php
class UserMailer
{
    protected $mailer = null;

    public function __construct(Mailer $mailer)
    {
        $this->mailer = $mailer;
    }
    
    public function do()
    {
        $this->mailer->sendMail(...);
    }
}
```

在上面的代码中，`UserMailer` 类的依赖对象 `Mailer` 会被容器自动传递给构造函数（自动装配）。
 
> 提示，控制器类、命令类和任务类都支持方法注入。

## 容器配置
 
你可以通过在别名或接口与其具体实现之间创建绑定关系，来配置容器。

在 Spiral 中，通过 [引导程序（Bootloader）](/framework/bootloaders.md) 来定义上述绑定关系。

要配置应用容器，可以使用 `Spiral\Core\BinderInterface` 或者 `Spiral\Core\Container` 的任意一种（后者是前者的具体实现）。

例如，要把一个接口与其具体实现进行绑定，代码如下：

```php
public function boot(Container $container)
{
    $container->bind(MyInterface::class, MyImplementation::class);
}
```

单例模式的绑定：

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, MyImplementation::class);
}
```

如果要绑定的对象需要指定参数，方法如下：

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, bind(MyImplementation::class, [
        'param' => 'value'
    ]));
}
```

> 要了解如何管理依赖项的配置，请参考 [配置对象（Config Objects）](/framework/config.md)。

当然，也可以通过`闭包函数（closure）`来自动配置你的实现类：

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, function() {
        return new MyImplementation('some-value');
    });
}
```

闭包函数本身也支持自动依赖注入，如以下示例，`$class` 参数会在 `MyImplementation` 被依赖时，由容器自动注入。

```php
public function boot(Container $container)
{
    $container->bindSingleton(MyImplementation::class, function(SomeClass $class) {
        return new MyImplementation($class);
    });
}
```

可以通过容器对象的 `has` 方法检查是否包含某个类/接口的绑定：

```php
dump($container->has(MyImplementation::class));
```

要从容器中移除已有的绑定，可以通过 `removeBinding` 方法：

```php
$container->removeBinding(MyService::class);
```

## 单例延迟加载

如果你的类希望通过单例模式使用，可以实现 `Spiral\Core\Container\SingletonInterface` 接口，这样就无需进行单例绑定：

```php
use Spiral\Core\Container\SingletonInterface;

class MyService implements SingletonInterface
{
    public function method()
    {
        //...
    }
}
```

实现了上述的接口之后，容器会自动以单例模式处理这个类：

```php
protected function index(MyService $service)
{
    dump($this->container->get(MyService::class) === $service);
}
```

## 工厂接口
某些场景下，你可能希望构造所需的类，但不要自动解析它的构造函数所的依赖对象。这种情况可以通过 `Spiral\Core\FactoryInterface` 来实现：

```php
public function makeClass(FactoryInterface $factory)
{
    return $factory->make(MyClass::class, [
        'parameter' => 'value'
        // 除了明确指定的参数以外，其它依赖会自动解析
    ]); 
}
```

## 解析接口

除了明确指明依赖项的类型以外，有时候在调用一个方法时并不确定它的参数类型（比如在抽象类中调用具体实现类的方法），这种情况下，可以借助 `Spiral\Core\ResolverInterface` 来实现：

```php
abstract class Handler
{
    /** @var ResolverInterface */
    protected $resolver;

    public function __construct(ResolverInterface $resolver)
    {
        $this->resolver = $resolver;
    }

    public function run(array $params)
    {
        $method = new \ReflectionMethod($this, 'do'); // 要调用的包含自动注入参数的方法
        $method->setAccessible(true);

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // 解析所需的参数自动注入
        );
    }
}
```

现在 `do` 方法就可以实现自动依赖注入了：

```php
class MyHandler extends Handler
{
    public function do(SomeClass $some)
    {
        // ...    
    }
}
```

## 自动装配

Spiral 框架通过提供丰富的自动装配功能，尽量对业务层代码隐藏容器的实现和配置。虽然自动装配的规则非常简单，但掌握他们还是很重要的，这样才能避免框架使用中出现一些异常行为。

### 依赖项自动解析

框架容器能够自动提供具体类的实例来解决构造函数及其它方法函数的依赖关系。例如：

```php
class MyController
{
    public function __construct(OtherClass $class, SomeInterface $some)
    {
    }
}
```

在上面的示例中，容器将尝试自动构造 `OtherClass` 的实例来解决对它的依赖，但是对于 `SomeInterface` 接口，容器并不清楚要用它的哪一个具体实现来解决依赖，因此需要明确建立接口与要使用的具体实现类之间的绑定关系：

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

需要特别注意的是，容器会尝试解决构造函数的 *所有* 依赖（除非你明确提供了某个依赖的具体值）。这意味着所有的类构造函数依赖参数都必须是可用的，否则的话，该参数需要声明为可选参数：

```php
// 如果在实例化该类时没有明确传入 $value 参数，会报错
__construct(OtherClass $class, $value)

// 如果在实例化该类时没有传入 `$value` 参数，会使用 `null`
__construct(OtherClass $class, $value = null) 

// 如果 SomeInterface 没有绑定到具体实现，且构造时没有传入，会报错
__construct(OtherClass $class, SomeInterface $some) 

// 如果构造时没有提供 SomeInterface 的具体实现，会自动使用 `null`
__construct(OtherClass $class, SomeInterface $some = null) 
```

### 上下文自动装配

除了常规的方法注入以外，容器还能自动解析注入依赖上下文。具体来说，这个能力使容器能够在一个方法函数中自动识别同一个类的两个不同实现：

```php
protected function index(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}
```

> 上面例子中的 `primary` 和 `secondary` 是数据库连接的名称，参见[数据库配置](/database/configuration.md)。


如果要实现自己的依赖注入器，需要实现 `Spiral\Core\Container\InjectorInterface`：

下面简单通过示例说明如何创建一个依赖注入器：

```php
class Injector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        return new MyClass($context);
    }
}
```

上面创建的依赖注入器里用到的 `MyClass` 类如下：

```php
class MyClass 
{
    public function __construct(string $name)
    {
        //...
    }
}
```

构造完成后，要在容器中注册注入器：

```php
$container->bindInjector(MyClass::class, Injector::class);
```

现在我们就可以像下面这样使用 `MyClass` 作为方法参数了：

```php
public function method(MyClass $john, MyClass $bob)
{
    dump($john);
    dump($bob);
}
```

在使用 `Spiral\Core\FactoryInterface` 时，可以始终绕过上下文注入：


```php
dump($factory->make(MyClass::class, ['name' => 'abc']));
```

## 使用单例模式

许多内部应用服务以单例对象的形式驻留在内存中。这些对象不是通过实现静态方法 `getInstance`, 而是被配置为在多个请求时间保留和复用。

把服务或者控制器声明为单例是获得少许性能改善的捷径，但是在使用时必须要注意一些规则，以避免内存和状态泄露。

### 定义单例

Spiral 框架提供了很多方法可以把类对象声明为单例。首先，可以通过创建引导程序，在其中把类和它自己的类名绑定：

```php
class ServiceBootloader extends Bootloader
{
    protected const SIGNLETONS = [
        Service::class => Service::class
    ];
}
```

如此一来，在注入依赖时，你通过依赖容器请求这个类时得到的始终都是同一个实例。

另一种方法不需要引导程序，可以在类的代码中中进行定义，这种情况需要实现 `Spiral\Core\Container\SingletonInterface` 接口。实现这个接口等于告诉容器”这个类只实例化一次“：

```php
use piral\Core\Container\SingletonInterface;

class Service implements SingletonInterface
{
    // ...
}
``` 

> 单例接口可以由服务类、控制器类、中间件类等实现。但不要在数据仓库和关系映射中实现，因为后者的状态由 ORM 进行管理。

### 使用限制

把服务保留在内容中实现在多个请求之间复用，可以避免一次又一次地执行复杂的初始化和计算过程。但必须时刻铭记这样的服务必须以 **无状态** 的方式设计，并且不包含任何用户数据。

以下是严格禁止的：
- 在单例中保留用户信息
- 保留对 PSR-7 请求对象的引用（使用 `InputManager` 代替）
- 保留对 session 的引用（使用 `SessionScope` 代替）
- 保留 RBAC 身份（使用 `GuardScope` 代替）

### 预热

多个用户请求之间不会发生改变的数据可以保留在服务中。比如，应用程序依赖于庞大的 XML 作为配置源。


```php
class Service implements SingletonService 
{
    private $configCache;


    public function getConfig(): array
    {
        if ($this->configCache !== null) {
            return $this->configCache;
        }
    
        $this->configCache = $this->readConfig(); // 复杂的计算或者磁盘读取
    
        return $this->configCache;
    }
}   
```

通过上面代码演示的实现方式，就可以只做一次复杂的计算并保留在内存中，在之后的用户请求中就只从内存中获取这些信息。
