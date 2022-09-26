# Console - User Commands

You can add new console commands to your application or register plugin commands using bootloaders. By default, the
Console component configured to automatically find commands in the `app/src` directory.

To add a new `symfony/console` based Command, simply drop it into your application.

## Command class

To create a new command, you can either extend `Symfony\Component\Console\Command\Command` or `Spiral\Console\Command`
which provides some syntax sugar.

```php
use Spiral\Console\Command;

class MyCommand extends Command
{
    const NAME        = 'my:command';
    const DESCRIPTION = 'This is my command';

    const ATTRIBUTES = [];
    const OPTIONS    = [];

    /**
     * Perform command
     */
    protected function perform(): int
    {
        // do something special...
        return self::SUCCESS;
    }
}
```

Use constants `NAME` and `DESCRIPTION` to give a name to your Command. You can invoke it now using:

```bash
php app.php my:command 
```

## Arguments and Options

Spiral's `Command` class makes it easy to define needed arguments and options:

```php
const ARGUMENTS = [
    ['argument', InputArgument::REQUIRED, 'Argument name.']
];
    
const OPTIONS = [
    ['option', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

To get user's data via arguments and/or options, you can make use of `$this->argument("argName")` or
`$this->option("optName")`.

## Signature

An alternative way to declare the `name` of the command, `arguments` and `options` is the constant `SIGNATURE`.

```php
class SomeCommand extends Command 
{
    protected const SIGNATURE = 'check:http {url : Site url} {--S|skip-ssl-errors : Skip SSL errors}';

    public function perform(): int
    {
        $url = $this->argument('url');
        $skipErrors = $this->option('skip-ssl-errors');

        // ...
    }
}
```

All user supplied arguments and options are wrapped in curly braces. In the following example, the command defines one 
required argument: `url`.

You may also make arguments optional or define default values for arguments:

```php
// Optional argument...
'check:http {url?}'
 
// Optional argument with default value...
'check:http {url=foo}'
```

`Options`, like arguments, are another form of user input. Options are prefixed by two hyphens `--`.

```php
'{--S|skip-ssl-errors : Skip SSL errors}'
```

If the user must specify a value for an `option`, you should suffix the option name with a `=` sign.

```php
'some:command {argument} {--option=default}'
```

If you would like to define `arguments` or `options` to expect multiple input values, you may use the `*` character.

```php
'check:http {url*}'
```

## Perform method

You can put your user code into the `perform` method. The `perform` support method injection and provide `$this->input`
and `$this->output` properties to work with user input.

```php
protected function perform(MyService $service): int
{
    $this->output->writeln($service->doSomething());
    
    return self::SUCCESS;
}
```

## Helper Methods

You can use a set of helper methods available inside the `Spiral\Console\Command`. Given examples are intended to be
called in the `perform` method.

To write into output:

```php
$this->writeln('hello world');
```

To write into output without advancing to the new line:

```php
$this->write('hello world');
```

To write formatted output without advancing to the new line:

```php
$this->sprintf('Hello, <comment>%s</comment>', $name);
```

> **Note**
> This method is compatible with the `sprintf` definition.

To check if current verbosity mode is higher than `OutputInterface::VERBOSITY_VERBOSE`:

```php
dump($this->isVerbose());
```

> **Note**
> You can freely use the `dump` method in console commands.

To render table:

```php
$table = $this->table([
    'Column #1:',
    'Column #2:',
]);

foreach ($data as $row)
{
    $table->addRow([
        $row[1],
        $row[2]
    ]);
}

$table->render();
```

To add table separator:

```php
$table->addRow(new Symfony\Component\Console\Helper\TableSeparator());
```

To determine if the input option is present:

```php
$this->hasOption(...);
```

To determine if the input argument is present:

```php
$this->hasArgument(...);
```

To asks for confirmation:

```php
$status = $this->confirm('Are you sure?', default: false);
```

To asks a question:

```php
$status = $this->ask('Are you sure?', default: 'no');
```

To prompt the user for input but hide the answer from the console:

```php
$status = $this->secret('User password');
```

To write a message as an information output:

```php
$this->info('Some message');
```

To write a message as comment output:

```php
$this->comment('Some message');
```

To write a message as question output:

```php
$this->question('Some question');
```

To write a message as error output:

```php
$this->error('Some error');
```

To write a message as warning output:

```php
$this->warning('Some warning');
```

To write a message as alert output:

```php
$this->alert('Some alert');
```

To write a blank line:

```php
$this->newLine();
$this->newLine(count: 5);
```

## ApplicationInProduction

`ApplicationInProduction` is a class that makes it easy to ask the user for confirmation for actions in command if 
an application is running in production mode.

```php
use Spiral\Console\Confirmation\ApplicationInProduction;

final class MigrateCommand extends Command
{
    protected const NAME = 'db:migrate';
    protected const DESCRIPTION = '...';

    public function perform(ApplicationInProduction $confirmation): int
    {
        if (!$confirmation->confirmToProceed()) {
            return self::FAILURE;
        }
        
        // run migrations...
    }
}
```

## Events

| Event                                | Description                                                     |
|--------------------------------------|-----------------------------------------------------------------|
| Spiral\Console\Event\CommandStarting | The Event will be fired `before` executing the console command. |
| Spiral\Console\Event\CommandFinished | The Event will be fired `after` executing the console command.  |
