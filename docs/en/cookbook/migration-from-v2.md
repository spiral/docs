# Cookbook — Spiral Framework Upgrade Guide: 2.14 to 3.0

The release of Spiral Framework 3.0 brings significant enhancements and changes over the 2.x series, including support
for both Symfony 6.x and 7.x components, various bug fixes, and usability improvements.

## Key Changes

### Impact: High

#### PHP 8.1

Spiral Framework 3.x requires a minimum PHP version of 8.1.

All classes and interfaces within the Spiral now include typed arguments, properties, and return types. If your
application or package extends any of Spiral's core classes and overrides these methods or properties, it is necessary
to introduce type declarations in your code as well.

> Initially, focus on the `InjectableConfig::$config` property, which must now be typed as array.

#### Readonly properties

Be aware that many class properties have been marked as `readonly` in the latest version. If your code involves directly
setting values in Spiral's core objects, you will need to verify whether these properties can still be overwritten. This
change requires you to adjust how you interact with Spiral's core objects, possibly necessitating the use of constructor
parameters or dedicated methods for setting these values.

## Updating Dependencies

### Impact: High

It's essential to update the dependencies in your application's `composer.json` file to ensure compatibility with Spiral
Framework 3.x. Follow the instructions below to update and add the necessary packages.

### Updates Required

Modify your `composer.json` to update the following packages:

- `symfony/console` to `^6.0`
- `spiral/framework` to `^3.0`
- `spiral/cycle-bridge` to `^2.0`
- `spiral/roadrunner-bridge` to `^3.0`
- `spiral/nyholm-bridge` to `^1.3`
- `spiral/testing` to `^2.0`

After updating and adding these dependencies, run `composer update` to install the new versions and ensure your project
is ready for Spiral 3.x.

## Environment

### Impact: Low

The `Environment` class no longer overwrites existing environment (ENV) variables.

Variables defined in your `.env` file can now be overridden by external environment variables. This means that variables
set at the server level or as system-level environment variables will take precedence. This change also allows you to
set environment variables specifically for PHPUnit.

To revert to the previous behavior where `.env` file variables could overwrite external environment variables, you need
to instantiate the `\Spiral\Boot\Environment` object with the overwrite argument set to `true`.

#### Example

```php
App::create(
    directories: ['root' => __DIR__]
)->run(new \Spiral\Boot\Environment(overwrite: true));
```

