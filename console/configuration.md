# Console - Configuration
All of the provided application skeletons include Console component by default. To enable component in alternative builds make
sure to require composer package `spiral/console` and modify the application bootloader:

```php
[
    //...
    Spiral\Bootloader\CommandBootloader::class,
]
```

Make sure to include this bootloader at last, as it will also activate the set of default commands for previously added 
components.

To invoke application command run:

```bash
$ php app.php command:name
```

To get a list of available commands:

```bash
$ php app.php
```

To get help about the particular command:

```bash
$ php app.php help command:name
```

## Invoke in Application
You can invoke console commands inside your application or application tests. This approach can be useful to create mock data for tests or automatically pre-configure the database.

Use `Spiral\Console\Console` to do that:
```php
use Spiral\Console\Console;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

// ...

public functiobn test(Console $console)
{
    $input = new ArrayInput(['args' => 'value']);
    $output = new BufferedOutput();
    
    return $console->run($command, $input, $output);
}
```

## Symfony/Console
Spiral Console dispatcher built at top of powerful [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) 
component.

> You are able to register native Symfony Commands in your CLI application.

## Configuration
To apply the custom configuration to the Console component use `Spiral\Config\ConfiguratorInterface` or create a config file in `app/config/console.php`:

```php
return [
     // application name
     'name'      => null,
     
     // application version
     'version'   => null,
     
     // list of application commands (if auto-discover disabled)
     'commands'  => [],
     
     // list of commands and sequences to run in `app configure`
     'configure' => [],
     
     // list of commands and sequences to run in `app update`
     'update'    => []
];
```

You can modify some of these values during application bootload via `Spiral\Bootloader\ConsoleBootloader`. To register new 
user command:

```php
public function boot(ConsoleBootloader $console)
{
    $console->addCommand(MyCommand::class);
}
```

> Note, by default Console component use auto-discover mode to find all user commands in `app/` automatically.

To register command in configure/update sequence:

```php
public function boot(ConsoleBootloader $console)
{
  $console->addUpdateSequence('my:command', '<info>Running my:command...</info>');
}
```

## Connection with RoadRunner
Please note, console commands are invoked outside of RoadRunner server. Make sure to run an instance of application server
if any of your commands must communicate with it.
