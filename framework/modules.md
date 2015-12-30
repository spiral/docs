# Making Modules
Spiral framework will be useless without an ability to alter it's core functionality, connect extensions, add resources or visual plugins (widgets/tags). Any of this features can be achieved using ModuleManager and Bootloaders.

## Writing Bootloader
Often your module might need to define set of container bindings or bootload specific code, in this case you can simply use technique described in [bootloaders](/framework/bootloaders.md) section.

> In some cases (for example, when you package contains only widgets or delcarative singletons) you might skip Bootloader creation for your module.

## Writing Module Class
Usually your module code goes with set of resources to be added to application or set of configs to be mounted. Both of this operations can be performed using module class which can be created by simply implementing `Spiral\Modules\ModuleInterface`:

```php
class MyModule implements ModuleInterface
{
    /**
     * {@inheritdoc}
     */
    public function register(RegistratorInterface $registrator)
    {

    }
    
    /**
     * {@inheritdoc}
     */
    public function publish(PublisherInterface $publisher, DirectoriesInterface $directories)
    {

    }
}
```

### Publish method
Public method in your module are responsible for moving and publishing files into application, for example you can move configurations or some assets using directories interface aliases:

```php
/**
 * {@inheritdoc}
 */
public function publish(PublisherInterface $publisher, DirectoriesInterface $directories)
{
   $publisher->publish(
        __DIR__ . 'config/config.php',                    //Module config source
        $directories->directory('config') . 'config.php', //Final config filename
        PublisherInterface::FOLLOW,                       //FOLLOW = do not overwrite existed
        FilesInterface::READONLY                       
    );

    //You can either publish whole directory content
    $publisher->publishDirectory(
        __DIR__ . '../resources/',
        $directories->directory('public') . 'resources/',
        PublisherInterface::OVERWRITE
    );
}
```

### Register method
If you already checked sample spiral application you might notice set of weird comments located all across configuration files:

```php
'namespaces'  => [
    /*
     * This is default application namespace which can be used without any prefix.
     */
    'default'  => [
        directory("application") . 'views/',
        /*{{namespaces.default}}*/
    ],
    /*
     * This namespace contain few framework views like http error pages and exception view
     * used in snapshots. In addition, same namespace used by Toolkit module to share it's
     * views and widgets.
     */
    'spiral'   => [
        directory("libraries") . 'spiral/framework/source/views/',
        directory('libraries') . 'spiral/toolkit/source/views/',
        /*{{namespaces.spiral}}*/
    ],
    'profiler' => [
        directory('libraries') . 'spiral/profiler/source/views/',
        /*{{namespaces.profiler}}*/
    ],
    /*{{namespaces}}*/
],
```

Such placeholders provide module an ability to write some strings or code into configuration files (based on your approval), the most common example is when module wants to register it's own view namespace(s), this can be achived by following code:

```php
public function register(RegistratorInterface $registrator)
{
    $registrator->configure('views', 'namespaces', 'vendor/my-module', [
        "'my-module' => [",
        "   directory('libraries') . 'vendor/my-module/source/views/',",
        "]"
    ]);
}
```

Now, you can register this module in your system which will automatically alter views config (if allowed) and publish all nesessary resources.

> Attention, **do not** alter configs using absolute paths, only use directory based [aliases](/application/directories.md) if you want your module work in any environment.

## Installing, Registering and Publishing your module
Registering and publishing modules can be done using simple console commands 'register {module}' and 'publish {module}'. Commands will automatically resolve your module class by converting given name into namespace and added postfix "Module". 

> You can read about packing your module into composer package [here](https://getcomposer.org/doc/02-libraries.md).

For example, we can create module class which are named `Vendor\CoolModule`, to register such extension we have to use following command:

```
spiral register vendor/cool
```

You can also publish module files:

```
spiral publish vendor/cool
```

### Registering command
When you run `register` command with your module specified, all configuration sections will be highlighted for you and your confirmation will be requested:

```
> spiral register spiral/toolkit
Module requests following configs to be altered:
+--------+-------------------+--------------------------------------------------------------------------------------+
| Config | Section           | Added Lines                                                                          |
+--------+-------------------+--------------------------------------------------------------------------------------+
| views  | namespaces.spiral | directory('libraries') . 'spiral/toolkit/source/views/',                             |
| views  | dark.processors   | //Provides ability to automatically include js and css requested by widgets and tags |
|        |                   | Spiral\Toolkit\AssetManager::class,                                                  |
+--------+-------------------+--------------------------------------------------------------------------------------+

Confirm module registration (y/n)
```

You can either say "no" and alter configuration files manually or accept modification which will modify and validate (at this moment only syntax validation) altered files:

```
> spiral register spiral/toolkit -vv
Module requests following configs to be altered:
+--------+-------------------+--------------------------------------------------------------------------------------+
| Config | Section           | Added Lines                                                                          |
+--------+-------------------+--------------------------------------------------------------------------------------+
| views  | namespaces.spiral | directory('libraries') . 'spiral/toolkit/source/views/',                             |
| views  | dark.processors   | //Provides ability to automatically include js and css requested by widgets and tags |
|        |                   | Spiral\Toolkit\AssetManager::class,                                                  |
+--------+-------------------+--------------------------------------------------------------------------------------+

Confirm module registration (y/n) y

[Registrator] Syntax of config 'views' has been checked.
[Registrator] Config 'views' were updated with new content.
Module 'Spiral\ToolkitModule' has been successfully registered.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/scripts/spiral/bundle.js' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/scripts/spiral/bundle.js.map' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/scripts/spiral/sf.js' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/scripts/spiral/sf.js.map' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/blank.css' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/blank.less' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/lock.less' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/mixins.less' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/spiral.css' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/spiral.less' already published and latest version.
[Publisher] File 'vendor/spiral/toolkit/source/../resources/styles/spiral/variables.less' already published and latest version.
Module 'Spiral\ToolkitModule' has been successfully published.
```

### Publish command
Publish command does not require any confirmation and usually used when module gets updated resources such as assets, images and etc.

> Check spiral [toolkit](https://github.com/spiral/toolkit/blob/master/source/ToolkitModule.php) or [profiler](https://github.com/spiral/profiler/blob/master/source/ProfilerModule.php) repositories to get some examples.
You should also remember to add your module path into tokenizer config if your code contain ORM/ODM entities or console commands which has to be automatically located by ClassLocator or InvocationLocator.
