# Default Configuration
All of the provided application skeletons are already pre-configured using optimal settings. You can edit any of the settings
by editing the file(s) in `app/config/`.

> If config file does not exist - create it using `<?php return [];` as a base.

## Environment
Web and GRPC templates use DotEnv extension to read environment values from `.env` file located in the root of your project.

```env
# Debug mode disabled view cache and enabled higher verbosity.
DEBUG = true

# Set to application-specific value, used to encrypt/decrypt cookies and etc.
ENCRYPTER_KEY = {encrypt-key}

# Set to TRUE to disable confirmation in `migrate` commands.
SAFE_MIGRATIONS = true
```

You can access these values using `Spiral\Boot\EnvironmentInterface` or via short function `env`.

```php
public function index(EnvironmentInterface $env)
{
    dump($env->get('ENCRYPTER_KEY'));
    dump(env('ENCRYPTER_KEY'));
}
```

## Configuration
Default component configuration location inside the related Bootloader. You can alter such configuration using other bootloaders (see Auto-Configuration) or by creating a default configuration file in `app/config`.

Web and GRPC skeletons include `app/config/database.php` config file:

```php
use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        // database name => driver
        'default' => ['driver' => 'runtime'],
    ],
    'drivers'   => [
        // driver name => options
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            'profiling'  => true,
        ],
    ]
];
```

Identically, you can edit any configuration for any of component. For example, we can change default HTTP headers 
via `app/config/http.php`:

```php
return [
    'headers' => [
        'Server'       => 'Spiral',
        'Content-Type' => 'text/html; charset=UTF-8'
    ],
];
```

To find which config file corresponds to proper config object check the value of [CONFIG constant](https://github.com/spiral/http/blob/master/src/Config/HttpConfig.php#L17):

```php
final class HttpConfig extends InjectableConfig
{
    const CONFIG = 'http';
    
    // ...
```

> See the reference for each component configuration in the related documentation section. 
