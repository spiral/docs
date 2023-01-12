# Getting started — Configuration

The Spiral Framework makes it super easy to set up your application with config files. All the config files live in the
`app/config` directory, and they let you configure things like connection to a database, set up cache storage, manage
queues, etc.

You don't even have to create any config files yourself, because the Spiral Framework comes with a default setup. But if
you want to, you can also use environment variables to change the main parameters of your application.

> **Note**
> Caching the config files is not necessary as they are only loaded once during application bootstrapping and not
> reloaded unless the application is restarted.

## Environment Variables

The Spiral Framework integrates with [Dotenv](https://github.com/vlucas/phpdotenv), so you can store all your
environment variables in a `.env` file at the root of your project.

> **Note**
> If you're starting a new project, there's a `.env.sample` file that you can use as a guide.

Using environment variables is a great way to separate the configuration of your application from the code itself. This
makes it easy to change certain options depending on the environment you're running the application in, without having
to modify the code.

For example, you may have different database credentials for your local development environment and for production.
Instead of hardcoding these credentials in the code, you can set them as environment variables and then reference them
in your configuration files.

<details>
  <summary>Click to show example of available environment variables.</summary>

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
> Any variable in your `.env` file can be overridden by external environment variables such as server-level or
> system-level environment variables.

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

In the Spiral Framework, all config objects are injectable, which makes it easy to use them in your application.

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

Each injectable config class in the Spiral Framework contains
a [CONFIG constant](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L19) that defines the name of
the corresponding config file. When the container resolves an injectable config object, it automatically loads all
values from the config file and assigns them to the properties of the config object.

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

<br><br>

**That's it! You've successfully configured your awesome application.**

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Config objects](../framework/config.md)
* [Kernel and Environment](../framework/kernel.md)