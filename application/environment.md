# Application Environment
Application life cycle will most likely require set of values to be provided outside of it's configuration files, for example database password(s), encryption key and etc.

In order to deliver such values use `EnvironmentInterface` contract:

```php
interface EnvironmentInterface
{
    /**
     * Unique environment ID.
     *
     * @return string
     */
    public function getID(): string;
    
    /**
     * Set environment value.
     *
     * @param string $name
     * @param mixed  $value
     */
    public function set(string $name, $value);

    /**
     * Get environment value.
     *
     * @param string $name
     * @param mixed  $default
     *
     * @return mixed
     */
    public function get(string $name, $default = null);
}
```

You are free to access environment values in your configuration files and controller via short function `env`:

```php
echo env('DATABASE_PASSWORD');
```

> Please note, this function will ONLY work in IoC scope (i.e. inside your application).

## DotEnv configuration
By default, your application relies on '.env' file located in a root directory of your project:

```env
#This value does not used by application core, but you can route some config parameters based on this
#value
SPIRAL_ENV = develop

#This option will enable spiral profiler, benchmarking and global log collection
DEBUG = true

#Set to true to let application store bootloaders data in app memory (you will have to run app:reload command)
CACHE_BOOTLOADERS = false

#Encryption key used by Encrypter component to encrypt/decrypt data and protect your cookies
SPIRAL_KEY = ~random-string~

#Production applications must always have view cache turned on, disabled cache can only be useful
#in development
VIEW_CACHE =

#When false translator will be reloading locale bundles on every request
TRANSLATOR_CACHE = true

#Default session handler, use `(null)` to enable native php session mechanism
SESSION_HANDLER = files

#Cookie name to store session token (make sure this cookie excluded from protection!)
SESSION_COOKIE = SID

#Preferred exception view (supported: [dark|light]/[slow|fast])
EXCEPTION_VIEW = spiral:exceptions/light/slow.php

#Database configuration
DB_NAME = spiral
DB_USERNAME = username
DB_PASSWORD =

#Only with ODM component installed
MONGO_DB = spiral
```

You can track file changes using `getID()` method of `EnvironmentInterface`, any '.env' change will generate new environment id.

## Custom Environments
If you consider '.env' approach slow switch to pure _ENV based implementation:
 
```php
//Follow only values defined in _ENV
$environment = new Environment();

//Custom ENV values
$enviroment->set('KEY', 'VALUE');

$application = App::init([
    'root'        => $root,
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
], $environment);
```

> This approach can be beneficial to test your application.