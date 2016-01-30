# Console Dispatcher and Embedded commands
Spiral Console dispatcher built at top of powerful [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) components and provides ability to register console commads using automatic Tokenizer based indexation.

## Running CLI
To get access to Spiral CLI you must execute `php webroot/index.php {command}` from your root folder. There is two file shortcuts for Windows and Unix systems which will simplify that command to `spiral {command}` or `./spiral {command}`.

> Do not forget to give executable permission to 'spiral' file. In addition, you might rename `spiral.bat` and `spiral` file into project specific names.

## Execute commands outside of CLI environment
As in case with HttpDispatcher you can get access to ConsoleDispather and it's commands at any moment, let's try to execute 'help' command from our controller:

```php
protected function indexAction()
{
    dump($this->console->command('help'));
}
```

You can provide custom Input and Output interfaces into command method or even specify parameters via array:

```php
  dump($this->console->command('command', [
    'name' => 'Some value' 
  ]));
```

## External Access
In some cases you might want to get access to Symfony Console application without letting spiral to start,
this can be achieved following way (see `index.php` file):

```php
//Initiating shared container, bindings, directories and etc
$application = App::init([
    'root'        => $root,
    'libraries'   => $root . 'vendor/',
    'application' => $root . 'app/',
    //other directories calculated based on default pattern, @see Core::__constructor()
]);

//Let's NOT start!
//$application->start();

//Symfony Console instance with App specific commands
$application->console->application();
```

## Embedded Commands
Spiral provides big set of commands you might use if your application:

| Command            | Description                                                              |
| ---                | ---                                                                      |
| configure          | Configure file permissions, install modules and render view file         |    
| register           | Register module configs and publish it's resources                       |    
| publish            | Publish specific module resources                                        |    
| server             | Run Spiral Development server on specified host and port                 |
| update             | Perform application schemas and cache update                             |
| console:reload     | **Reindex console commands (run after creating command in application)** |
| app:key            | Update encryption key for current environment                            |
| app:reload         | Reload application boot-loading list *(only when cache is enabled)*      |
| app:touch          | Touch configuration files to reset their cached state                    |
| db:describe        | Describe table schema of specific database                               |
| db:list            | Get list of available databases, their tables and records count          |
| i18n:dump          | Dump given locale using specified dumper and path                        |
| i18n:reload        | Force Translator to reload locales                                       |
| i18n:index         | Index all declared translation strings and usages                        |
| inspect            | Inspect ORM and ODM models to locate unprotected field and rules         |
| inspect:entity     | Inspect specified entity schema to locate unprotected field and rules    |
| odm:schema         | Update ODM schema.                                                       |
| odm:uml            | Export ODM schema to UML                                                 |
| orm:schema         | Update ORM schema.                                                       |
| views:compile      | Compile every available view file                                        |
| views:reset        | Clear view cache for all environments                                    |

> You can get list of currently available commands by executing: `ConsoleDisptacher->getCommands()`. If you want
to hide core commands from user access simply edit your tokenizer.php config file to exclude core directories from
indexation. You can also bind your own implementation of ClassLocatorInterface to define your class location
mechanism (for example based on some config).
