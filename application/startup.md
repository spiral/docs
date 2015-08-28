# Application startup
Application enterpoint located in webroot/index.php file and only performs application configuration. Let's check content of `index.php` file.

```php
<?php
/**
 * Spiral Framework, SpiralScout LLC.
 *
 * @package   spiralFramework
 * @author    Anton Titov (Wolfy-J)
 * @copyright ©2009-2011
 */
define('SPIRAL_INITIAL_TIME', microtime(true));

//No comments
mb_internal_encoding('UTF-8');

//Error reporting
error_reporting(E_ALL | E_STRICT);
ini_set('display_errors', false);

//Root directory
$root = dirname(__DIR__);

//Composer
require $root . '/vendor/autoload.php';

//Forcing work directory
chdir($root);

//Let's start!
Application::init([
    'root'        => $root,
    'libraries'   => $root . '/vendor',
    'application' => $root . '/application'
], true)->start();
```

First of all we defining helper constant 'SPIRAL_INITIAL_TIME' used by spiral Profiler module to correclty display application benchmarking. Secondly spiral will force internal encdoing to `UTF-8`, due spiral support multiple languages and databases this is required step.

Due spiral application requires few directory names to be defined, we can use current project root (`__DIR__`) to defined our primary directory and make sure this is our 
working directory. Right after that we can connect Composer autoloading.

The last, but the most imporant step is to create global container used by helpers and trais and pre-load some application components. All of this operations
are located in Core::init method, which requires only base list of directories to work (`root`, `libraries` and `application`). As you may notice, you can 
easily alter your application directory and every nested directory like "classes" or "runtime" with follow this definition.

## What is happening in Core init method
Core method will create instance of spiral core using Core or Appllication class (depends who actually call `init` method). Due Application class simply extending Core
and only defines custom bootrap method, we can easly replace it with core itself:

```php
//Let's start!
\Spiral\Core\Core::init([
    'root'        => $root,
    'libraries'   => $root . '/vendor',
    'application' => $root . '/application'
], true)->start();
```

Let's view the content of Core::init method to better understand how spiral boots.

```php
public static function init(array $directories, $catchErrors = true)
{
    /**
     * @var Core $core
     */
    $core = new static($directories + ['framework' => dirname(__DIR__)]);

    if (empty(self::container())) {
        self::setContainer($core);
    }

    $core->bindings = [
            static::class                => $core,
            self::class                  => $core,
            ContainerInterface::class    => $core,
            ConfiguratorInterface::class => $core,
            HippocampusInterface::class  => $core,
            CoreInterface::class         => $core,
        ] + $core->bindings;

    //Error and exception handlers
    if ($catchErrors) {
        register_shutdown_function([$core, 'handleShutdown']);
        set_error_handler([$core, 'handleError']);
        set_exception_handler([$core, 'handleException']);
    }

    foreach ($core->autoload as $module) {
        $core->get($module);
    }

    //Bootstrapping our application
    $core->bootstrap();

    return $core;
}
```

First you may notice, that there is additional method arguments, which allows spiral to to handle global error handlers, in cases where you want spiral work 
under another application you can disable error handling and do it on higher level.

Instanse of spiral core defined based on what class call init method, in default application it will be `Application` class, so we can rewrite our fist init 
line with:

```php
    $core = new \Application($directories + ['framework' => dirname(__DIR__)]);
```

Due spiral core extends basic spiral Container class and implements set of interfaces required for different compoments with can bind our instace of every 
interface we implemented, we can do it using `bindSingleton` method or directly overwrite bindings:

```php
$core->bindings = [
            static::class                => $core,
            self::class                  => $core,
            ContainerInterface::class    => $core,
            ConfiguratorInterface::class => $core,
            HippocampusInterface::class  => $core,
            CoreInterface::class         => $core,
        ] + $core->bindings;
```

Once core initated and error hanlding configured we can pre-load some of core modules defined in Core/Application property autoload, loading perfomed using Container method `get`:

```php
foreach ($core->autoload as $module) {
    $core->get($module);
}
```

By default autoloadin will contain only two components to load `Loader` and `ModuleManager`:
* Loader will mount itself at top of Composer class loading method and will cache locations of every loaded class in runtime loadmat, this will help to improve performance
a little bit.
* [ModuleManager] (components/modules.md) will load bindings requested by installed modules, and bootstrap module itself if it needed. You can see what modules mounted in your application checking `config/modules.php` file.

The last step in initiation is to bootload core. This methods by default overwriten by `Application` and contains no code.

> Default (`Core`) implementation of bootload method will try to load file `application/bootload.php` - old fashion.

## What is happening in Core start method
Once we have created core and configured environment we can start our application, application flow by itself does not controlled by `Core` or `Application` classes, it is
dedicated to Dispatcher specific for current enviroment, for example in CLI mode spiral will construct `ConsoleDispatcher`, in web `HttpDispatcher`.

Let's look into start method closely:

```php
public function start(DispatcherInterface $dispatcher = null)
{
    $this->dispatcher = !empty($dispatcher) ? $dispatcher : $this->createDispatcher();
    $this->dispatcher->start();
}
```

As you can see you are able to provide your own instance of DispatcherInterface into start method and defined your own flow, in other scenario spiral will create
dispatcher based on sapi.

We can try now to "simplify" index.php file to reflect what will happen in web enviroment (attention, you should not be doing that, due dispather variable will not be set):

```php
//Let's start!
$application = Application::init([
    'root'        => $root,
    'libraries'   => $root . '/vendor',
    'application' => $root . '/application'
], true);

$application->http->start();
```

> You can easily redefine start() method in your `Application` class to use your custom dispatcher.

Read [this section] (/http/flow.md) to know how HttpDispather work.