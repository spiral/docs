# Framework Modules and Extensions
Spiral Framework provides powerful base for writing and mounting your modules/extension packages, such packages can simply add new functionality or even signifcanlty alter framework flow using custom bindings.

> Disclaimer: i have to admit that "modules" is not the best name, "extension" or "packages" will be much better, hovewer a lot of legacy code already uses this name. Let's stick to name "modules" with allowance to use "extensions" periodically.

## General Princible
Spiral modules are managed and bootloaded in runtime using ModuleManager component which is available using short binding "modules". Such component mounted in Core autoloaded list and executed before your application `bootstrap()` method. Every registered module automatically added into configuration file "modules" which, in usual scenario, looks like:

```php
return [
    'spiral/profiler' => [
        'class'     => 'Spiral\Profiler\Profiler',
        'bootstrap' => false,
        'bindings'  => []
    ],
    'spiral/toolkit'  => [
        'class'     => 'Spiral\Toolkit\ToolkitModule',
        'bootstrap' => false,
        'bindings'  => []
    ]
];
```

You don't need to modify such file manually due spiral will do everything for you using console toolkit. In most of cases module installation are performed using `composer require` command, default spiral application will automatically reconfigure project after such command called and install newely available modules. If you wish to get list of already installed modules, their version and description you can execute console command `modules`:

```
+-----------------+------------+---------------+----------+---------------------+----------------------------------------------------------------------------+
| Module:         | Version:   | Status:       | Size:    | Location:           | Description:                                                               |
+-----------------+------------+---------------+----------+---------------------+----------------------------------------------------------------------------+
| spiral/profiler | dev-master | installed     | 158.0 kB | ../github/profiler  | Profiler panel/middleware to manage logs, environment and benchmarks.      |
| spiral/redis    | dev-master | not installed | 22.9 kB  | vendor/spiral/redis | Provides simplified access to Redis Clients using predis extension.        |
| spiral/toolkit  | undefined  | installed     | 381.9 kB | ../github/toolkit   | Spiral toolkit includes set of virtual tags, frontend library and resource |
|                 |            |               |          |                     | manager.                                                                   |
+-----------------+------------+---------------+----------+---------------------+----------------------------------------------------------------------------+
```

You can always manually force module installation without using composer using commands `modules:install --all` and `modules:update` (update module public resources).

ModuleManager will automatically locate every available extensing using spiral [Tokenizer] (tokenizer.md) and `ModuleInterface`, which automatically creates only one requiment for your modules - be loadable.

### Module Class
Module class must only implement `ModuleInterface` which requires to declare static method `getDefinition` and non static "bootsrap".

```php
interface ModuleInterface
{
    /**
     * Module bootstrapping. Custom code can be placed here.
     */
    public function bootstrap();

    /**
     * Module definition should explain where module located, name, description and other meta
     * information about package, by default Definition can be created based on composer.json file.
     *
     * This method is static as it should be called without constructing module object.
     *
     * @param ContainerInterface $container
     * @return DefinitionInterface
     */
    public static function getDefinition(ContainerInterface $container);
}
```

> Such class, in most of cases are only aggregates data/classes required for module installation/behaviour, you can locate your primary extension functionality in another class.

### DefinitionInterface
Must provide information about module location, dependencies, descriptions and be able to create instance of `InstallerInterface`.

```php
interface DefinitionInterface
{
    /**
     * Module name, can contain version or other short description.
     *
     * @return string
     */
    public function getName();

    /**
     * Module description.
     *
     * @return string
     */
    public function getDescription();

    /**
     * Module class name.
     *
     * @return string
     */
    public function getClass();

    /**
     * Location of module class, all module files should be located in same directory.
     *
     * @return string
     */
    public function getLocation();

    /**
     * Total module size in bytes (will calculate all module files including resources, views and
     * module class itself).
     *
     * @return int
     */
    public function getSize();

    /**
     * Get list of module dependencies.
     *
     * @return array
     */
    public function getDependencies();
    
    /**
     * Module installer responsible for operations like copying resources, registering configs, view
     * namespaces and declaring that Module::bootstrap() call is required.
     *
     * @return InstallerInterface
     */
    public function getInstaller();
}
```

### InstallerInterface
Such class are responsible for your module installation, bindings, public resources and etc.

```php
interface InstallerInterface
{
    /**
     * Ways to resolve file conflicts happen while moving public module files to application webroot
     * directory, conflicts may happen only if target file was altered or just different than module
     * declaration.
     */
    const NONE      = 0;
    const OVERWRITE = 1;
    const IGNORE    = 2;

    /**
     * Check if modules requires bootstrapping.
     *
     * @return bool
     */
    public function needsBootstrapping();

    /**
     * Declared module bindings, must be compatible with active container instance and be serializable
     * into array.
     *
     * @return array
     */
    public function getBindings();

    /**
     * Perform module installation. This method must mount all public files, configs, migrations and
     * etc.
     *
     * @param int $conflicts Method to resolve file conflicts.
     * @throws InstallerException
     */
    public function install($conflicts = self::OVERWRITE);

    /**
     * Perform module update, method must udpate all module files, no configs or migrations must be
     * created/altered.
     *
     * @param int $conflicts
     * @throws InstallerException
     */
    public function update($conflicts = self::OVERWRITE);
}
```

