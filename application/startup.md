# Application startup
Te better understand how spiral application works let's try to check it's primary enterpoint localed in webroot/index.php:


```php
<?php
/**
 * Spiral Framework.
 *
 * @license   MIT
 * @author    Anton Titov (Wolfy-J)
 * @copyright ©2009-2015
 */
define('SPIRAL_INITIAL_TIME', microtime(true));
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
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
]);

//Let's start!
$application->start();
```

First of all we are defining helper constant 'SPIRAL_INITIAL_TIME' which is used by spiral Profiler module to correclty display application benchmarking and profiling timeline.

```php
define('SPIRAL_INITIAL_TIME', microtime(true));
```

Secondly spiral will force internal encoding to `UTF-8`, due spiral support multiple languages and databases this is required step.

```php
mb_internal_encoding('UTF-8');
```

Now we can include Composer loader which will help us to load any requred component:

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

The last, but the most imporant step is to intiate spiral application and it's shared container required for helper traits and sugar functionality. 

All of this operations are located in `Core::init` method, which requires only base list of directories to work (`root`, `libraries` and `application`). As you may notice, you can easily alter your application directory and every nested directory like "classes" or "runtime" with follow this definition.

## What is happening in Core init method
Core method will create an instance of `App` class. Let's view the content of `Core::init` method to better understand how spiral boots (not very small method).

```php
public static function init(
    array $directories,
    ContainerInterface $container = null,
    $handleErrors = true
) {
    if (empty($container)) {
        //Default spiral container
        $container = new SpiralContainer();
    }
    
    $container->bindSingleton(ContainerInterface::class, $container);
    
    //Some sugar for modules, technically can be used as wrapper only here and in start method
    if (empty(self::staticContainer())) {
        //todo: better logic is required, stack wrapping?
        self::staticContainer($container);
    }
    /**
     * @var Core $core
     */
    $core = new static($directories, $container);
    
    //Core binding
    $container->bindSingleton(self::class, $core);
    $container->bindSingleton(static::class, $core);
    $container->bindSingleton(DirectoriesInterface::class, $core);
    $container->bindSingleton(BootloadManager::class, $core->bootloader);
    $container->bindSingleton(HippocampusInterface::class, $core->memory);
    $container->bindSingleton(CoreInterface::class, $core);
    
    //Setting environment (by default - dotenv extension)
    $core->environment = new Environment(
        $core->directory('root') . '.env',
        $container->get(FilesInterface::class),
        $core->memory
    );
    
    $core->environment->load();
    
    $container->bindSingleton(EnvironmentInterface::class, $core->environment);
    $container->bindSingleton(Configurator::class, $container->make(
        Configurator::class, ['directory' => $core->directory('config')]
    ));
    
    //Error and exception handlers
    if ($handleErrors) {
        register_shutdown_function([$core, 'handleShutdown']);
        set_error_handler([$core, 'handleError']);
        set_exception_handler([$core, 'handleException']);
    }
    
    $core->bootload()->bootstrap();
    
    return $core;
}
```

Let's try to give short description of what is going on in this method:
* First we have to create an instance of container if none was provided from outside (`Spiral\Core\ContainerInterface`, not Interop), by default spiral will use it's own containter implementation with prepared interface bindings and autowiring abilities.
* Once container is created we have to set it as shared container to make all sugar functionality work in our application (potentially shared container might we set using scoping methodics similar to http requests).
* Now we can create our core which in this case will be based on App class.
* Created core implements some basic interfaces such as DirectoriesInterface, CoreInterface and etc, so we are going to configure container for that.
* After that we can load our enviroment which is, by default, going to use dotenv package and .env file in a root of your project.
* Once enviroment is set we can initiate components configurator (resposinble for supplying configurations across framework and app).
* The last step before bootstrapping and bootloading is to set error handling methods which are, by defualt, localed in a core.
* Bootloading process executed before 'bootstrap' (see your `App`) and ensures that all classes and bootloaders listed in App->load are initiated.

Once all this steps are done we can return our application instance and start it.

## What is happening in Core start method
When you start your App it will automatically resolve valid instance of DispatcherInterface to be used for this enviroment, for example in CLI mode spiral will construct `ConsoleDispatcher`, in web `HttpDispatcher`.

Let's look into start method closely:

```php
public function start(DispatcherInterface $dispatcher = null)
{
    $this->dispatcher = !empty($dispatcher) ? $dispatcher : $this->createDispatcher();
    $this->dispatcher->start();
}
```

As you can see you are able to provide your own instance of `DispatcherInterface` into start method and define our own flow, in other scenario spiral will create dispatcher based on sapi.

We can try now to "simplify" index.php file to reflect what will happen in web enviroment (attention, you should not be doing that, due dispatcher variable will not be set and core will not be able to notify your http about errors):

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

> You can easily redefine start() method in your `App` class to use your custom dispatcher.

Read [this section](/http/flow.md) to know how HttpDispather work.

> You can also set your own dispatched in App `bootstrap` method.
