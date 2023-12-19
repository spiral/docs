# The Basics — Scaffolding

Spiral provides `spiral/scaffolder` component. This powerful tool enables developers to quickly and easily generate
application code for various classes, using a set of console commands:

- application bootloaders,
- console commands,
- application configs,
- HTTP controllers, middleware, request filters,
- queue job handlers.

and more...

## Installation

To enable the component, you just need to add the `Spiral\Scaffolder\Bootloader\ScaffolderBootloader` class to the
bootloader list:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

## Configuration

After adding the bootloader, you can customize the component to fit your needs by replacing declaration generators and
their options using the `scaffolder` configuration file.

Here is default configuration for the available declarations:

```php 
use Spiral\Scaffolder\Declaration;

return [
    'declarations' => [
        Declaration\BootloaderDeclaration::TYPE => [
            'namespace' => 'Bootloader',
            'postfix' => 'Bootloader',
            'class' => Declaration\BootloaderDeclaration::class,
        ],
        Declaration\ConfigDeclaration::TYPE => [
            'namespace' => 'Config',
            'postfix' => 'Config',
            'class' => Declaration\ConfigDeclaration::class,
            'options' => [
                'directory' => directory('config'),
            ],
        ],
        Declaration\ControllerDeclaration::TYPE => [
            'namespace' => 'Controller',
            'postfix' => 'Controller',
            'class' => Declaration\ControllerDeclaration::class,
        ],
        Declaration\FilterDeclaration::TYPE => [
            'namespace' => 'Filter',
            'postfix' => 'Filter',
            'class' => Declaration\FilterDeclaration::class,
        ],
        Declaration\MiddlewareDeclaration::TYPE => [
            'namespace' => 'Middleware',
            'postfix' => '',
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Command',
            'postfix' => 'Command',
            'class' => Declaration\CommandDeclaration::class,
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Job',
            'postfix' => 'Job',
            'class' => Declaration\JobHandlerDeclaration::class,
        ],
    ],
];
```

You can customize the class namespace, postfix, and declaration type for each available class declaration type. there's
no need to override the entire declaration configuration. Instead, you can customize only the specific
declaration types that you require.

Here's an example of how you might customize the configuration:

```php app/config/scaffolder.php
use Spiral\Scaffolder\Declaration;

return [
    // ...
    'declarations' => [
        Declaration\MiddlewareDeclaration::TYPE => [
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Endpoint\Console',
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Endpoint\Queue',
            'postfix' => 'Job',
        ],
    ],
];
```

> **Note**
> This approach is particularly helpful in larger applications where many different declaration types may be used. By
> customizing only what's necessary, the configuration process can be simplified and errors can be minimized.

### Changing directory for generated classes

By default, the scaffolder will generate classes in the `app/src` directory. In some cases, you may want to change the
directory where the scaffolder generates classes.

You can accomplish this by using the `directory` option.

```php app/config/scaffolder.php
return [
    'directory' => directpry('app') . '/Generated' // <=============
];
```

You can also change the directory for specific declarations. For example, you may want to generate console commands in
the `app/src/Endpoint/Console` directory. To accomplish this, you can use the `directory` option:

```php app/config/scaffolder.php
return [
    'declarations' => [
        Declaration\CommandDeclaration::TYPE => [
            // ...
            'directory' => directpry('app') . '/Endpoint/Console' // <=============
        ],
    ],
];
```

## Adding custom declarations via ScaffolderBootloader

It's possible to register custom declarations. You can accomplish this by registering your custom declarations with the
`ScaffolderBootloader`:

```php app/src/Application/Bootloader/ScaffolderBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader as BaseScaffolderBootloader;

final class ScaffolderBootloader extends Bootloader
{
    public function init(BaseScaffolderBootloader $scaffolder): void
    {
        $scaffolder->addDeclaration('Repository', [
            'namespace' => 'App\\Repository',
            'postfix'   => 'Repository',
            'class'     => RepositoryDeclaration::class,
            'options'   => [
                'orm' => 'cycle',
                // some custom options
            ],
        ]);
    }
}
```

By registering custom declarations, you can extend the functionality of the component to meet the specific
needs of your application.

## Available Commands

| Command           | Description                       |
|-------------------|-----------------------------------|
| create:bootloader | Create Bootloader declaration     |
| create:command    | Create Command declaration        |
| create:config     | Create Config declaration         |
| create:controller | Create Controller declaration     |
| create:middleware | Create Middleware declaration     |
| create:filter     | Create Request filter declaration |
| create:jobHandler | Create Job Handler declaration    |

Some packages may provide their own Commands. For example, the `Cycle Bridge` package (if installed) provides the
following Commands:

| Command           | Description                          |
|-------------------|--------------------------------------|
| create:migration  | Create Migration declaration         |
| create:repository | Create Entity Repository declaration | 
| create:entity     | Create Entity declaration            | 

