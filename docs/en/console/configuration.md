# Console — Getting started

The Spiral Framework's Console component makes it easy to create and manage console commands within your application. By
leveraging the power of the symfony/console package, the Console component provides a convenient interface for working
with console commands.

All of the provided application skeletons include the Console component by default. To enable the component in
alternative builds, make sure to require composer package `spiral/console` and modify the application bootloader:

```php app/src/Application/Kernel.php
protected const LOAD = [
    //...
    \Spiral\Bootloader\CommandBootloader::class,
];
```

To invoke application command just run:

```terminal
php app.php command:name
```

To get a list of available commands:

```terminal
php app.php list
```

To get help about a particular command:

```terminal
php app.php help command:name
```

## Invoke in Application

It is possible to invoke console commands inside your application or application tests. This can be a useful approach
for creating mock data for tests, automatically pre-configuring the database, or performing other tasks that require the
use of console commands.

To invoke a console command from within your application or tests, you can use the `Spiral\Console\Console` service.
This service provides a `run()` method that allows you to execute a console command by its name with arguments.

Here is an example of how you might use it to invoke a console command from within your application:

```php
use Spiral\Console\Console;
use Symfony\Component\Console\Input\ArrayInput;
use Symfony\Component\Console\Output\BufferedOutput;

// ...

public function test(Console $console): string
{
    $input = new ArrayInput([
        '--mount' => '.env',
        '-p' => '{encrypt-key}'
    ]);
    
    $output = new BufferedOutput();
    
    return $console->run('encrypt:key', $input, $output)->fetch();
}
```

## Symfony/Console

The Spiral Console dispatcher is built at the top of the
powerful [Symfony Console](http://symfony.com/doc/current/components/console/introduction.html) component.

> **Note**
> You can register native Symfony Commands in your CLI application.

## Configuration

To apply the custom configuration to the Console component, use `Spiral\Config\ConfiguratorInterface` or create a config
file in `app/config/console.php`:

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

You can modify some of these values during application bootload via `Spiral\Bootloader\ConsoleBootloader`. To register
a new user command:

```php
public function boot(ConsoleBootloader $console): void
{
    $console->addCommand(MyCommand::class);
}
```

> **Note**
> The component is configured by default to automatically discover commands located in the `app/src` directory.

## Connection with RoadRunner

**Please note that console commands invoke outside of the RoadRunner server. Make sure to run an instance of application
server if any of your commands must communicate with it.**
