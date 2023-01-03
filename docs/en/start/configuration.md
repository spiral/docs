# Configuration

## Introduction

All of the configuration files for Spiral Framework are located in the `app/config` directory.
These files contain options that allow you to configure your database connection, cache storage, sessions and more.

## Environment

It can be useful to have different configuration values depending on the environment in which the application is running.
For example, you may want a simple cache storage for local development in order to reduce the resource requirements of your testing environment, while a more robust and scalable storage may be preferred for production.

To make it easy to manage different configuration values for different environments, Spiral uses the [DotEnv](https://github.com/vlucas/phpdotenv) PHP library.
In a new project installation, you will find a `.env.sample` file in the root directory of your application that defines several common environment variables.
When you install Spiral app, this file is automatically copied to `.env`.

Here is the example of `.env` in the application:

```dotenv
# Environment (prod or local)
APP_ENV=local

# Debug mode set to TRUE disables view caching and enables higher verbosity
DEBUG=true

# Verbosity level
VERBOSITY_LEVEL=verbose # basic, verbose, debug

# Set to an application specific value, used to encrypt/decrypt cookies etc
ENCRYPTER_KEY=def000006531014288871c4c553d2b010e51e35a7f9550d6d23cecdd8b0729acdf9a0323f9d50a94c6d9166ef3ca6b931ac6b5579a71c4a32103b00ed64fa8987411238f

# Monolog
MONOLOG_DEFAULT_CHANNEL=default
MONOLOG_DEFAULT_LEVEL=DEBUG # DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Queue
QUEUE_CONNECTION=roadrunner

# Cache
CACHE_STORAGE=rr-local

# Storage
STORAGE_DEFAULT=default

# Telemetry
TELEMETRY_DRIVER=null

# Session
SESSION_LIFETIME=86400
SESSION_COOKIE=sid

# Authorization
AUTH_TOKEN_TRANSPORT=cookie
AUTH_TOKEN_STORAGE=session

# Mailer
MAILER_DSN=
MAILER_QUEUE=local
MAILER_QUEUE_CONNECTION=
MAILER_FROM="Spiral <sendit@local.host>"

# Set to TRUE to disable confirmation in `migrate` commands
SAFE_MIGRATIONS=true

# Cycle Bridge
CYCLE_SCHEMA_CACHE=true
CYCLE_SCHEMA_WARMUP=false

# Sentry
SENTRY_DSN=

# RoadRunner Logger
LOGGER_FORMAT="%message% %context% %extra%\n"

# Serializer
DEFAULT_SERIALIZER_FORMAT=json # csv, xml, yaml

# Temporal bridge configuration
TEMPORAL_ADDRESS=127.0.0.1:7233
TEMPORAL_TASK_QUEUE=default
```

You can access these values using `Spiral\Boot\EnvironmentInterface`

```php
public function index(EnvironmentInterface $env): void
{
    dump($env->get('ENCRYPTER_KEY'));
}
```

or via `env()` function

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

The default component configuration is located inside the related Bootloader. You can alter such configuration using other
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

In exactly the same way, you can edit any configuration for any component. For example, we can change default HTTP headers
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

In the Spiral Framework, all config objects are injectable.
This means that when you try to resolve a config object through the container, it will automatically load all values from the config file or use default settings.

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