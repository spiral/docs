# Console command input validation

The ёspiral/filtersё is a powerful component for filtering and validating input data. It allows you to create filters
that can be used to map and validate input data from various sources, such as HTTP requests, gRPC requests and console
commands.

With filters, you can map console command input data into structured objects, and then use those objects to validate the
input data.

Using filters to validate console command input data can help to ensure that your commands are receiving clean, valid
data, and can also make it easier to manage your validation logic. Additionally, filters can be reused across different
console commands, which can help to reduce code duplication and make it easier to maintain your application.

## Console command

Let's imagine that we have a console command that accepts a user's name and email address as input:

```php
namespace App\Command;

use Spiral\Console\Command;

final class UserRegister extends Command
{
    protected const SIGNATURE = <<<CMD
        user:register
        {username : User username}
        {email : User email address}
        {--a|admin : Mark as admin}
        {--s|send-verification-email : Send a verification email to the user}
CMD;

    public function perform(): int
    {
        // ...

        return self::SUCCESS;
    }
}
```

And we want to validate the input data before we use it to create a new user. We can do this by using filters.

## Input source

In order to use the filters component with console commands, you will need to bind an instance of
`Spiral\Filters\InputInterface` to your console command's input.

Here is an example of how you might bind an instance of `InputInterface` to your console command's input:

```php
namespace App\Application\Console;

use Spiral\Filters\InputInterface as FilterInputInterface;
use Symfony\Component\Console\Input\InputInterface;

final class ConsoleInput implements FilterInputInterface
{
    public function __construct(
        private readonly InputInterface $input
    ) {
    }

    public function withPrefix(string $prefix, bool $add = true): self
    {
        return $this;
    }

    public function getValue(string $source, string $name = null): mixed
    {
        return match ($source) {
            'argument' => $this->input->getArgument($name),
            'option' => $this->input->getOption($name),
            default => throw new \InvalidArgumentException('Invalid input source'),
        };
    }

    public function hasValue(string $source, string $name): bool
    {
        return match ($source) {
            'argument' => $this->input->hasArgument($name),
            'option' => $this->input->hasOption($name),
            default => throw new \InvalidArgumentException('Invalid input source'),
        };
    }
}
```

In order to use your `ConsoleInput` implementation with your console command, you will need to register it in a
bootloader.

Here is an example:

```php
namespace App\Application\Bootloader;

use App\Application\Console\ConsoleInput;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Filters\InputInterface;

final class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        InputInterface::class => ConsoleInput::class,
    ];
}
```

## Attributes

The `spiral/filters` component uses attributes to define the rules that should be used to validate input data.

There are two types of attributes that can be used in our application to request data from console inout:

### Argument

The attribute represents an input field that is passed as an `argument` to a console command.

```php
namespace App\Application\Console\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Argument extends AbstractInput
{
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): mixed
    {
        return $input->getValue('argument', $this->getKey($property));
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'argument:' . $this->getKey($property);
    }
}
```

### Option

The attribute represents an input field that is passed as an `option` to a console command.

```php
namespace App\Application\Console\Attribute;

use Spiral\Attributes\NamedArgumentConstructor;
use Spiral\Filters\Attribute\Input\AbstractInput;
use Spiral\Filters\InputInterface;

#[\Attribute(\Attribute::TARGET_PROPERTY), NamedArgumentConstructor]
final class Option extends AbstractInput
{
    public function __construct(
        public readonly ?string $key = null,
    ) {
    }

    public function getValue(InputInterface $input, \ReflectionProperty $property): mixed
    {
        return $input->getValue('option', $this->getKey($property));
    }

    public function getSchema(\ReflectionProperty $property): string
    {
        return 'option:' . $this->getKey($property);
    }
}
```

## Create filter

Now that we have defined the attributes that we will use to request data from console input, we can create a filter:

```php
namespace App\Command;

use App\Application\Console\Attribute\Argument;
use App\Application\Console\Attribute\Option;
use Spiral\Filters\Model\Filter;

class UserRegisterFilter extends Filter
{
    #[Argument]
    public string $username;

    #[Argument]
    public string $email;

    #[Option]
    public bool $admin = false;

    #[Option(key: 'send-verification-email')]
    public bool $sendVerificationEmail = false;
}
```

## Use filter

Now that we have created a filter, we can use it to validate the input data that is passed to our console command:

```php
namespace App\Command;

use Spiral\Console\Command;

final class UserRegister extends Command
{
    protected const SIGNATURE = <<<CMD
        user:register
        {username : User username}
        {email : User email address}
        {--a|admin : Mark as admin}
        {--s|send-verification-email : Send a verification email to the user}
CMD;

    public function perform(UserRegisterFilter $input): int
    {
        $this->writeln(\sprintf('Username: %s', $input->username));
        $this->writeln(\sprintf('Email: %s', $input->email));
        $this->writeln(\sprintf('Is admin: %s', $input->admin ? 'yes' : 'no'));

        // $user = new User(
        //     username: $filter->username,
        //     email: $filter->email,
        //    admin: $filter->admin
        // );

        // Store the user in the database...

        if ($input->sendVerificationEmail) {
            $this->writeln('Sending verification email...');

            // Send the verification email...
        }

        $this->writeln(\sprintf('User %s registered', $input->username));

        return self::SUCCESS;
    }
}
```

Now you can run the console command:

```bash
php app.php user:register john_smith john@site.com -as
```

## Validation

The `spiral/filters` component uses the `spiral/validation` component to validate the input data.

> **Note**
> The component relies on [Validation](../validation/factory.md) component, make sure to read it first.

In order to use validation, you will need to define the rules that should be used to validate the input data.

Here is an example:

```php
use App\Application\Console\Attribute\Argument;
use App\Application\Console\Attribute\Option;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class UserRegisterFilter extends Filter implements HasFilterDefinition
{
    #[Argument]
    public string $username;

    #[Argument]
    public string $email;

    #[Option]
    public bool $admin = false;

    #[Option(key: 'send-verification-email')]
    public bool $sendVerificationEmail = false;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => ['required', 'string', ['string::longer', 3], ['string::shorter', 32]],
            'email' => ['required', 'email'],
        ]);
    }
}
```

Now if you run the console command with invalid input data, you will get an error message:

```bash
php app.php user:register jh john
```

Something like this:

```
[Spiral\Filters\Exception\ValidationException]
The given data was invalid.
```

But the error message won't contain details about the validation errors. If you want to get the details, you will need
to create an interceptor for console commands that will handle the `Spiral\Filters\Exception\ValidationException`
exception and display detailed information about the validation errors:

```php
namespace App\Command;

use Spiral\Console\Command;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Symfony\Component\Console\Output\OutputInterface;

class HandleValidationExceptions implements CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            return $core->callAction($controller, $action, $parameters);
        } catch (\Spiral\Filters\Exception\ValidationException $e) {
            $output = $parameters['output'];
            \assert($output instanceof OutputInterface);

            $output->writeln('<fg=red>Validation errors:</>');
            foreach ($e->errors as $key => $error) {
                $output->writeln(\sprintf('<fg=red>%s: %s</>', $key, $error));
            }

            return Command::FAILURE;
        }
    }
}
```

And register the interceptor in `app\config\console.php`:

```php
<?php

declare(strict_types=1);

return [
    'interceptors' => [
        App\Command\HandleValidationExceptions::class,
        // ...
    ],
];
```

> **Note**
> Read more about console interceptors [here](../console/interceptors.md).

That's it. Now if you run the console command with invalid input data:

```bash
php app.php user:register jh john -as
```

you will get an error message with details about the validation errors like this:

```
Validation errors:
username: Text must be longer or equal to 3.
email: Must be a valid email address.
```