> **See more** about how to deal with environment variables in the [documentation](../framework/kernel.md#environment)

## Http

### Impact: High

Bootloaders no longer automatically register middleware within the application.

#### Middleware Previously Registered Through Bootloaders

The following list includes middleware that was previously registered through specific bootloaders. With the update,
these middleware need to be manually registered according to your application's requirements and the desired order of
execution:

- `Spiral\Broadcasting\Bootloader\WebsocketsBootloader` -> `Spiral\Broadcasting\Middleware\AuthorizationMiddleware`
- `Spiral\Bootloader\Auth\HttpAuthBootloader` -> `Spiral\Auth\Middleware\AuthMiddleware`
- `Spiral\Bootloader\Debug\HttpCollectorBootloader` -> `Spiral\Debug\StateCollector\HttpCollector`
- `Spiral\Bootloader\Http\CookiesBootloader` -> `Spiral\Cookies\Middleware\CookiesMiddleware`
- `Spiral\Bootloader\Http\CsrfBootloader` -> `Spiral\Csrf\Middleware\CsrfMiddleware`
- `Spiral\Bootloader\Http\ErrorHandlerBootloader` -> `Spiral\Http\Middleware\ErrorHandlerMiddleware`
- `Spiral\Bootloader\Http\JsonPayloadsBootloader` -> `Spiral\Http\Middleware\JsonPayloadMiddleware`
- `Spiral\Bootloader\Http\SessionBootloader` -> `Spiral\Session\Middleware\SessionMiddleware`

This change grants you the flexibility to select which middleware are essential for your application and the sequence in
which they should be integrated.

> **See more** about HTTP middleware in the [documentation](../http/middleware.md)

#### Example

```php
<?php

declare(strict_types=1);

namespace App\Bootloader;

use App\Controller\AuthController;
use App\Controller\HomeController;
use App\Controller\PageController;
use Spiral\Auth\Middleware\AuthMiddleware;
use Spiral\Bootloader\Http\RoutesBootloader as BaseRoutesBootloader;
use Spiral\Cookies\Middleware\CookiesMiddleware;
use Spiral\Csrf\Middleware\CsrfMiddleware;
use Spiral\Debug\StateCollector\HttpCollector;
use Spiral\Http\Middleware\ErrorHandlerMiddleware;
use Spiral\Http\Middleware\JsonPayloadMiddleware;
use Spiral\Router\Loader\Configurator\RoutingConfigurator;
use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
    protected function globalMiddleware(): array
    {
        return [
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class
        ];
    }

    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                CookiesMiddleware::class,
                SessionMiddleware::class,
                CsrfMiddleware::class,
                AuthMiddleware::class,
            ],
            'api' => [
                //
            ]
        ];
    }
}
```

### Impact: Low

The Spiral's HTTP component no longer includes a default implementation for `Spiral\Http\EmitterInterface` straight out
of the box.

To incorporate an emitter implementation when utilizing HTTP within your application.
For example, if you want to use Spiral with `nginx`, you can add a specific package by running the following command:

```terminal
composer require spiral/sapi-bridge
```

Afterwards, ensure to register the necessary bootloader in your application's bootstrap process. This can be done by
including the `Spiral\Sapi\Bootloader\SapiBootloader` class in the LOAD array within your application's bootloader
configuration:

```php app/Application/Bootloader/AppBootloader.php
protected const LOAD = [
    // ...
    \Spiral\Sapi\Bootloader\SapiBootloader::class,
];
```

> **See more** about how to install and configure RoadRunner bridge in
> the [documentation](../start/server.md#roadrunner-bridge).

### Impact: Medium

The Spiral has discontinued its own implementation of the PSR-7 interfaces, opting instead to integrate
the `nyholm/psr7`package.

As a result of this change, the following classes have been removed from the framework:

- `Spiral\Bootloader\Http\DiactorosBootloader`
- `Spiral\Http\Diactoros\ResponseFactory`
- `Spiral\Http\Diactoros\ServerRequestFactory`
- `Spiral\Http\Diactoros\StreamFactory`
- `Spiral\Http\Diactoros\UploadedFileFactory`
- `Spiral\Http\Diactoros\UriFactory`

To adapt to this change, you should switch to using `nyholm/psr7` for PSR-7 HTTP message interfaces. This involves
updating your application's dependencies and potentially adjusting any custom implementations that previously relied on
the now-removed Spiral classes. This shift aligns the Spiral with a widely adopted and community-supported PSR-7
implementation, facilitating interoperability and standardization.

### Impact: Medium

The Spiral has made a significant change to the `RendererInterface` for error handling. Specifically, the third argument
of the `renderException` method within `Spiral\Http\ErrorHandler\RendererInterface` has been modified. Previously
accepting a string for the error message, it now requires a `\Throwable` for the exception itself.

The updated method signature is as follows:

```php
interface RendererInterface
{
    public function renderException(Request $request, int $code, \Throwable $exception): Response;
}
```

This change means that when implementing the `RendererInterface`, your method must now handle a `\Throwable` object
instead of a simple string message. This adjustment allows for more detailed error handling and reporting, as the entire
exception object, with its stack trace and other details, can be used within the renderException method.

### Impact: Low

We don't use `laminas/diactoros` package anymore. Replace it with `nyholm/psr7`

### Impact: Low

The Spiral now supports the definition of custom input bags, offering greater flexibility in handling different types of
input data.

#### Example

The following example illustrates how to define a custom input bag for handling file uploads using Symfony's FilesBag.

```php
<?php

declare(strict_types=1);

namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;
use Spiral\Validation\Symfony\Http\Request\FilesBag;

class SampleBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

To access the files through the custom input bag, you can use the following method:

```php
$container->get(\Spiral\Http\Request\InputManager::class)->symfonyFiles('avatar');
```

This feature enhances the modularity and extensibility of input handling within the Spiral, enabling developers to
tailor the framework's request handling capabilities to the specific needs of their application.

> **See more** about HTTP input manager in the [documentation](../http/request-response.md#inputmanager)

### Impact: Low

There has been a change in the default behavior of the `InputBag` class regarding how it handles keys with `null`
values. This adjustment affects the way you check for the existence of keys and their values within an `InputBag`
instance.

Previously, the behavior of `has` and `isset` might have been implicitly expected to be consistent. However, with the
updated behavior, there is a distinction:

```php
use Spiral\Http\Request\InputBag;

$bag = new InputBag([4 => null]);

$bag->has(4); // true
isset($bag[4]); // false
```

This change underscores the difference between checking for the presence of a key in the bag (`has` method) versus
checking if a key is both present and has a non-null value (`isset`). The `has` method simply verifies the existence of
a key, regardless of its value, while `isset` checks for the key and also ensures that the value is not null.

### Other improvements

- Added AuthTransportMiddleware. See [PR](https://github.com/spiral/framework/pull/717)
- Added the ability to use middlewares via Autowire. See [PR](https://github.com/spiral/framework/pull/726)
- Improved route group middleware. See [PR](https://github.com/spiral/framework/pull/730)

## Bootloaders

### Impact: Medium

As part of the Spiral's ongoing evolution, several bootloaders have been relocated into their respective component
namespaces. This reorganization is aimed at improving modularity and making it easier to manage specific functionalities
within the framework. Below is a list of the moved bootloaders and their new locations:

- `Spiral\Bootloader\TokenizerBootloader` -> `Spiral\Tokenizer\Bootloader\TokenizerBootloader`
- `Spiral\Bootloader\AttributesBootloader` -> `Spiral\Attributes\Bootloader\AttributesBootloader`
- `Spiral\Bootloader\Distribution\DistributionBootloader` -> `Spiral\Distribution\Bootloader\DistributionBootloader`
- `Spiral\Bootloader\Distribution\DistributionConfig` -> `Spiral\Distribution\Config\DistributionConfig`
- `Spiral\Bootloader\Storage\StorageBootloader` -> `Spiral\Storage\Bootloader\StorageBootloader`
- `Spiral\Bootloader\Storage\StorageConfig` -> `Spiral\Storage\Config\StorageConfig`
- `Spiral\Bootloader\Security\ValidationBootloader` -> `Spiral\Validation\Bootloader\ValidationBootloader`
- `Spiral\Bootloader\Views\ViewsBootloader` -> `Spiral\Views\Bootloader\ViewsBootloader`
- `Spiral\Bootloader\Http\DiactorosBootloade` -> `Spiral\Http\Bootloader\DiactorosBootloader`

### Impact: High

There have been significant method renaming in the Spiral's bootloading process to better reflect their purpose and
functionality:

- The `boot` method has been renamed to `init`.
- The `start` method has been renamed to `boot`.

This change is aimed at clarifying the lifecycle stages of bootloaders. The `init` method is now used for initial setup
and configuration, while the `boot` method is meant for starting or enabling the functionalities provided by the
bootloader.

### Impact: Low

The Spiral now automatically bootloads all dependencies declared in the `init` and `boot` methods of bootloaders. This
means you no longer need to explicitly list bootloaders in the `DEPENDENCIES` constant for them to be recognized and
loaded by the framework.

For example, directly injecting a bootloader into the boot or init method:

```php
public function boot(AttributesBootloader $bootloader)
// or
public function init(AttributesBootloader $bootloader)
```

Is functionally equivalent to the previous approach of declaring dependencies:

```php
protected const DEPENDENCIES = [
    AttributesBootloader::class,
];
```

This enhancement simplifies the configuration of bootloaders by reducing redundancy and making the process more
intuitive.

> **See more** about bootloaders in the [documentation](../framework/bootloaders.md)

## Kernel

### Impact: Medium

The `Spiral\Boot\AbstractKernel::init` method has been removed. For initializing the application, the `create` method
should be used as a replacement. This change streamlines the process of setting up and running the Spiral application.

#### Example

To initialize and run your application, you should now use the following approach:

```php
App::create(
    directories: ['root' => __DIR__]
)->run();
```

### Impact: Medium

The `create` method of `Spiral\Boot\AbstractKernel` has been marked as final. This modification indicates that the
method cannot be overridden.

### Impact: Low

The naming of methods within `Spiral\Boot\AbstractKernel` has been updated to better reflect their functionality:

- The `starting` method has been renamed to `booting`.
- The `started` method has been renamed to `booted`.

### Other improvements

- Added a new callback for Kernel class. See [PR](https://github.com/spiral/framework/pull/720)

## Injectable config

### Impact: Medium

The Spiral has enhanced the `InjectableConfig` class to automatically merge predefined configurations specified in
the `$config` property of an `InjectableConfig` subclass with the configurations loaded from an external configuration
file. This improvement facilitates more flexible and dynamic configuration management by allowing predefined defaults to
be easily overridden or extended through external configuration files.

#### Example

Consider the following example where a custom configuration class extends the `InjectableConfig` class. The predefined
configuration within the class will be merged with the contents of an external configuration file:

```php
class SomeConfig extends \Spiral\Core\InjectableConfig
{
    // Predefined default configurations
    protected array $config = [   // <=== Will be merged with data from some-config.php
        'default' => 'sync',
        'aliases' => []
    ];
}
```

Suppose you have an external configuration file `app/config/some-config.php` with additional or overriding settings:

```php
// app/config/some-config.php

return [
    'aliases' => [...]
];
```

When the `SomeConfig` class is instantiated and fetched from the container, the resulting configuration will be a merged
version of the predefined `$config` array and the contents of the `some-config.php` file.

```php
var_dump($container->get(SomeConfig::class)->default); // sync
```

This mechanism ensures that default settings can be specified directly in the code while allowing for flexible overrides
or additions through external configuration files, thereby combining the benefits of both approaches for managing
application configurations.

> **See more** about injectable configuration in the [documentation](../framework/config.md)

## Data grid

### Impact: Low

The [spiral/data-grid](https://github.com/spiral/data-grid) package is no longer bundled with the Spiral Framework by
default. If your application relies on the Data Grid component, you will now need to include it explicitly as a
dependency.

To add the Data Grid component to your project, execute the following command:

```bash
composer require spiral/data-grid
```

For detailed instructions on how to install and use the Data Grid component as a standalone package, including its setup
and configuration, refer to the [documentation](../component/data-grid.md).

## Queue

### Impact: High

The Spiral has made a significant change to its queue system by removing the `pushCallable` method from
the `Spiral\Queue\QueueTrait`. This means that directly pushing objects and callables into a queue is no longer
supported by default, as the framework has ceased the out-of-the-box use of `opis/closure`.

### Other improvements

- Added the ability to configure serializers for different types of jobs.
  See [PR](https://github.com/spiral/framework/pull/749)
- Added the ability to set defaultSerializer for jobs. See [PR](https://github.com/spiral/framework/pull/768)

## Validation

The Spiral Framework has evolved its approach to validation by no longer providing a default validator as part of its
core package. This decision empowers developers to select the validation tool that best fits their project's needs,
ensuring greater flexibility and adaptability in application development.

Developers now have the opportunity to choose among several validation options, including:

- [spiral/validatior](../validation/spiral.md)
- [spiral-packages/symfony-validator](../validation/symfony.md)
- and [spiral-packages/laravel-validator](../validation/laravel.md)

or create their [own validator](../validation/factory.md)

### Impact: High

Currently, there is no default implementation for the `Spiral\Validation\ValidationInterface` within the framework. To
integrate validation into your Spiral project, you should select a validator from the options provided above or another
preferred validator, then install it via Composer. For example, to use Spiral's own validation package:

```bash
composer require spiral/validatior
```

After choosing and installing a validator, you must bind the `ValidationInterface` with your chosen implementation
within your application's bootloader.

```php
use Spiral\Validation\ValidationInterface;
use Spiral\Validation\ValidationProviderInterface;
use Spiral\Validator\FilterDefinition;

class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        ValidationInterface::class => [self::class, 'initValidation'],
    ];

    // ...

    public function initValidation(ValidationProviderInterface $provider): ValidationInterface
    {
        return $provider->getValidation(FilterDefinition::class);
    }
}
```

## spiral/monolog-bridge

### Impact: High

The constant `Spiral\Monolog\LogFactory::DEFAULT` has been deprecated in version 2.x of the Spiral Framework and
subsequently removed in version 3.x. As a replacement, use `Spiral\Monolog\Config\MonologConfig::DEFAULT_CHANNEL` for
referencing the default logging channel.

This change necessitates updates to your application's logging configuration or any code references that previously
relied on `LogFactory::DEFAULT`. Adjustments should be made to use `MonologConfig::DEFAULT_CHANNEL` to ensure
compatibility with Spiral Framework 3.x.

### Impact: Low

To configure the default Monolog channel, you can set an environment variable as follows:

```dotenv
# ...
MONOLOG_DEFAULT_CHANNEL=stderr
```

Additionally, you can specify this default channel and configure handlers and processors directly in the Monolog
configuration array within your application's config file:

```php
[
    'default' => env('MONOLOG_DEFAULT_CHANNEL', 'stderr'),
    'handlers' => [
        'default' => [
            // ...
        ],
        'stderr' => [
            // ...
        ],
        'stdout' => [
            // ...
        ],
    ],

    'processors' => [
        'default' => [
            // ...
        ],
        'stdout' => [
            // ...
        ],
    ],
]
```

> See [pull request](https://github.com/spiral/framework/pull/675)

## Deprecations and Removals

#### RoadRunner

In Spiral Framework 3.x, built-in integration with RoadRunner has been removed to streamline the core framework and
offer more flexibility in choosing server solutions. For projects requiring RoadRunner integration:

- Use the [spiral/roadrunner-bridge](https://github.com/spiral/roadrunner-bridge) package to seamlessly integrate
  RoadRunner with the Spiral Framework.

The following components have also been removed and replaced with their respective alternatives:

- `spiral/jobs` is no longer part of the Spiral Framework. Instead, use
  the [spiral/queue](https://spiral.dev/docs/queue-configuration/3.0/en) component for job queue management.
- `spiral/broadcast` has been replaced with
  the [spiral/broadcasting](https://spiral.dev/docs/component-broadcasting/3.0/en) component, offering enhanced
  functionality for broadcasting events.

#### CycleORM, Database, Migrations

Spiral Framework 3.x has decoupled its direct integration with `cycle/orm`, `cycle/database`, and `cycle/migrations`,
allowing for a more modular architecture. To integrate CycleORM into your Spiral project:

- Employ the [spiral/cycle-bridge](https://github.com/spiral/cycle-bridge) package, which facilitates the integration of
  CycleORM and related components.

## New Exception Handler

The Spiral Framework now centralizes all exception handling through `Spiral\Exceptions\ExceptionHandler`, eliminating
the need for additional snapshotter and logger components for exception management within the codebase.

The updated `ExceptionHandler` supports the registration of custom exception renderers and reporters for specific
environments, enhancing flexibility and control over exception handling and reporting.

> **See more** about exception handling in the [documentation](../basics/errors.md)

## Filters

### Impact: Low

Added a new method `Spiral\Filters\InputInterface::hasValue`

### Impact: High

The Filters component has undergone a complete redesign. To utilize filters from Spiral version 2.x within the 3.0
environment, please incorporate the [spiral/filters-bridge](https://github.com/spiral/filters-bridge) package.

### Redesigned filters

1. The dependency on the `spiral/validation` component for validators is no longer necessary.
2. Filters now leverage PHP attributes for mapping input data.
3. Enables automatic validation during container resolution, as required.
4. Compatibility with third-party validators is now supported, including:
    - [spiral/validator](https://spiral.dev/docs/security-validator/3.0/en) (corrected from `validatior`)
    - [spiral-packages/symfony-validator](https://github.com/spiral-packages/symfony-validator)
    - [spiral-packages/laravel-validator](https://github.com/spiral-packages/laravel-validator)

#### Filter Example

The following example demonstrates the use of the redesigned filters:

```php
use Spiral\Filters\Attribute\Input;
use Spiral\Filters\Attribute\NestedArray;
use Spiral\Filters\Attribute\NestedFilter;
use Spiral\Filters\Attribute\Setter;

