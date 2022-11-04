# Console - Installation and Configuration

All of the provided application skeletons include the Console component by default. To enable the component in alternative
builds, make sure to require composer package `spiral/console` and modify the application bootloader:

```php
[
    //...
    Spiral\Bootloader\CommandBootloader::class,
]
```

Make sure to include this bootloader last, as it will also activate the set of default commands for previously added
components.

To invoke application command run:

```bash
php app.php command:name
```

To get a list of available commands:

```bash
php app.php
```

To get help about a particular command:

```bash
php app.php help command:name
```

## Invoke in Application

You can invoke console commands inside your application or application tests. This approach can be useful to create mock
data for tests or automatically pre-configure the database.

Use `Spiral\Console\Console` to do that:

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
> By default, the Console component uses an auto-discovery mode to find all user commands in `app/` automatically.

## Sequences

There are two types of sequences in the Spiral Framework:

### Configure

A set of commands that will be run after invoke command `php app.php configure`

To register a command in configure sequence:

```php
use Symfony\Component\Console\Output\OutputInterface;
use Psr\Container\ContainerInterface;

public function boot(ConsoleBootloader $console): void
{
    // Add console command in a sequence
    $console->addConfigureSequence('my:command', '<info>Running my:command...</info>');
    
    // Add closure in a sequence
    // It supports auto-wiring of arguments
    $console->addConfigureSequence(function(OutputInterface $output, ContainerInterface $container) {
        // do something
        $output->writeln('...');
    }, '<info>Running my:command...</info>');
}
```

### Update

A set of commands that will be run after invoke command `php app.php update`

To register command in update sequence:

```php
public function boot(ConsoleBootloader $console): void
{
    $console->addUpdateSequence('my:command', '<info>Running my:command...</info>');
    
    // Add closure in a sequence
    // It supports auto-wiring of arguments
    $console->addUpdateSequence(function(OutputInterface $output, ContainerInterface $container) {
        // do something
        $output->writeln('...');
    }, '<info>Running my:command...</info>');
}
```

### Adding sequence

You can create your own sequence. To create a new sequence (or to add a command to an already created custom sequence), 
call the `addSequence` method in the `Spiral\Console\Bootloader\ConsoleBootloader` class. The name of the sequence must 
be passed as the first parameter. The sequence name cannot be `configure` or `update`. They are already used in the framework.

```php
public function boot(ConsoleBootloader $console): void
{
    $console->addSequence('name', 'my:command', '<info>Running my:command...</info>');
    
    // Add closure in a sequence
    // It supports auto-wiring of arguments
    $console->addSequence('name', function(OutputInterface $output, ContainerInterface $container) {
        // do something
        $output->writeln('...');
    }, '<info>Running my:command...</info>');
}
```

## Connection with RoadRunner

Please note that console commands invoke outside of the RoadRunner server. Make sure to run an instance of application
server if any of your commands must communicate with it.
