# Framework — Config Objects

The Spiral framework exposes all the underlying configuration of its components using config objects. The core purpose
of config objects is to separate the bootload and runtime phase and provide an easily accessible source of the
configuration.

![Application Control Phases](https://user-images.githubusercontent.com/67324318/186413037-e60f89fd-9313-44c5-b4f8-eb2585a77230.png)

While configuration can change during the bootload process, at runtime all the values are frozen and forbidden for
modification.

## Configuration Provider

All of the configuration values can be accessed via `Spiral\Config\ConfiguratorInterface` in the form of an array.

To demonstrate that, we can create a config file `app/config/app.php`:

```php
<?php

declare(strict_types=1);

return [
    'values' => [123]
];
```

> **Note**
> You can use `json` format as well, or extend `ConfiguratorInterface` to add custom config readers.

To access the configuration values in your service or controller:

```php
use App\Config\AppConfig;
use Spiral\Config\ConfiguratorInterface;

// ...

public function index(ConfiguratorInterface $configurator)
{
    dump($configurator->getConfig(AppConfig::CONFIG));
}
```

> **Note**
> You can check if configuration exists using method `exists`. 

## Config Object

It is not very convenient to read the configuration in the form of arrays. The framework provides the OOP abstraction to
read your values. We can create this class manually or automatically generate it via `spiral/scaffolder`:

```bash
php app.php create:config app -r
``` 

> **Note**
> Use option `-r` to reverse engineer the configuration structure.

The resulted config class located in `app/src/config/AppConfig.php`:

```php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    protected array $config = [
        'values' => []
    ];

    /** @return int[] */
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

> **Note**
> The config object provides read-only API, changing values at runtime is not possible to prevent unwanted side-effect
> in long-running applications.

Every Spiral component provides the config object you can use in your application.

## Default Configuration in Bootloader

In many cases, the default configuration might be enough for most of the applications. Use a custom bootloader to define
default configuration values to avoid the need to create unnecessary files. Environment variables can be used as default 
values.

```php
namespace App\Bootloader;

use App\Config\AppConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;

class AppBootloader extends Bootloader
{
    public function init(ConfiguratorInterface $configurator, EnvironmentInterface $env): void
    {
        $configurator->setDefaults(AppConfig::CONFIG, [
            'values' => [432],
            'other' => $env->get('VALUE_FROM_ENV', 'default')
        ]);
    }
}
```

The file `app/config/app.php` will overwrite default configuration values. Remove this file to use the default
configuration.

> **Note**
> The overwrite is done on the first level keys of configuration array.

## Auto-Configuration

Some components will expose auto-configuration API to change its settings during the application bootload time. Usually,
such API is available through the component bootloader.

> **Note**
> For example `HttpBootloader`->`addMiddleware`.

We can provide our auto-configuration API in our Bootloader. Use `ConfiguratorInterface`->`modify` for this purpose.
Our Bootloader will be declared as Singleton to speed up processing a bit.

```php
namespace App\Bootloader;

use App\Config\AppConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Append;
use Spiral\Core\Container\SingletonInterface;

class AppBootloader extends Bootloader implements SingletonInterface
{
    public function __construct(
        private ConfiguratorInterface $configurator
    ) {
    }

    public function init(): void
    {
        $this->configurator->setDefaults(AppConfig::CONFIG, [
            'values' => [432]
        ]);
    }

    public function addValue(int $value): void
    {
        // append new value to the values section of app config
        $this->configurator->modify(AppConfig::CONFIG, new Append('values', null, $value));
    }
}
```

Now, we can modify configuration via strict API from another bootloader:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class ValueBootloader extends Bootloader
{
    public function init(AppBootloader $app): void
    {
        $app->addValue(800);
    }
}
```

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

You will receive this exception `Spiral\Config\Exception\ConfigDeliveredException`: *Unable to patch config `app`,
config object has already been delivered.*

It's recommended to set default values and change configs in the `init` method. And request a configuration file only in 
the `boot` method. The `init` method is called before `boot`. This will ensure that when the `boot` method is called, 
the configs will be in the correct state.