class CreateUserFilter extends SymfonyFilter 
{
    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Post]
    public string $username;

    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Post(key: 'first_name')]
    public string $name;

    #[Asset\NotBlank]
    #[Asset\Length(['min' => 30])]
    #[Input\Query(key: 'last_name')]
    public string $lastName;

    #[Input\Query(key: 'page_num')]
    public int $page;

    #[Asset\NotBlank]
    #[Input\Attribute]
    private string $wsPath;

    #[Asset\NotBlank]
    #[Input\BearerToken]
    public string $token;

    #[Input\Cookie(key: 'utm_source')]
    public string $utmSource;

    #[Input\Cookie]
    public string $utmId;

    #[NestedArray(
        class: UserTagsFilter::class,
        input: new Inout\Post('tags')
    )]
    public array $tags = [];

    #[NestedFilter(class: UtmFilter::class)]
    public UtmFilter $utm;

    #[Input\Post]
    #[Setter(filter: 'md5')]
    public string $password;
}
```

```php
class UserController
{
    public function createUser(CreateUserFilter $filter): array
    {
        $user = new User(
            $filter->username,
            $filter->firstName,
            $filter->lastName
        );

        $user->setUtmData(
            $filter->utm->id,
            $filter->utm->source,
            ...
        );

        foreach($filter->tags as $tag) {
            $user->addTag($tag->role);
        }

        $this->em->persist($user);
    }
}
```

> **See more** about filters in the [documentation](../filters/configuration.md)

## Injectable enums

Imagine you need to determine if your application is running in production mode. A simple approach is to check
if `$env->get('APP_ENV') === 'production'`. However, enums offer a more convenient and type-safe alternative.

```php
enum AppEnvironment: string
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }
}
```

When the container resolves the enum class, it will provide an enum object with the value defined in the environment.

```php
class MigrateCommand extends Command 
{
    const NAME = '...';

