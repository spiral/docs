# 框架 - 配置对象

Spiral 框架把所有底层配置选项都通过配置对象（config object）暴露出来。配置对象的核心设计目的分离引导阶段和运行阶段，并提供易于访问的配置源。

![引用程序控制阶段](https://user-images.githubusercontent.com/796136/64906478-e213ff80-d6ef-11e9-839e-95bac78ef147.png)

配置项可以在引导阶段进行调整，但是到了运行阶段所有的配置都被冻结，禁止修改。

## 配置提供程序

框架的所有配置项的值都可以借助 `Spiral\Config\ConfiguratorInterface` 以数组形式访问。

作为示例，我们首先创建一个配置文件 `app/config/app.php`:

```php
<?php

declare(strict_types=1);

return [
    'values' => [123]
];
```

> 示例中采用的是 PHP 文件和数组形式，你也可以采用 `json` 格式，或者通过实现 `ConfiguratorInterface` 来实现自己的配置读取器。

要在服务或者控制器代码中访问上面的配置项，可以这样做：

```php
use Spiral\Config\ConfiguratorInterface;

// ...

public function index(ConfiguratorInterface $configurator)
{
    dump($configurator->getConfig('app'));
}
```

> 如果需要检查配置文件是否存在，可以用 `exists` 函数。

## 配置对象

用数组形式来读取配置项相比之下不是特别方便。因此 Spiral 框架提供了面向对象的抽象来读取配置。你可以手工创建配置对象类，当然也可以通过 `spiral/scaffolder` 提供的命令来自动创建：

```php
$ php app.php create:config app -r
``` 

> 使用 `-r` 选项对配置文件结构进行反向工程。

上面的命令会创建 `app/src/config/AppConfig.php` 文件，代码如下：

```php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class AppConfig extends InjectableConfig
{

    public const CONFIG = 'app';

    protected $config = [
        'values' => []
    ];

    /**
     * @return array|int[]
     */
    public function getValues(): array
    {
        return $this->config['values'];
    }
}
``` 

基础类 `Spiral\Core\InjectableConfig` 让我们可以无需为 `AppConfig` 做任何 IoC 容器配置即可在代码中使用这个类。`CONFIG` 常量的值是配置文件的文件名。

```php
use App\Config\AppConfig;

// ...

public function index(AppConfig $appConfig)
{
    dump($appConfig->getValues());
}
```

> 配置对象提供只读 API, 因为在运行阶段不允许修改配置项的值，这是为了避免意外的副作用。

Spiral 官方的每个可以配置的框架都提供了配置对象供应用程序代码使用。

## 引导程序中的默认配置

大多数情况下，应用程序并不需要复杂的自定义配置，大多数的默认配置足可满足使用。同样的，你也可以在自定义引导程序中定义默认配置，减少不必要的配置文件。

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;

class AppBootloader extends Bootloader
{
    public function boot(ConfiguratorInterface $configurator): void
    {
        $configurator->setDefaults('app', [
            'values' => [432]
        ]);
    }
}
```

上述示例中，AppBooloader 定义了默认配置，而 `app/config/app.php` 中的配置会覆盖默认配置。如果该配置文件被删除，则继续使用默认配置。

> 配置是嵌套数组时，在第一级 key 上进行覆盖。

## 自动配置

部分组件会暴露自动配置 API 来允许在应用程序引导阶段修改它的配置。通常情况这类 API 都是通过组件的引导程序使用。

> 例如 `HttpBootloader`->`addMiddleware`.


创建自定义引导程序时，也可以实现自动配置 API. 通过 `ConfiguratorInterface` 接口定义的 `modify` 函数来实现。这种情况下，把引导程序定义为单例对象能够稍微提升速度。

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Append;
use Spiral\Core\Container\SingletonInterface;

class AppBootloader extends Bootloader implements SingletonInterface

    private $configurator;

    public function __construct(ConfiguratorInterface $configurator)
    {
        $this->configurator = $configurator;
    }

    public function boot(): void
    {
        $this->configurator->setDefaults('app', [
            'values' => [432]
        ]);
    }

    public function addValue(int $value)
    {
        // append new value to the values section of app config
        $this->configurator->modify('app', new Append('values', null, $value));
    }
}
```

这样既可从另外一个引导程序中借助该 API 来进行配置的修改：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class ValueBootloader extends Bootloader
{
    public function boot(AppBootloader $app): void
    {
        $app->addValue(800);
    }
}
```

> 需要保证调用的引导程序在 `AppBootloader` 之后执行，或者借助 `DEPENDENCIES` 常量来实现。

## 配置生命周期

Spiral 框架提供了安全机制，以确保配置对象一旦被某个组件依赖或使用，就不能在改变它的配置值（也即之前提到的配置冻结）。

举个例子，将 `AppConfig` 注入 `ValueBootloader`:

```php
namespace App\Bootloader;

use App\Config\AppConfig;
use Spiral\Boot\Bootloader\Bootloader;

class ValueBootloader extends Bootloader
{
    public function boot(AppBootloader $app, AppConfig $appConfig): void
    {
        // forbidden
        $app->addValue(800);
    }
}
```

You will receive an exception `Spiral\Config\Exception\ConfigDeliveredException`: *Unable to patch config `app`, 
config object has already been delivered.*
由于 `AppConfig` 已经被 `ValueBootloader` 依赖（自动初始化和注入），因此上面的代码会抛出 `Spiral\Config\Exception\ConfigDeliveredException` 异常："*Unable to patch config app, config object has already been delivered.*"