>Install command must mount module bindings, configs, migrations and etc. Update command is only suppose to mount/update public resources.

In most of cases you would to extend abstract module class `Module` which will help you to create instances of Installer and definition, such implemenentation will use 
composer.json file to create valid instance of defintion and populate it's fields.

## Module Creation
Let's try to create our sample module and locate it in our application for simplification. We are going to use default module implementation which requires us to create composer.json file. Let's try to dedicate folder `application/classes/Modules/Sample` for such purposes (no autoload section is declared in our case).

Composer.json file might look like:
```json
{
  "name": "vendor/sample",
  "description": "Sample spiral module.",
  "license": "MIT",
  "require": {
    "spiral/framework": "*"
  }
}
```

Now we have to create our module class (Modules\Sample\SampleModule):

```php
class SampleModule extends Module
{
    //We don't need to create any methods yet
}
```

Now we are able to run `modules` command and check it's output (i erased other modules to simplify tutorial):

```
+-----------------+------------+---------------+----------+------------------------------------+-------------------------------------------------------------+
| Module:         | Version:   | Status:       | Size:    | Location:                          | Description:                                                |
+-----------------+------------+---------------+----------+------------------------------------+-------------------------------------------------------------+
 vendor/sample    | undefined  | not installed | 384 B    | application/classes/Modules/Sample | Sample spiral module.                                       |
+-----------------+------------+---------------+----------+------------------------------------+-------------------------------------------------------------+
```

As you can see spiral successufully located our module and ready to install it. In addition to that, default implemnetation resolved module location based on location of
composer.json file (all file related operation in module will be relative to such location).

Now, we would like to tweak our module installer, abstract class `Module` provides us convient method getInstaller for such purposes, let's overwrite this method:

```php
class SampleModule extends Module
{
    public static function getInstaller(
        ContainerInterface $container,
        DefinitionInterface $definition
    ) {
        $installer = parent::getInstaller($container, $definition);

        return $installer;
    }
}
```

Now we able to define logic required to install update our module using default implentation of `InstallerInterface` - `Installer`.

### Bootstrapping
Fisrt of all, you might want to bootstrap your module with application, let's tell our Installer about that:

```php
$installer->requireBootstrap(true);
```

