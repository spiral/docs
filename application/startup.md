# Application startup
To better understand how spiral application started check file located in `webroot/index.php`:

```php
<?php
/**
 * Spiral Framework
 *
 * @author    Anton Titov (Wolfy-J)
 */
define('SPIRAL_INITIAL_TIME', microtime(true));

//No comments
mb_internal_encoding('UTF-8');

//Error reporting
error_reporting(E_ALL | E_STRICT);
ini_set('display_errors', false);

//Root directory
$root = dirname(__DIR__) . '/';

//Composer
require $root . 'vendor/autoload.php';

//Forcing work directory
chdir($root);

//Initiating shared container, bindings, directories and etc
$application = App::init([
    'root'        => $root,
    'runtime'     => $root . 'runtime/',
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
]);

//Let's start!
$application->start();
```

First of all we define helper constant 'SPIRAL_INITIAL_TIME' which is used by spiral Profiler module to correctly display application profiling timeline.

```php
define('SPIRAL_INITIAL_TIME', microtime(true));
```

Secondly spiral will force internal encoding to `UTF-8`, due spiral support multiple languages and databases this is required step.

```php
mb_internal_encoding('UTF-8');
```

Now we can include Composer loader which will help us to load any required component:

```php
require $root . 'vendor/autoload.php';
```

Due spiral application requires few directory names to be defined, we can use current project root (`__DIR__`) to define our primary directory and make sure this is our working directory.

```php
//Root directory
$root = dirname(__DIR__) . '/';

//...

//Forcing work directory
chdir($root);
```

The last, but the most important step is toinitiatee spiral application and it's shared container required for helper traits and sugar functionality. 

All of this operations are located in `Core::init` method, which requires only base list of directories to work (`root`, `libraries` and `application`).

## What is happening in Core init method
The application bootloading is located inside `App::init` method:

```php
public static function init(
    array $directories,
    EnvironmentInterface $environment = null,
    InteropContainer $container = null,
    bool $handleErrors = true
): self {
    //Spiral requires specific DI layer, you can fully overwrite it by providing
    //\Spiral\Core\ContainerInterface or rewrite it partially by using outer Interop compatible
    //for dependency management (applied to get(), has() rules and argument resolution).
    $container = $container instanceof ContainerInterface
        ? $container
        : new SpiralContainer($container);

    //Spiral core interface, @see SpiralContainer (still needed?)
    $container->bindSingleton(ContainerInterface::class, $container);

    /** @var Core $core */
    $core = new static($directories, $container);

    //Core binding
    $container->bindSingleton(self::class, $core);
    $container->bindSingleton(static::class, $core);

    //Core shared interfaces
    $container->bindSingleton(CoreInterface::class, $core);
    $container->bindSingleton(DirectoriesInterface::class, $core);

    //Core shared components
    $container->bindSingleton(BootloadManager::class, $core->bootloader);
    $container->bindSingleton(MemoryInterface::class, $core->memory);

    //Application environment (by default - dotenv extension, applied to all env() functions!)
    if (empty($environment)) {
        $environment = new DotenvEnvironment(
            $core->directory('root') . '.env',
            $core->memory
        );
    }

    $core->setEnvironment($environment);

    //Config factory to intercept all configuration file loading
    $container->bindSingleton(
        ConfiguratorInterface::class,
        $container->make(ConfigFactory::class, [
            'directory' => $core->directory('config')
        ])
    );

    if ($handleErrors) {
        //Do not enabled in tests
        register_shutdown_function([$core, 'handleShutdown']);
        set_error_handler([$core, 'handleError']);
        set_exception_handler([$core, 'handleException']);
    }

    $scope = self::staticContainer($container);
    try {
        //Bootloading our application in a defined GLOBAL container scope
        $core->bootload()->bootstrap();
    } finally {
        self::staticContainer($scope);
    }

    return $core;
}
```

## What is happening in Core start method
When you start your App it will automatically resolve valid instance of DispatcherInterface to be used for this environment, for example in CLI mode spiral will construct `ConsoleDispatcher`, in web `HttpDispatcher`.

Let's look into start method closely:

```php
public function start(DispatcherInterface $dispatcher = null)
{
    $this->dispatcher = $dispatcher ?? $this->createDispatcher();
    $this->dispatcher->start();
}
```

As you can see you are able to provide your own instance of `DispatcherInterface` into start method and define our own flow, in other scenario spiral will create dispatcher based on SAPI.

We can now try to "simplify" index.php file to reflect what will happen in web environment:

```php
$app = App::init([
    'root'        => $root,
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
]);

//Let's start!
$app->http->start();
```

Accessing http dispatcher directly can be valuable if you are planning to use application as middleware in other PSR7 frameworks.

> You can easily redefine start() method in your `App` class to use your custom dispatcher.

Read [this section](/http/flow.md) to know how HttpDispatcher work.

> You can also set your own dispatched in App `bootstrap` method.