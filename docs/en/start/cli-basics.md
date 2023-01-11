# Getting started — First CLI command

The Spiral Framework provides convenient way to create console applications. It has the built-in support for console
commands, which allows you to create command-line interfaces (CLIs) for your application. With console commands, you can
automate tasks, perform maintenance, and interact with your application in a way that is not possible with a standard
web interface.

Managing and using console commands in the Spiral Framework is very easy. The framework provides a convenient interface
for working with console commands by leveraging the power of the `symfony/console` package.

Here are the basic steps to creating a console command in the Spiral Framework:

## Creating a command

Here's an example of a basic console command that outputs current date to the console:

```php
namespace App\Interface\Console;

use Spiral\Console\Command;

final class CurrentDateCommand extends Command
{
    protected const SIGNATURE = 'current:date {format=Y-m-d : Date format}';
    protected const DESCRIPTION = 'Get current date';

    public function __invoke(): void
    {
        $this->writeln(\date($this->argument('format')));
    }
}
```

The Spiral Framework is configured by default to automatically discover commands located in the `app/src` directory
using the [static analysis component](../advanced/tokenizer.md). This means that you don't have to manually register
your commands or create a separate configuration file for them.

## Running the command

To get help information for your command, you can run the followed command in your terminal.

```bash
php app.php help current:date
```

This will display the command's signature, description, and any available arguments or options.

```bash
Description:
  Get current date

Usage:
  current:date [<format>]

Arguments:
  format                Date format [default: "Y-m-d"]

Options:
  -h, --help            Display help for the given command. When no command is given display help for the list command
  -q, --quiet           Do not output any message
  -V, --version         Display this application version
      --ansi|--no-ansi  Force (or disable --no-ansi) ANSI output
  -n, --no-interaction  Do not ask any interactive question
  -v|vv|vvv, --verbose  Increase the verbosity of messages: 1 for normal output, 2 for more verbose output and 3 for debug
```

<br>
<br>

**That's it! You've successfully set up your first console command in the Spiral Framework.**

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Cli configuration](../console/configuration.md)
* [Create command](../console/commands.md)
* [Interceptors](../console/interceptors.md)
* [Command input validation](../cookbook/console-validation.md)