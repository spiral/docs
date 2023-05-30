# Getting started — First CLI command

Spiral offers a convenient approach to creating console applications. It comes with built-in support for console
commands, enabling you to develop command-line interfaces (CLIs) for your application. Console commands empower you to
automate tasks, perform maintenance operations, and interact with your application in ways beyond the limitations of a
standard web interface.

Working with console commands in Spiral is incredibly straightforward. The framework provides a user-friendly interface
that leverages the power of the `symfony/console` package.

Let's walk through the basic steps of creating a console command.

## Creating a command

To create your first command effortlessly, use the scaffolding command:

```terminal
php app.php create:command CurrentDate
```

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mCurrentDateCommand[39m' has been successfully written into '[33mapp/src/Endpoint/Console/CurrentDateCommand.php[39m'.
```

Now, let's inject some logic into our freshly created command.

Here's an example of a console command that outputs current date to the console:

```php app/src/App/Endpoint/Console/CurrentDateCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'current:date')]
final class CurrentDateCommand extends Command
{
    #[Argument(description: 'Date format')]
    public string $format = 'Y-m-d';

    public function __invoke(): int
    {
        $this->writeln(\date($this->format));

        return self::SUCCESS;
    }
}
```

By default, Spiral is configured to automatically discover commands located in the `app/src` directory through the
[static analysis component](../advanced/tokenizer.md). This means you don't have to manually register your commands or
create a separate configuration file for them.

## Running the command

To retrieve help information for your command, execute the following command in your terminal:

```terminal
php app.php help current:date
```

This will display the command's signature, description, and any available arguments or options.

```output
[33mDescription:[39m
  Get current date

[33mUsage:[39m
  current:date [<format>]

[33mArguments:[39m
  [32mformat[39m                Date format[33m [default: "Y-m-d"][39m

[33mOptions:[39m
  [32m-h, --help[39m            Display help for the given command. When no command is given display help for the [32mlist[39m command
  [32m-q, --quiet[39m           Do not output any message
  [32m-V, --version[39m         Display this application version
  [32m    --ansi|--no-ansi[39m  Force (or disable --no-ansi) ANSI output
  [32m-n, --no-interaction[39m  Do not ask any interactive question
  [32m-v|vv|vvv, --verbose[39m  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

<br>

**That's it! You have successfully set up your first console command in Spiral.**

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Cli configuration](../console/configuration.md)
* [Create command](../console/commands.md)
* [Interceptors](../console/interceptors.md)
* [Command input validation](../cookbook/console-validation.md)
* [Scaffolding](../basics/scaffolding.md)