    // Resolve and detect current environment
    public function perform(AppEnvironment $appEnv): int
    {
        if ($appEnv->isProduction()) {
            // Deny
            return 0;
        }

        // ...
        return 1;
    }
}
```

To utilize this feature, implement `Spiral\Boot\Injector\InjectableEnumInterface` and use
the `Spiral\Boot\Injector\ProvideFrom` attribute to specify a method that returns the specific enum object.

```php
<?php

declare(strict_types=1);

namespace Spiral\Boot\Environment;

use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum DebugMode implements InjectableEnumInterface
{
    case Enabled;
    case Disabled;

    public function isEnabled(): bool
    {
        return $this === self::Enabled;
    }

    public static function detect(EnvironmentInterface $environment): self
    {
        return \filter_var($environment->get('DEBUG'), \FILTER_VALIDATE_BOOL) ? self::Enabled : self::Disabled;
    }
}
}
```

This approach enhances type safety and code clarity by leveraging enums for environment-specific logic.

> **See more** about injectable enums in the [documentation](../container/injectors.md#enum-injectors)

## New serializer component

The Spiral Framework has introduced a new component, `spiral/serializer`, designed for serialization and deserialization
of payloads. This component is flexible, supporting custom drivers for various serialization formats.

### Configuration

To configure the serializer component, you need to define your preferred serializers in the `app/config/serializer.php`
file:

```php
// app/config/serializer.php
use Spiral\Core\Container\Autowire;