> **See more**
> Read more about `Cycle Bridge` package and available console
> commands [here](https://spiral.dev/docs/packages-cycle-bridge).

### Bootloader

This command creates a Bootloader class. Bootloaders are responsible for initializing and configuring components
during application startup. With this command, you can quickly generate the code for a new Bootloader, which
you can then customize to meet your specific needs.

> **See more**
> Read more about bootloaders in [Framework — Bootloaders](../framework/bootloaders.md) section.

```terminal
php app.php create:bootloader <name>
```

`<Name>Bootloader` class will be created.

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\BootloaderDeclaration::TYPE => [
    'namespace' => 'Application\Bootloader',
],
```

#### Example

```terminal
php app.php create:bootloader App
```

The output is:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [];
    protected const DEPENDENCIES = [];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

By using the `-d` option, you can generate a Domain-specific Bootloader.

#### Example

```terminal
php app.php create:bootloader App -d
```

The output is:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

final class AppBootloader extends DomainBootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];
    protected const DEPENDENCIES = [];
    protected const INTERCEPTORS = [
        // Put your interceptors here
    ];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

### Console Command

This command creates a Console Command class. Commands provide a way to execute application functionality via the
console. With this command, you can generate the code for a new Command declaration, which you can then customize to
implement the desired console functionality.

> **See more**
> Read more about console commands in [Console — Getting started](../console/configuration.md) section.

```terminal
php app.php create:command <name> [alias]
```

`<Name>Command` class will be generated. A command name will be equal to `name` or `alias` (if this value is set).

If an alias is not provided, one will be generated automatically from the name. For example, if the name is `CreateUser`
the alias will be `create:user`.

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\CommandDeclaration::TYPE => [
    'namespace' => 'Endpoint\Console',
],
```

#### Example without `alias`

```terminal
php app.php create:command UserRegister
```

The output is:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### Example with alias

```terminal
php app.php create:command UserRegister create:user
```

The output is:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user')]
final class UserRegisterCommand extends Command
```

#### Options and arguments

You can also use `-a` and `-o` options to add arguments and options to the command.

```terminal
php app.php create:command UserRegister -a username -a password -o isAdmin
```

The output is:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the username argument?')]
    private string $username;

    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the password argument?')]
    private string $password;

    #[Option(description: 'Argument description')]
    private bool $isAdmin;

    public function __invoke(): int
    {
        // Put your command logic here
        $this->info('Command logic is not implemented yet');

        return self::SUCCESS;
    }
}
```

#### Command description

You can also use `-d` option to add a description to the command.

```terminal
php app.php create:command UserRegister -d "Register a new user"
```

The output is:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user', description: 'Register a new user')]
final class UserRegisterCommand extends Command
```

### Application Config

This command creates a Config class. Configs provide a way to manage application configuration settings. With this
command, you can generate the code for a new Config, which you can then customize to manage the configuration settings
for your application.

> **See more**
> Read more about application configs in [Framework — Config Objects](../framework/config.md) section.

```terminal
php app.php create:config <name>
```

`<Name>Config` class and `<app directory>/config/<name>.php` file will be created if it doesn't
exist.

#### Available options:

`reverse (r)` - Using this flag, the scaffolder will look for `<app directory>/config/<name>.php` file and create a
Config class based on its contents. The generated class will include default values and getters, and in some cases, it
will also include by-key-getters for array values. If an array-value consists of more than one sub-values with the same
types for keys and sub-values, the scaffolder will try to create a by-key-getter method. If a generated key is
conflicting with an existing method, the by-key-getter will be omitted.

#### Example with an empty config file

```terminal
php app.php create:config app
```

Output config file:

```php app/config/app.php
return [];
```

Output config class:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [];
}
```

#### Example with reversing

```php app/config/app.php
return [
    //will create "getParam()" by-key-getter (successfully singularized name)
    'params' => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //will create "getParameterBy()" by-key-getter (unsuccessfully singularized name)
    'parameter' => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //will create "getValueBy()" by-key-getter (because "getValue()" conflicts with the next "value" field)
    'values' => [
        1 => 'value',
        2 => 'another value',
    ],
    'value' => 'third value',
    //won't create by-key-getter due to only 1 sub-value
    'few' => [
        'one' => 'value',
    ],
    //won't create by-key-getter due to mixed values' types
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //won't create by-key-getter due to mixed keys' types
    'mixedKeys' => [
        'one' => 'value',
        2 => 'another value',
    ],
    //won't create by-key-getter to name conflicts
    //(because "getConflict()" and "getConflictBy()" conflicts with the next "conflict" and "conflictBy" fields)
    'conflicts' => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict' => 'third conflic',
    'conflictBy' => 'fourth conflic',
];
```

```terminal
php app.php create:config my -r
```

The output is:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Default values for the config.
     * Will be merged with application config in runtime.
     */
    protected array $config = [
        'params' => [],
        'parameter' => [],
        'values' => [],
        'value' => '',
        'few' => [],
        'mixedValues' => [],
        'mixedKeys' => [],
        'conflicts' => [],
        'conflict' => '',
        'conflictBy' => '',
    ];

    public function getParams(): array
    {
        return $this->config['params'];
    }

    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    public function getValues(): array
    {
        return $this->config['values'];
    }

    public function getValue(): string
    {
        return $this->config['value'];
    }

    public function getFew(): array
    {
        return $this->config['few'];
    }

    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```

### Http Controller

This command creates a Controller class. Controllers handle HTTP requests and responses for specific endpoints in your
application. With this command, you can generate the code for a new Controller class, which you can then customize
to handle HTTP requests and responses as needed.

> **See more**
> Read more about HTTP controllers in [HTTP — Getting started](../http/configuration.md) section.

```terminal
php app.php create:controller <name>
```

`<Name>Controller` class will be created. Available options:

* `action (a)` (multiple values allowed) - you can add actions using this option
* `prototype (p)` - if set, `PrototypeTrait` will be added

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\ControllerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web',
],
```

#### Example with empty action list

```terminal
php app.php create:controller User
```

The output is:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
}

