# Framework — Config Objects

Spiral uses config objects to separate the bootload and runtime phases of an application, and to provide
an easily accessible source of configuration.

The main benefit of using config objects is that they allow for easy modification of configuration during the bootload
process, while ensuring that the configuration remains immutable at runtime. This helps to prevent unexpected changes to
the configuration and allows for more predictable behavior of the application.

![Application Control Phases](https://user-images.githubusercontent.com/67324318/186413037-e60f89fd-9313-44c5-b4f8-eb2585a77230.png)

**Here are some benefits of using config objects:**

- Config objects allow for easy modification of configuration values during the bootload process, making it simple to
  change the behavior of an application without modifying the code.
- Once the application has started, config objects freeze the values, preventing unexpected changes to the configuration
  at runtime. This helps to ensure that the application behaves as expected.
- Config objects centralize configuration information, making it easy to locate and modify as needed. This improves the
  overall organization and maintainability of the codebase.
- By keeping configuration information separate from the application's code, config objects help to improve the
  separation of concerns and make it easier to understand and maintain the codebase.

## Configuration Provider

Spiral provides a `Spiral\Config\ConfiguratorInterface` which allows for easy access to configuration values stored in 
config files. It provides methods to retrieve and check the existence of config values.

To demonstrate that, we can create a config file `app/config/github.php`:

```php app/config/github.php
<?php

return [
    'access_token' => 'xxx-xxxx',
    // ...
];
```

You can use any name you want, but it's recommended to use the name of the service that will use this config.

> **Note**
> You can use `json` format as well, or extend `ConfiguratorInterface` to add custom config readers.

### Using

To access the configuration values in your service, you can use the `ConfiguratorInterface` in the following way:

```php
use Spiral\Config\ConfiguratorInterface;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(ConfiguratorInterface $configurator)
    {
        if (!$configurator->exists('github')) {
            throw new \RuntimeException('Github configuration is missing');
        }
        
        $config = $configurator->get('github');
        $this->accessToken = $config['access_token'] ?? throw new \RuntimeException('Missing access token');
    }

    // ...
}
```

## Config Object

It is not very convenient to read the configuration in the form of arrays. The framework provides the OOP abstraction to
read your values. We can create this class manually or automatically generate it via `spiral/scaffolder`:

```terminal
php app.php create:config github -r
``` 

> **Note**
> `-r` option is used to reverse-engineer the configuration structure from the config file `app/config/github.php`.

The resulted config class located in `app/src/Config/GithubConfig.php`:

```php app/src/Config/GithubConfig.php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class GithubConfig extends InjectableConfig
{
    public const CONFIG = 'github';

    protected array $config = [
        'access_token' => '',
        // ...
    ];

    public function getAccessToken(): string
    {
        return $this->config['access_token'];
    }
}
``` 

The base class `Spiral\Core\InjectableConfig` allows you to request this object immediately in your code without any
IoC container configuration. The constant `CONFIG` contains the name of the configuration file.

> **Note**
> It's also possible to customize the generated class and add additional methods as needed.

```php
use App\Config\GithubConfig;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(GithubConfig $config)
    {
        $this->accessToken = $config->getAccessToken();
    }

    // ...
}
```

> **Note**
> The config object provides read-only API, changing values at runtime is not possible to prevent unwanted side-effect
> in long-running applications.

Every Spiral component provides the config object you can use in your application.

By using a config class, it's easy to access and manage configuration values in your Spiral application,
while also enjoying the benefits of using OOP abstraction and improved code organization.

## Default Configuration in Bootloader

Use a custom bootloader to define default configuration values to avoid the need to create unnecessary files.
Environment variables can be used as default values.

This can be useful in cases where the default configuration is enough for most of the applications, and to avoid the
need to create unnecessary config files.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;

final class GithubBootloader extends Bootloader
{
    public function init(ConfiguratorInterface $configurator, EnvironmentInterface $env): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }
}
```

This approach can be useful to set default configuration values that are common to all environments, or to provide a
fallback value if a specific environment variable is not set

If you want to use the default configuration, you can simply remove the `app/config/github.php` file.

> **Note**
> This feature allows to set default configuration values that can be easily overridden if needed, while still providing
> a fallback if no configuration file is present.

## Auto-Configuration

Some components will expose auto-configuration API to change its settings during the application bootload time. Usually,
such API is available through the component bootloader.

> **Note**
> For example `HttpBootloader`->`addMiddleware`.

We can provide our auto-configuration API in our Bootloader. Use `ConfiguratorInterface`->`modify` for this purpose.
Our Bootloader will be declared as Singleton to speed up processing a bit.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Set;
use Spiral\Core\Container\SingletonInterface;

class GithubBootloader extends Bootloader implements SingletonInterface
{
    public function __construct(
        private readonly ConfiguratorInterface $configurator
    ) {
    }

    public function init(): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN')
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }

    public function setAccessToken(string $token): void
    {
        $this->configurator->modify(
          GithubConfig::CONFIG, 
          new Set('access_token', $token)
        );
    }
}
```

Now, we can modify configuration via strict API from another bootloader:

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class SomeBootloader extends Bootloader
{
    public function init(GithubBootloader $github): void
    {
        $github->setAccessToken('xxx-xxxx');
    }
}
```

## Config Lifecycle

The framework provides a security mechanism to make sure that you are not changing config values after the config object
is requested by any of the components (a.k.a. config is frozen).

To demonstrate it, inject `GithubConfig` to `SomeBootloader`:

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;

class SomeBootloader extends Bootloader
{
    public function boot(GithubBootloader $github, GithubConfig $config): void
    {
        // forbidden
        $github->setAccessToken(800);
    }
}
```

You will receive this exception `Spiral\Config\Exception\ConfigDeliveredException`: *Unable to patch config `github`,
config object has already been delivered.*

It's recommended to set default values and change configs in the `init` method. And request a configuration file only in
the `boot` method. The `init` method is called before `boot`. This will ensure that when the `boot` method is called,
the configs will be in the correct state.