return [
    'default' => env('DEFAULT_SERIALIZER_FORMAT', 'json'),
    'serializers' => [
        'json' => JsonSerializer::class,
        'yaml' => new YamlSerializer(),
        'custom' => new Autowire(CustomSerializer::class),
    ],
];
```

Below is an example of a `Queue` class utilizing the serializer component to serialize payloads before pushing them to a
pipeline:

```php
class Queue 
{
    public function __construct(
        private readonly PipelineInterface $pipeline,
        private readonly \Spiral\Serializer\SerializerInterface $serializer,
    ) {}

    public function push(array $payload): void
    {
        $this->pipeline->push(
            $this-serializer->serialize($payload),
        );
    }
}
```

In scenarios where you need to dynamically select the serialization format, you can use the `SerializerManager` to
retrieve the appropriate serializer based on options provided at runtime:

```php
class Queue 
{
    public function __construct(
        private readonly PipelineInterface $pipeline,
        private readonly \Spiral\Serializer\SerializerManager $manager
    ) {}

    public function push(array $payload, Options $options): void
    {
        // $options->format = 'json'
        // $options->format = 'xml'
        // ...

        $this->pipeline->push(
            $this-manager
                ->getSerializer($options->format)
                ->serialize($payload)
        );
    }
}
```

> **See more** about the serializer component in the [documentation](../advanced/serializer.md)

## Console improvements

#### Command signature

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

#### Named sequences

The Spiral now supports the creation of named sequences, allowing for the organization and reuse of command sequences.

```php
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Console\Bootloader\ConsoleBootloader;

