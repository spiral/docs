# Getting started — Configuration

Spiral makes it super easy to set up your application with config files. All the config files live in the
`app/config` directory, and they let you configure things like connection to a database, set up cache storage, manage
queues, etc.

You don't even have to create any config files yourself, because Spiral comes with a default setup. But if
you want to, you can also use environment variables to change the main parameters of your application.

> **Note**
> Caching the config files is not necessary as they are only loaded once during application bootstrapping and not
> reloaded unless the application is restarted.

## Environment Variables

Using environment variables is a great way to separate the configuration of your application from the code itself. This
makes it easy to store sensitive information like database credentials, API keys, and other configs that you don't want
to hardcode into your application.

Spiral integrates with [Dotenv](https://github.com/vlucas/phpdotenv) through the
`Spiral\DotEnv\Bootloader\DotenvBootloader` class. This bootloader is responsible for loading the environment variables
from the `.env` file and making them available to the application.

It is a common practice to include a `.env.sample` file in a new project which can be used as a guide for setting up the
environment variables.

<details>
  <summary>Click to show .env.sample</summary>

```dotenv .env
# Environment (prod or local)
APP_ENV=local

# Debug mode set to TRUE disables view caching and enables higher verbosity
DEBUG=true
VERBOSITY_LEVEL=verbose # basic, verbose, debug

# Set to an application specific value, used to encrypt/decrypt cookies etc
ENCRYPTER_KEY=...

# Monolog
MONOLOG_DEFAULT_CHANNEL=default
MONOLOG_DEFAULT_LEVEL=DEBUG # DEBUG, INFO, NOTICE, WARNING, ERROR, CRITICAL, ALERT, EMERGENCY

# Queue
QUEUE_CONNECTION=roadrunner

# Cache
CACHE_STORAGE=roadrunner

# Telemetry
TELEMETRY_DRIVER=null

# Serializer
DEFAULT_SERIALIZER_FORMAT=json # csv, xml, yaml

# Session
SESSION_LIFETIME=86400
SESSION_COOKIE=sid

# Authorization
AUTH_TOKEN_TRANSPORT=cookie
AUTH_TOKEN_STORAGE=session

# Mailer
MAILER_DSN=
MAILER_FROM="My site <no-reply@site.com>"
```

</details>

> **Warning**
> A variable that is defined in the `$_SERVER` or `$_ENV` superglobal will take precedence over the value
> of the same variable that is defined in the `.env` file.

The values from the `.env` the file will be copied to your application environment and available
via `Spiral\Boot\EnvironmentInterface` or the `env` function.

### Accessing Environment Variables

You can access environment variables using `Spiral\Boot\EnvironmentInterface`

```php
use Spiral\Boot\EnvironmentInterface;

final class GithubClient
{
    public function __construct(
        private readonly EnvironmentInterface $env
    ) {}
    
    public function getAccessToken(): ?string
    {
        return $this->env->get('GITHUB_ACCESS_TOKEN');
    }
}
```

or via a short function `env()`

```php 
return [
    'access_token' => env('GITHUB_ACCESS_TOKEN'),
    // ...
];
```

### Pre-Processing

Remember that the values in `.env` will be pre-processed, the following changes will take place:

| Value   | PHP Value |
|---------|-----------|
| true    | true      |
| (true)  | true      |
| false   | false     |
| (false) | false     |
| null    | null      |
| (null)  | null      |
| empty   | ''        |

> **Note**
> The quotes around strings will be stripped automatically.

## Configuration

While environment variables are a great way to configure certain options of your application, there may be cases where
you need to make more complex or fine-grained changes to the configuration that are not possible or practical to do with
environment variables alone. In such cases, you can directly modify the config files for the specific components that
you want to change.

For example, you can change the default HTTP headers:

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers' => [
        'Server' => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
    'middleware' => [],
];
```

### Accessing Configuration Values

### Config objects

In Spiral, all config objects are injectable, which makes it easy to use them in your application.

```php
use Spiral\Http\Config\HttpConfig;

final class HttpClient 
{
    private readonly string $basePath;

    public function __construct(
        HttpConfig $config // <-- Container will automatically load values from app/config/http.php
    ) {
        $this->basePath = $this->config->getBasePath();
    }
}
```

Each injectable config class in Spiral contains
a [CONFIG constant](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19) that defines the name of
the corresponding config file. When the container resolves an injectable config object, it automatically loads all
values from the config file and assigns them to the `$config` property of the config object.

> **Note**
> When a config object loads its associated config file, it automatically merges it with the default settings defined in
> the config class. This means that you don't have to include every single option in your config file and only the ones
> you want to change. The default settings are used as a fallback if a value is not present in the config file.

For example, for **HTTP config**:

```php spiral/framework/src/Http/src/Config/HttpConfig.php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
}
```

> **Note**
> See the reference for each component configuration in the related documentation section.

### Determining application environment

The current application environment is determined via the `APP_ENV` variable. You may access this value using the
`Spiral\Boot\Environment\AppEnvironment` injectable enum class.

> **See more**
> Read more about injectable enums in the [Advanced — Container injectors](../advanced/injectors.md#enum-injectors) 
> section.

When you request the `AppEnvironment` from the container it will automatically inject an Enum with the correct value.

```php
use Spiral\Boot\Environment\AppEnvironment;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly AppEnvironment $env
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->env->isProduction()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Determining debug mode

The current debug mode is determined via the `DEBUG` variable. You may access this value using the
`Spiral\Boot\Environment\DebugMode` injectable enum class.

```php
use Spiral\Boot\Environment\DebugMode;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly DebugMode $debug
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->debug->isEnabled()) {
                // ...
            }
            
            // ...
        }
    }
}
```

### Determining verbosity level

The current verbosity level is determined via the `VERBOSITY_LEVEL` variable. You may access this value using the
`Spiral\Exceptions\Verbosity` injectable enum class.


```php
use Spiral\Exceptions\Verbosity;
use Psr\Http\Server\MiddlewareInterface;

final class ErrorHandlerMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly Verbosity $verbosity
    ) {
    }

    public function process(
        ServerRequestInterface $request, 
        RequestHandlerInterface $handler
    ): ResponseInterface {
        try {
            return $handler->handle($request);
        } catch (Throwable $e) {
            if ($this->verbosity === Verbosity::BASIC)
                // ...
            }
            
            if ($this->verbosity === Verbosity::VERBOSE)
                // ...
            }
            
            // ...
        }
    }
}
```

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Config objects](../framework/config.md)
* [Advanced — Container injectors](../advanced/injectors.md)
* [Kernel and Environment](../framework/kernel.md)