```

#### Example with a `prototype` option

```terminal
php app.php create:controller User -p
```

The output is:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class UserController
{
    use PrototypeTrait;
}
```

#### Example with an actions list

```bash
php app.php create:controller User \
      -a index \
      -a show \
      -a create \
      -a update \
      -a delete
```

The output is:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function index(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function show(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function create(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function update(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function delete(): ResponseInterface
    {
    }
}
```

### Request Filter

This command creates a Request Filter class. Request filters provide a way to map and validate HTTP requests before they
are handled by the controller. With this command, you can generate the code for a new Request Filter
declaration, which you can then customize to modify HTTP requests as needed.

> **See more**
> Read more about request filters in [Filters — Getting started](../filters/configuration.md) section.

```terminal
php app.php create:filter <name>
```

`<Name>Filter` class will be generated.

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\FilterDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Filter',
],
```

#### Example

```terminal
php app.php create:filter CreateUser
```

> **Warning**
> Make sure that you enabled `Spiral\Validation\Bootloader\ValidationBootloader` bootloader in your application.

The output is:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
}
```

#### Create a filter with properties

`property (p)` - option is used to define properties for the filter class. Each property is defined using the
format `<name>:<source>:<type>`, where:

- `<name>` is the name of the property,
- `<source>` is the source of the input data (e.g. `post`, `get`, `header`, `cookie`, `server`, etc),
- and `<type>` is the type of the property (e.g. `string`, `int`, `bool`, or `array`).

```terminal
php app.php create:filter CreateUser -p username:post -p tags:post:array -p ip:ip -p token:header -p status:query:int
```

The output is:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
    #[Post(key: 'username')]
    public string $username;

    #[Post(key: 'tags')]
    public array $tags;

    #[RemoteAddress(key: 'ip')]
    public string $ip;

    #[Header(key: 'token')]
    public string $token;

    #[Query(key: 'status')]
    public int $status;
}
```

> **Note**
> Read more about available attributes [here](../filters/filter.md).

#### Create a filter with validation rules

To generate a filter with validation rules, simply add the `-s` option to the command:

```terminal
php app.php create:filter CreateUser -p ... -s
```

The output is:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CreateUserFilter extends Filter implements HasFilterDefinition
{
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            // Put your validation rules here
        ]);
    }
}
```

> **Warning**
> There should be installed a validation library in your application. Read more about avaliable validation libraries
> [here](../validation/factory.md).

### HTTP Middleware

This command creates a Middleware class. Middleware provides a way to modify HTTP requests and responses as they pass
through the application's middleware stack. With this command, you can generate the code for a new Middleware
declaration, which you can then customize to modify HTTP requests and responses as needed.

> **See more**
> Read more about middleware in [HTTP — Middleware](../http/middleware.md) section.

```terminal
php app.php create:middleware <name>
```

`<Name>` class will be generated.

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\MiddlewareDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Middleware',
    'postfix' => 'Middleware',
],
```

#### Example

```terminal
php app.php create:middleware Logger
```

The output is:

```php app/src/Endpoint/Web/Middleware/LoggerMiddleware.php
declare(strict_types=1);

namespace App\Endpoint\Web\Middleware;

class LoggerMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(
        \Psr\Http\Message\ServerRequestInterface $request,
        \Psr\Http\Server\RequestHandlerInterface $handler,
    ): \Psr\Http\Message\ResponseInterface
    {
        return $handler->handle($request);
    }
}
```

### Job Handler

This command creates a Job Handler class. Job Handlers handle jobs that are used to handle queued tasks. With this
command, you can generate the code for a new Job Handler declaration, which you can then customize to
handle jobs as needed.

> **See more**
> Read more about jobs and queues in [Queue — Getting started](../queue/configuration.md) section.

```terminal
php app.php create:jobHandler <name>
```

`<Name>Job` class will be created.

#### Configuration

We use the following declaration configuration in our example:

```php
Spiral\Scaffolder\Declaration\JobHandlerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Job',
],
```

#### Example

```terminal
php app.php create:jobHandler UserRegisteredNotification
```

The output is:

```php app/src/Endpoint/Job/UserRegisteredNotificationJob.php
declare(strict_types=1);

namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class UserRegisteredNotificationJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
    }
}
```
