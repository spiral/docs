# Making Modules
Spiral framework makes it easy to split your application code to external package. Spiral use Composer to load your classes, so you only need to define proper [Bootloader](/v1.0.0/frameworkwork/bootloaders.md) in your module to bind your implementations. 

> You can read about packing your module into composer package [here](https://getcomposer.org/doc/02-libraries.md).

## Writing Module Class
In addition to Bootloaders you can automatically alter application configs and move some files in by creating module class in your extension:

```php
class MyModule implements ModuleInterface
{
    public function register(RegistratorInterface $registrator)
    {

    }
    
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
If you already checked sample spiral application you might notice set of placeholder comments located all across configuration files:

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

Such placeholders provide module an ability to write some strings or code into configuration files (based on your approval), the most common example is when module wants to register it's own view namespace(s):

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

Now, you can register this module in your system which will automatically alter views config (if allowed) and publish all necessary resources.

> Attention, **do not** alter configs using absolute paths, only use directory based [aliases](/v1.0.0/application/directories.md) if you want your module work in any environment.

## Install, Register and Publish your module
Register/publish your module using 'register {module}' and 'publish {module}' commands accordingly. Commands will automatically resolve your module class by converting given name into namespace and added postfix "Module". 

For example, we can create module class which are named `Vendor\CoolModule`, to register such extension we have to use following command:

```
spiral register vendor/cool
```

You can also publish module files without altering any of configuration files:

```
spiral publish vendor/cool
```

### Register command
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

> Do not forget about git.

### Publish command
Publish command does not require any confirmation and usually used when module gets updated resources such as assets, images and etc.

> Check spiral [toolkit](https://github.com/spiral-modules/toolkit/blob/master/source/ToolkitModule.php) or [profiler](https://github.com/spiral-modules/profiler/blob/master/source/ProfilerModule.php) repositories to get some examples.

## ORM/ODM
In cases when your module defines databases models you have to also register your module path in `tokenizer` config to make it classes available for location.

```php
$registrator->configure('tokenizer', 'directories', 'spiral/auth', [
    "directory('libraries') . 'vendor/module/source/Module/Database/',"
]);
```
