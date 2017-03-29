# Console Dispatcher and Embedded commands
Spiral Console dispatcher built at top of powerful [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) component and provides ability to register console commands using automatic Tokenizer class lookup.

> You are able to register native Symfony Commands in your CLi application.

## Running CLI
To get access to Spiral CLI you must execute `php webroot/index.php {command}` from your root folder. Thought, there is two file shortcuts for Windows and Unix systems which will simplify that command to `spiral {command}` or `./spiral {command}`.

> Do not forget to give executable permission to 'spiral' file. In addition, you might rename `spiral.bat` and `spiral` file into project specific names.

## Execute commands outside of CLI environment
As in case with HttpDispatcher you can get access to ConsoleDispatcher and it's commands at any moment, let's try to execute 'help' command from our controller:

```php
protected function indexAction()
{
    dump($this->console->run('help')->getOutput()->fetch());
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
$application->console->consoleApplication();
```

## Configuration
By default ConsoleDispatcher is able to locate command automatically, you always alter it's behaviour by modifying console config:

```php
return [
    /*
     * When set to true console will try to locate commands automatically using Tokenizer.
     */
    'locateCommands'    => true,

    /*
     * Manually registered set of commands. Use this section when locateCommands is off.
     */
    'commands'          => [],

    /*
     * This set of commands will be executed when command "configure" run. You can declare command
     * options, header and footer.
     */
    'configureSequence' => [
        'views:compile' => ['options' => []],
        'ide-helper'    => ['options' => [], 'header' => "\nGenerating IDE helper classes..."],

        /*{{sequence.configure}}*/
    ],

    /*
     * Set of commands executed inside "update" command.
     */
    'updateSequence'    => [
        'odm:schema' => ['options' => []],
        'orm:schema' => ['options' => []],
        /*{{sequence.update}}*/
    ]
];
```

> Located commands are cached in static application cache, if you want re-locate commands execute `console:reload` in your terminal.