class MyPackageAssetsBootloader extends Bootloader
{
    public function init(ConsoleBootloader $console): void
    {
        $console->addSequence('publish:assets', PublishCssFilesCommand::class);
        $console->addSequence('publish:assets', PublishJsFilesCommand::class);
    }
}
```

Using the named sequences:

```php
use Spiral\Console\Command\SequenceCommand;
use Psr\Container\ContainerInterface;
use Spiral\Console\Config\ConsoleConfig;

class PublishAssetsCommand extends SequenceCommand
{
    public function perform(ConsoleConfig $config, ContainerInterface $container): int
    {
        $this->info("Publishing assets:");
        $this->newLine();

        return $this->runSequence(
            $config->getSequence('publish:assets'), 
            $container
        );
    }
}
```

#### `__invoke` method

Commands can now utilize the `__invoke` method as an alternative to the `perform` method, simplifying the command
definition.

```php
class CheckHttpCommand extends Command 
{
    // ...

    public function __invoke(): int
    {
        // ...
    }
}
```

#### Application in production confirmation

A new feature ensures confirmation prompts are shown when attempting to perform sensitive operations in a production
environment.

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

#### New helper methods

A suite of helper methods has been introduced to facilitate various command line interactions:

```php
class SomeCommand extends Command 
{
    // ...

