# Console — User Commands

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


To create your first command effortlessly, use the scaffolding command:

```terminal
php app.php create:command My
```

> **Note**
> Read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md#console-command) section.

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mMyCommand[39m' has been successfully written into '[33mapp/src/Endpoint/Console/MyCommand.php[39m'.
```

```php app/src/App/Endpoint/Console/MyCommand.php
namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'my')]
final class MyCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

To invoke your command, run the following command in the console:

```terminal
php app.php my
```

## Attributes

Since 3.6 release Spiral offers the ability to define console commands using PHP attributes. This allows for a more
intuitive and streamlined approach to defining commands with clear separation of concerns.

Here's an example of defining a console command using attributes:

```php
namespace App\Api\Cli\Command;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;
use Symfony\Component\Console\Input\InputOption;

#[AsCommand(
    name: 'app:create:user', 
    description: 'Creates a user with the given data')
]
final class CreateUser extends Command
{
    #[Argument]
    private string $email;

    #[Argument(description: 'User password')]
    private string $password;

    #[Argument(name: 'username', description: 'The user name')]
    private string $userName;

    #[Option(shortcut: 'a', name: 'admin', description: 'Set the user as admin')]
    private bool $isAdmin = false;

    public function __invoke(): int
    {
        $user = new User(
            email: $this->email,
            password: $this->password,
        );
        
        $user->setIsAdmin($this->isAdmin);
        
        // Save user to database...

        return self::SUCCESS;
    }
}
```

To define the name and description of a console command, you can use either the `Spiral\Console\Attribute\AsCommand`
or `Symfony\Component\Console\Attribute\AsCommand` attribute.

### Arguments

You can mark a class property as an argument for a console command using
the `Spiral\Console\Attribute\Argument` attribute.

```php
#[Argument]
private string $email;
```

`Argument` attribute without any additional parameters, indicating that it will use the property name as the argument
name and will not include a description.

```php
#[Argument(description: 'User password')]
private string $password;
```

Use a description parameter to provide a brief description of the argument.

```php
#[Argument(name: 'username', description: 'The user name')]
private string $username;
```

The name parameter allows you to specify a custom argument name that is different from the property name.

> **Note**
> Any property marked with the Argument attribute must have a scalar typehint, such as `string`, to indicate the
> type of value that the argument should accept.

If you want to set a default value for an argument, you can specify the default value as the property's default value:

```php
#[Argument]
private string $username = 'guest';
```

In this case if no value is provided for the argument during the command invocation, the default value will be used.

In some cases, you may want to make an argument optional in a console command. To do this, you can use a nullable
typehint for the property that represents the argument and specify the default value as `null`.

```php
#[Argument]
private ?string $username = null;
```

### Options

To mark a class property as an option for a console command, you can use the `Spiral\Console\Attribute\Option`
attribute:

```php
#[Option(name: 'admin', description: 'Set the user as admin')]
private bool $isAdmin = false;
````

You can also specify a shortcut for the option using the shortcut parameter:

```php
#[Option(shortcut: 'a', ...)]
private bool $isAdmin = false;
```

By default, the `Option` attribute does not accept input for the option (e.g. `--admin`). If you want to allow the user
to provide a value for the option, you can set the `mode` parameter to `InputOption::VALUE_REQUIRED` or `InputOption::
VALUE_OPTIONAL`, depending on whether the option is required or optional, respectively.

### Questions

By default, when you invoke a console command without passing required arguments, the application will prompt you to
provide a value for each missing argument. However, you can customize the prompt message for each argument using the
`Spiral\Console\Attribute\Question` attribute.

It can be used both as a property attribute and as a class attribute.

```php
#[Option(
    mode: \Symfony\Component\Console\Input\InputOption::VALUE_REQUIRED, 
    ...
)]
private bool $isAdmin = false;
```

When used as a class attribute, you need to define the corresponding `argument` name for each property that requires a
custom prompt.

```php
#[Question(question: 'Provide user email', argument: 'email')]
final class CreateUserCommand extends Command
{
    #[Argument]
    private string $email;

    // ...
}
```

By setting a custom prompt for each argument, you can provide more specific and relevant information to the user, which
can make the interaction more intuitive and streamlined. It is a good practice to use prompts that are clear, concise,
and easy to understand.

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

You can use shortcuts for options to make it easier for users to specify the option when calling the command.

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

To ask a question:

```php
$status = $this->ask('Are you sure?', default: 'no');
```

To ask a multiple choice question:
```php
$name = $this->choiceQuestion(
    'Which of the following is package manager?',
    ['composer', 'django', 'phoenix', 'maven', 'symfony'],
    default: 2,
    allowMultipleSelections: true
);
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

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
