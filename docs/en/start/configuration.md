# Configuration

All of the provided application skeletons are already pre-configured using optimal settings. You can edit any of the
settings by editing the file(s) in `app/config/`.

> **Note**
> If config file does not exist - create it using `<?php return [];` as a base.

## Environment

[Web](https://github.com/spiral/app) and [GRPC](https://github.com/spiral/app-grpc) templates 
use [DotEnv](../extension/dotenv.md) extension to read environment values from `.env` file located in the root of your
project.

```env
# Debug mode set to TRUE disables view caching and enables higher verbosity.
DEBUG=true

# Set to application-specific value, used to encrypt/decrypt cookies, etc.
ENCRYPTER_KEY={encrypt-key}

# Set to TRUE to disable confirmation in `migrate` commands.
SAFE_MIGRATIONS=true
```

You can access these values using `Spiral\Boot\EnvironmentInterface` 

```php
public function index(EnvironmentInterface $env): void
{
    dump($env->get('ENCRYPTER_KEY'));
}
```

or via a short function `env`

```php
public function index(): void
{
    dump(env('ENCRYPTER_KEY'));
}
```

> **Note**
> Any variable in your `.env` file can be overridden by external environment variables such as server-level or 
> system-level environment variables.

## Configuration

The default component configuration located inside the related Bootloader. You can alter such configuration using other
bootloaders (see Auto-Configuration) or by creating a *default configuration* file in `app/config`.

> **Note**
> Each of the documentation sections will include the content of the default component configuration.

Web and GRPC skeletons include `app/config/database.php` config file:

```php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => null,
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

    'default' => 'default',

    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(
            connection: new Config\SQLite\MemoryConnectionConfig(),
            queryCache: true
        ),
        // ...
    ],
];
```

Identically, you can edit any configuration for any of component. For example, we can change default HTTP headers
via `app/config/http.php`:

```php
return [
    'basePath'   => '/',
    'headers' => [
        'Server' => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
    'middleware' => [],
];
```

To find which config file corresponds to the proper config object, check the value
of [CONFIG constant](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19):

```php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
```

> **Note**
> See the reference for each component configuration in the related documentation section. 

## Accessing Configuration Values

### Config objects

All of the config object in the Spiral Framework are injectable. It means than when you try to resolve a config object 
via container it will automatically load all values from config file or will use default settings.

```php
use Spiral\Http\Config\HttpConfig;

class SomeService 
{
    public function __construct(
        private HttpConfig $config // <-- Container will automatically load values from app/config/http.php
    ) {
        $path = $this->config->getBasePath();
    }
}
```

> **Note**
> Read more about config objects [here](../framework/config.md)
