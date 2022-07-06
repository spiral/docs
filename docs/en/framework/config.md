# Framework - Config Objects

The Spiral framework exposes all the underlying configuration of its components using config objects. The core purpose
of config objects is to separate the bootload and runtime phase and provide an easily accessible source of the
configuration.

![Application Control Phases](https://user-images.githubusercontent.com/796136/64906478-e213ff80-d6ef-11e9-839e-95bac78ef147.png)

While configuration can change during the bootload process, at runtime all the values are frozen and forbidden for
modification.

## Configuration Provider

All of the configuration values can be accessed via `Spiral\Config\ConfiguratorInterface` in the form of an array.

To demonstrate that we can create config file `app/config/app.php`:

```php
<?php

declare(strict_types=1);

return [
    'values' => [123]
];
```

> You can use `json` format as well, or extend `ConfiguratorInterface` to add custom config readers.

To access the configuration values in your service or controller:

```php
use Spiral\Config\ConfiguratorInterface;

// ...

public function index(ConfiguratorInterface $configurator)
{
    \dump($configurator->getConfig('app'));
}
```

> You can check if configuration exists using method `exists`.

## Config Object

It is not very convenient to read the configuration in the form of arrays. The framework provides the OOP abstraction to
read your values. We can create this class manually or automatically generate it via `spiral/scaffolder`:

```bash
$ php app.php create:config app -r
``` 

> Use option `-r` to reverse engineer the configuration structure.

The resulted config class located in `app/src/config/AppConfig.php`:

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

The base class `Spiral\Core\InjectableConfig` allows you to request this object immediately in your code without any
IoC container configuration. The constant `CONFIG` contains the name of the configuration file.

```php
use App\Config\AppConfig;

// ...

public function index(AppConfig $appConfig)
{
    \dump($appConfig->getValues());
}
```

> The config object provides read-only API, changing values at runtime is not possible to prevent unwanted side-effect
> in long-running applications.

Every Spiral component provides the config object you can use in your application.

## Default Configuration in Bootloader

In many cases, the default configuration might be enough for most of the applications. Use custom bootloader to define
default configuration values to avoid the need to create unnecessary files.

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

The file `app/config/app.php` will overwrite default configuration values. Remove this file to use the default
configuration.

> The overwrite is done on the first level keys of configuration array.

## Auto-Configuration

Some components will expose auto-configuration API to change its settings during the application bootload time. Usually,
such API is available through the component bootloader.

> For example `HttpBootloader`->`addMiddleware`.

We can provide our auto-configuration API in our Bootloader. Use `ConfiguratorInterface`->`modify` for this purpose.
Our Bootloader will be declared as Singleton to speed up processing a bit.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Append;
use Spiral\Core\Container\SingletonInterface;

class AppBootloader extends Bootloader implements SingletonInterface
{
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

Now, we can modify configuration via strict API from another bootloader:

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

> Make sure to locate the Bootloader after the `AppBootloader` or use `DEPENDENCIES` constant.

## Config Lifecycle

The framework provides a security mechanism to make sure that you are not changing config values after the config object
is requested by any of the components (a.k.a. config is frozen).

To demonstrate it, inject `AppConfig` to `ValueBootloader`:

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
