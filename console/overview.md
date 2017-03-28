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