As result module will always be constructed and method `bootstrap` will be called once we will perform extension installation (let's do it later).

### Public Resources
In many cases, you module might want to publish some files to be available for frontend (for example styles, scripts and etc). We can try to locate such files in `application/Modules/Sample/public` directory, let's simply put some image into it. If you will rerun command `modules` you'll see that module size just got bigger as it includes resource now.

Next we would like to register our resouce file in installer:

```php
$installer->publishFile('public/image.jpg', 'module/image.jpg');
```

As result, such file be moved to webroot/module/image.jpg when module is installed or updated. If you wish to register whole directory rather that one file, you can use different method:

```php
$installer->publishDirectory('public', 'module');
```

Result will be the same.

### Bindings
If you wish your module to mount some component bindings you can use installer method `setBindings`, we are going to create short binding associated with our module:

```php
$installer->addBinding('sample', self::class);
```

> Installer bindings support only simple string definions and mounted by ModuleManager (not by module itself), hovewer due you can implement SingletonInterface by your extension classes you can avoid using bootstrap method and make your module loaded on demand. 

```php
class SampleModule extends Module implements SingletonInterface
{
    /**
     * Declaring to IoC that we are singleton.
     */
    const SINGLETON = self::class;

    public static function getInstaller(
        ContainerInterface $container,
        DefinitionInterface $definition
    ) {
        $installer = parent::getInstaller($container, $definition);
        $installer->requireBootstrap(true);

        //From module/public to webroot/module
        $installer->publishDirectory('public', 'module');
        $installer->addBinding('sample', self::class);

        return $installer;
    }
}
```

Now, after installation, we will be able to access our module using short binding `sample`.

> You can also re-bind some system classes and [interfaces] (/framework/interfaces.md) to alter framework behaviour.

## Module Configurations
Obviuosly no one want to create a static module without ability to configure it on project basis, spiral provides convinient way to manage module configuration files and even makes possible to alter other application configurations (for example to automatically register module as middleware, cache store, add view namespaces and etc).

Let's try create configuration file and mount it inside our module:

Configuration file (application/classes/Modules/Sample/config/vendor/module.php):

```php
<?php
/**
 * This is sample module configuration file.
 */
return [
    'someValue' => 123
];
```

Now we can use `ConfigurableTrait` and `ConfiguratorInterface` in our module:

```php
class SampleModule extends Module implements SingletonInterface
{
    use ConfigurableTrait;

    /**
     * Declaring to IoC that we are singleton.
     */
    const SINGLETON = self::class;

    /**
     * Configuration section to be used.
     */
    const CONFIG = 'vendor/module';

    /**
     * @param ConfiguratorInterface $configurator
     */
    public function __construct(ConfiguratorInterface $configurator)
    {
        $this->config = $configurator->getConfig(self::CONFIG);
    }

    public static function getInstaller(
        ContainerInterface $container,
        DefinitionInterface $definition
    ) {
        $installer = parent::getInstaller($container, $definition);
        $installer->requireBootstrap(true);

        $installer->publishDirectory('public', 'module');
        $installer->addBinding('sample', self::class);


        return $installer;
    }
}
```

> Please note that we named our configuration 'vendor/module' to prevent possible collisions.

Now can tell our installer that we want to publish configuration file, we are going to achieve this goal by special class `ConfigWriter`, due this class has decent amount of dependecies we better request it using container:

```php
$installer->registerConfig($container->construct(ConfigWriter::class, [
    'name'   => 'vendor/module',
    'method' => ConfigWriter::MERGE_FOLLOW
]));
```

You might notice parameter "method", such parameter tells to ConfigWriter how it must process situations where configuration already exists, in our case it will merge existed (project specific) configuration with module one on order to keep project specific values (you can also use different method to replace existed config or even create your own implementation of ConfigWriter to perfom more controllable merge).

You might link your ConfigWriter to existed framework configuration to automatically alter it's values during installation, hovewer it's much more reliable to use specialized config writers, in our case let's say that our module want to make it's view files available under namespace "sample".

First of all we can create our view in "application/classes/Modules/Sample/views" directory (i'v created view named "test"). Once it's done we can alter our installer and register `ViewConfig` writer. Such writer will only required us to specify base module directory and expose nice methods to manulate with namespaces:

```php
/**
 * @var ViewConfig $viewConfig
 */
$viewConfig = $container->construct(ViewConfig::class, [
    'baseDirectory' => $definition->getLocation()
]);

$viewConfig->registerNamespace('sample', 'views');

//Registering view namespace
$installer->registerConfig($viewConfig);
```

Our final code for Installer configuration looks like:

```php
public static function getInstaller(
    ContainerInterface $container,
    DefinitionInterface $definition
) {
    $installer = parent::getInstaller($container, $definition);
    $installer->requireBootstrap(true);

    $installer->publishDirectory('public', 'module');
    $installer->addBinding('sample', self::class);

    $installer->registerConfig($container->construct(ConfigWriter::class, [
        'name'   => 'vendor/module',
        'method' => ConfigWriter::MERGE_FOLLOW
    ]));

    /**
     * @var ViewConfig $viewConfig
     */
    $viewConfig = $container->construct(ViewConfig::class, [
        'baseDirectory' => $definition->getLocation()
    ]);

    $viewConfig->registerNamespace('sample', 'views');

    //Registering view namespace
    $installer->registerConfig($viewConfig);

    return $installer;
}
```

## Install/update time!
We are done with our module installation configuration, now we can install it (if you module were loaded using componser spiral will do it automatically) using `modules:install vendor/sample -v` command.

```
> spiral modules:install vendor/sample -v
[vendor/sample] Mounting configurations.
[vendor/sample] Updating configuration 'vendor/module'.
[vendor/sample] Updating configuration 'views'.
[vendor/sample] Mounting migrations.
[vendor/sample] Publishing module files.
[vendor/sample] Publishing file '[module]module/image.jpg'.
Module 'vendor/sample' was successfully installed.
```

> You can always re-install module to update it's bindings and configs by adding `-f` flag to command.

We can jump to modules configuration file to see how it looks like:
```php
return [
    'spiral/profiler' => [
        'class'     => 'Spiral\Profiler\Profiler',
        'bootstrap' => false,
        'bindings'  => []
    ],
    'spiral/toolkit'  => [
        'class'     => 'Spiral\Toolkit\ToolkitModule',
        'bootstrap' => false,
        'bindings'  => []
    ],
    'vendor/sample'   => [
        'class'     => 'Modules\Sample\SampleModule',
        'bootstrap' => true,
        'bindings'  => [
            'sample' => 'Modules\Sample\SampleModule'
        ]
    ]
];
```

> You can also check newely created configuration file "vendor/module" and altered views config.

Now our module is installed, it's bindings mouted and resources published! Let's try to do something in our controller:

```php
public function index()
{
    dump($this->sample);
    dump($this->sample->config());

    return $this->views->render('sample:test');
}
```

If your module were updated and new resouce files appear (or old resouce files were changes, i'v added one more file into public module directory) we can re-publish it using `modules:update -v` command.

```
[vendor/sample] Publishing module files.
[vendor/sample] Module file '[module]module/image.jpg' already published.
[vendor/sample] Publishing file '[module]module/new.jpg'.
Module 'vendor/sample' was successfully udpated.
```

## Console Commands 
You can freely create your module specfic commands, spiral will locate and mount them automatically, if, for some reason, your command wasn't added while module instalation you can always run `console:refresh`. Only make sure your command class is loadable.