    public function perform(): int
    {
        // Determine if the input option is present.
        $this->hasOption(...);

        // Determine if the input argument is present.
        $this->hasArgument(...);

        // Ask for confirmation.
        $status = $this->confirm('Are you sure?', default: false): bool;

        // Ask a question.
        $status = $this->ask('Are you sure?', default: 'no'): mixed;

        // Prompt the user for input but hide the answer from the console.
        $status = $this->secret('User password'): string;

        // Write a message as information output.
        $this->info(...);

        // Write a message as comment output.
        $this->comment(...);

        // Write a message as question output.
        $this->question(...);

        // Write a message as error output.
        $this->error(...);

        // Write a message as warning output.
        $this->warning(...);

        // Write a message as alert output.
        $this->alert(...);

        // Write a message as standard output.
        $this->line(..., 'error');

        // Write a blank line.
        $this->newLine();

        $this->newLine(count: 5);
    }
}
```

#### Console command interceptors

Developers can now register custom interceptors for console commands, enhancing the control over command execution and
lifecycle.

> **See more** about console interceptors in the [documentation](../console/interceptors.md)

#### Output object replacing

It is now possible to replace the output object in console commands, allowing for customized output handling.

```php
use Symfony\Component\Console\Input\InputInterface;
use Symfony\Component\Console\Output\OutputInterface;

