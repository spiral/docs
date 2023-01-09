# Console - User Commands

You can add new console commands to your application or register plugin commands using bootloaders. The component is
configured by default to automatically discover commands located in the `app/src` directory.

This documentation page will guide you through the process of creating and using console commands in your application.
Whether you are a beginner or an experienced developer, you will find the information you need to get started.

## Command class

To create a new command, you can either extend `Symfony\Component\Console\Command\Command` or `Spiral\Console\Command`.
Extending the `Spiral\Console\Command` class provides some additional syntax sugar and convenience methods that can make
it easier to build your command.

> **Note**
> Which option you choose will depend on your needs and preferences. Both approaches are valid and can be used to create
> functional console commands.

Here is an example of a simple console command created by extending `Spiral\Console\Command`:

```php
use Spiral\Console\Command;

final class MyCommand extends Command
{
    const NAME        = 'my:command';
    const DESCRIPTION = 'This is my command';

    protected function perform(): int
    {
        // do something special...
        return self::SUCCESS;
    }
}
```

Use constants `NAME` and `DESCRIPTION` to give a name to your Command.

To invoke your command, run the following command in the console:

```bash
php app.php my:command 
```

## Signature

The `SIGNATURE` constant provides an alternative way to define the **name** of your command, as well as
its **arguments** and **options**. This can be a convenient way to specify all of this information in a single place,
rather than defining the name, arguments, and options separately.

```php
class SomeCommand extends Command 
{
    protected const SIGNATURE = <<<CMD
            check:http 
                {url : Site url} 
                {--S|skip-ssl-errors : Skip SSL errors}
CMD;


    public function perform(): int
    {
        $url = $this->argument('url');
        $skipErrors = $this->option('skip-ssl-errors');

        // ...
    }
}
```

In this example the `SIGNATURE` constant defines a command named `check:http` with a required argument `url`, and an
optional option `skip-ssl-errors`.

### Arguments

You can make an argument optional by including a `?` character after its name. For example, the `check:http {url?}`
defines an optional argument.

You can also define a default value for an argument by including an `=` character followed by the default value after
the argument name. For example, `check:http {url=foo}` defines an argument with a default value.

If you want to define an **argument** that expects multiple input values, you can use the `[]` character. For example,
`check:colors {colors[]}` defines an argument that expects multiple values.

If you want to make the argument optional, you can include a `?` character after the `[]` characters, as in
`check:colors {colors[]?}`. This will allow the user to omit the argument if desired.

### Options

Options are useful for specifying additional information or modifying the behavior of a command. They can be used to
enable or disable certain features, specify a configuration file or other input file, or set other parameters that
influence the command's behavior.

Options are a form of user input that is prefixed by two hyphens `--` when specified on the command line. In the
`SIGNATURE` constant, options are defined using the syntax `{--name}`, `{--name=value}` or `{--n|name}` if you want to
use shortcut.

Yes, you can use shortcuts for options to make it easier for users to specify the option when calling the command.

For example, `{--S|skip-ssl-errors}` defines an option named `skip-ssl-errors` with a shortcut of `S`. This means that
the option can be specified using either `--skip-ssl-errors` or `--S` on the command line.

Using shortcuts for options can make it easier and more convenient for users to specify the options they want to use
when calling your command. It's a good idea to choose clear and concise shortcut names that are easy to remember and
type.

If you want to define an **option** that expects multiple input values, you can use the `[]` character. For example,
`check:colors {--colors[]=}` defines an option that expects multiple values.

### Description

It's a good idea to include a description for each `argument` and `option` you define, as it will help users understand
the purpose and format of the input expected by your command. This can make it easier for users to use your command
effectively and avoid errors.

Here is an example of how you might use an argument with a description in a console command:

```php
check:http 
    {url : Site url} 
    {--S|skip-ssl-errors : Skip SSL errors}
```

## Perform method

You can put your user code into the `perform` method. The `perform` supports method injection and
provides `$this->input` and `$this->output` properties to work with user input.

```php
protected function perform(MyService $service): int
{
    $this->output->writeln($service->doSomething());
    
    return self::SUCCESS;
}
```

To get user's data via arguments and/or options, you can make use of `$this->argument("argName")` or
`$this->option("optName")`.

> **Note**
> In addition [here](/cookbook/console-validation.md) you can find information how to use `spiral/filters` component 
> for console commands. 

## Helper Methods

You can use a set of helper methods available inside the `Spiral\Console\Command`. Given examples are intended to be
called in the `perform` method.

To write into output:

```php
$this->writeln('hello world');
```

To write into output without advancing to a new line:

```php
$this->write('hello world');
```

To write formatted output without advancing to a new line:

```php
$this->sprintf('Hello, <comment>%s</comment>', $name);
```

> **Note**
> This method is compatible with the `sprintf` definition.

To check if the current verbosity mode is higher than `OutputInterface::VERBOSITY_VERBOSE`:

```php
dump($this->isVerbose());
```

> **Note**
> You can freely use the `dump` method in console commands.

To render a table:

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

To add a table separator:

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

To write a message as information output:

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

The `Spiral\Console\Confirmation\ApplicationInProduction` class provided by the component makes it easy to
ask the user for confirmation before running a command if the application is running in production mode. This can help
prevent accidental or unintended changes to the production environment.

To use it, you can inject an instance of it into your command's `perform()` method. Then, you can use 
the `confirmToProceed()` method to ask the user for confirmation before proceeding with the command.

```php
use Spiral\Console\Confirmation\ApplicationInProduction;

final class MigrateCommand extends Command
{
    protected const NAME = 'db:migrate';

    public function perform(ApplicationInProduction $confirmation): int
    {
        if (!$confirmation->confirmToProceed()) {
            return self::FAILURE;
        }
        
        // run migrations...
    }
}
```

## Arguments and Options

The Command class makes it easy to define arguments and options for your console command. By setting
the `ARGUMENTS` and `OPTIONS` constants:

```php
const ARGUMENTS = [
    ['argName', InputArgument::REQUIRED, 'Argument name.']
];
    
const OPTIONS = [
    ['optName', 'c', InputOption::VALUE_NONE, 'Some option.']
];
```

## Events

| Event                                | Description                                                     |
|--------------------------------------|-----------------------------------------------------------------|
| Spiral\Console\Event\CommandStarting | The Event will be fired `before` executing the console command. |
| Spiral\Console\Event\CommandFinished | The Event will be fired `after` executing the console command.  |