abstract class MyCommand extends \Spiral\Console\Command 
{
    protected function prepareOutput(InputInterface $input, OutputInterface $output): OutputInterface
    {
        return new \App\Console\MyOutput($input, $output);
    }
}
```

## Container refactoring

The Spiral Framework's container system has undergone significant refactoring, enhancing its flexibility and
capabilities. Below is a comprehensive guide detailing these improvements.

> **See more** about the container in the [documentation](../container/overview.md)

### Modular Container Design

The container now operates on a modular basis, allowing for the replacement of individual modules with custom
implementations before the container's initialization. This change introduces greater flexibility in adapting the
container to specific project needs.

```php
// Initialize shared container, bindings, directories and etc.
$app = App::create(
    directories: ['root' => __DIR__],
    container: new Spiral\Core\Container(
        config: new Spiral\Core\Config(
            resolver: App\CustomResolver::class
        )
    )
)->run();
```

### PHP 8.0 Union Types Support

Support for PHP 8.0 Union types has been added, allowing for more expressive type declarations in your application's
codebase.

### Variadic Arguments Support

The container now supports variadic arguments, providing more flexibility in how arguments are passed to methods and
functions:

- Arguments can be passed by parameter name, including both named and positional arguments within an array.
- Values can also be passed directly by parameter name.
- Positional trailing values are supported, enhancing how methods receive variable numbers of arguments.

### Other improvements

1. Added support for referenced parameters in the Resolver;
2. Support for a default object value;
3. The Factory is now more strict: no more arguments type conversion;
4. Added public method for arguments validation;
5. Support for `WeakReference` bindings;

## Tokenizer

Since version 3.x the Spiral Framework has introduced tokenizer listeners to improve overall performance.

> **See more** about the tokenizer listeners in
> the [documentation](../advanced/tokenizer.md#efficient-scanning-with-class-listeners)

## Mailer

Since version 3.x the Spiral Framework has introduced ability to use custom transports for Symfony Mailer.

> **See more** about the transport in the [documentation](../advanced/sendit.md#custom-mailer-transport)

## Packages for Spiral framework 3.x

The Spiral 3.x ecosystem is enriched with a variety of packages designed to extend its functionality and facilitate
development. Below is an overview of key packages available for Spiral Framework 3.0:

#### [`spiral/roadrunner-bridge:2.0`](https://github.com/spiral/roadrunner-bridge)

This package integrates Spiral with RoadRunner, supporting various plugins to enhance your application's capabilities,
including:

- HTTP
- Jobs
- KV (Key-Value storage)
- GRPC
- TCP
- Websockets

#### [`spiral/cycle-bridge:2.0`](https://github.com/spiral/cycle-bridge)

Provides a bridge to Cycle ORM v2, enabling powerful object-relational mapping within Spiral. Supported packages
include:

- CycleOrm
- Database
- Migrations

#### [`spiral/temporal-bridge:2.0`](https://github.com/spiral/temporal-bridge)

Offers integration with Temporal for writing and running reliable cloud applications. Temporal provides a robust
platform for building scalable applications.

#### [`spiral/testing:2.0`](https://github.com/spiral/testing)

A testing SDK designed specifically for the Spiral, facilitating easy and efficient testing of your applications.

#### [`spiral-packages/database-seeder`](https://github.com/spiral-packages/database-seeder)

This package provides the capability to seed your database with data using seed classes, enhancing the data setup
process for development and testing.

#### [`spiral-packages/notifications:2.0`](https://github.com/spiral-packages/notifications)

Enables sending notifications across various channels from the Spiral, simplifying the process of keeping users
informed.

#### [`spiral-packages/scheduler:2.0`](https://github.com/spiral-packages/scheduler)

Assists in managing scheduled tasks on your server, providing a streamlined way to handle cron jobs and other scheduled
operations.

#### [`spiral-packages/cqrs:2.0`](https://github.com/spiral-packages/cqrs)

Implements the Command/Query Responsibility Segregation (CQRS) pattern within the Spiral, facilitating clear separation
of command and query operations.

#### [`spiral-packages/signed-urls:1.0`](https://github.com/spiral-packages/signed-urls)

Facilitates the creation and validation of signed URLs in Spiral, offering a secure way to generate and verify access
links.