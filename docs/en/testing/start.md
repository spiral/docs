# Testing — Getting Started

Spiral is designed with a strong emphasis on testing. It comes with built-in support
for [PHPUnit](https://phpunit.de/), and includes a pre-configured `phpunit.xml` file for your application.

The default directory structure of a Spiral application includes a `tests` directory, which contains two
subdirectories: `Feature` and `Unit`.

**Unit** tests are designed to test small, isolated portions of code, often focusing on a single method. These tests do
not boot the entire Spiral application, and **therefore cannot access the database or other framework services**.

On the other hand, **Feature** tests are intended to test a larger portion of code, including interactions between
multiple objects, or even a full HTTP request to a JSON endpoint. These tests provide more comprehensive coverage of
your application and give you greater confidence that it is functioning as intended.

Spiral provides the `spiral/testing` package to help developers with the testing of their application. This package
offers a variety of helper methods that can simplify the process of writing tests for Spiral applications.

## Configuration

In order to run tests, you may need to set up certain environment variables. One way to do this is by using
the `phpunit.xml` file. This file is used by the PHPUnit testing framework to configure the testing
environment.

**Here is an example:**

```xml phpunit.xml

<phpunit>
    // ...
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="MONOLOG_DEFAULT_CHANNEL" value="stderr"/>
        <env name="CACHE_STORAGE" value="array"/>
        <env name="TOKENIZER_CACHE_TARGETS" value="true" />
        <env name="CYCLE_SCHEMA_CACHE" value="true" />
        // ...
    </php>
</phpunit>
```

> **Warning**
> When you run tests in a Docker container, the settings in Docker are more important than the ones in `phpunit.xml`. 
> This might cause trouble if they don't match. Make sure the Docker settings are right for your tests.

> **Notice**
> It is highly advised to enable `TOKENIZER_CACHE_TARGETS` and `CYCLE_SCHEMA_CACHE` for enhanced test performance. By 
> doing so, you allow caching of tokenizer and ORM schema which means they won't be executed with each test iteration. 
> However, please note it's important to clear this cache whenever you make changes to your code or the schema of your 
> entities to ensure tests run with up-to-date configurations.

## Unit tests

On the other hand, Unit tests, which focus on small, isolated portions of your code, should extend the
`PHPUnit\Framework\TestCase` class. Unit tests do not boot the entire Spiral application, so they are not able to access
the database or other framework services.

**Here is an example of a simple unit test:**

```php tests/Unit/UserTest.php
namespace Tests\Unit;

use PHPUnit\Framework\TestCase;

final class UserTest extends TestCase
{
    public function testGetId(): void
    {
        $user = new User(id: 101);
        $this->assertSame(101, $user->getId());
    }
}
```

## Feature tests

Feature tests, which test a larger portion of your code, should extend the `Tests\TestCase` abstract class. This class
is specifically designed to bootstrap your application, simulating the behavior of a web server, but with testing
environment variables.

**Here is an example of a simple feature test:**

```php tests/Feature/Controller/UserController/ShowActionTest.php
namespace Tests\Feature\Controller\UserController;

use Tests\TestCase;

final class ShowActionTest extends TestCase
{
    public function testShowPageNotFoundIfUserNotExist(): void
    {
        $http = $this->fakeHttp();
        $response = $http->get('/user/1');
        $response->assertNotFound();
    }
}
```

### Environment variables

The `Tests\TestCase` class includes a feature that allows you to set up environment variables for your test cases. This
can be useful if you need to test specific behavior based on environment variables.

```php
use Tests\TestCase;

final class SomeTest extends TestCase
{
    public const ENV = [
        'DEBUG' => false,
        // ...
    ];

    public function testSomeFeature(): void
    {
        //
    }
}
```

You can also define ENV variables using PHP attributes. This allows for more granular control over the test environment.

```php
use Tests\TestCase;
use Spiral\Testing\Attribute\Env;

final class SomeTest extends TestCase
{
    #[Env('DEBUG', false)]
    #[Env('APP_ENV', 'production')]
    public function testSomeFeature(): void
    {
        //
    }
}
```

----

### Interaction with Container

The `Tests\TestCase` class also includes a feature that allows you to interact with the container.

#### Getting a container instance

```php
$container = $this->getContainer();
````

#### Check if a service is not registered in a container

```php
$this->assertContainerMissed(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

#### Check if a container can create an object using autowiring

```php
$this->assertContainerInstantiable(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

#### Check if container has given alias and bound

```php
$this->assertContainerBound(\Spiral\Queue\QueueConnectionProviderInterface::class);
```

Check if container bound to a specific class

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

Check if container bound to a specific class with parameters

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class,
    ['foo' => 'bar']
);
```

You can also use a callback function for additional checks

```php
$this->assertContainerBound(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class,
    ['foo' => 'bar'],
    function(\Spiral\Queue\QueueManager $manager) {
        $this->assertEquals(..., $manager->....)
    }
);
```

#### Check if container bound as singleton

```php
$this->assertContainerBoundAsSingleton(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

#### Check if container bound as not singleton

```php
$this->assertContainerBoundNotAsSingleton(
    \Spiral\Queue\QueueConnectionProviderInterface::class,
    \Spiral\Queue\QueueManager::class
);
```

#### Mock a service in the container

In some cases you need to mock a service in the container. For example authentication service.

```php
namespace Tests\Feature\Controller\UserController;

use Tests\TestCase;

final class ProfileActionTest extends TestCase
{
    public function testShowPageNotFoundIfUserNotExist(): void
    {   
        $auth = $this->mockContainer(\Spiral\Auth\ActorProviderInterface::class);
        $auth->shouldReceive('getActor')->with(...)->once()->andReturnNull();

        $http = $this->fakeHttp();
        $response = $http->get('/user/profile');

        $response->assertNotFound();
    }
}
```

----

### Interaction with Config

Let's imagine that we have the following config:

```php app/config/http.php
return [
    'basePath'   => '/',
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8',
    ],
    'middleware' => [],
];
```

#### Define config values

In some cases you need to define config values for specific tests. You can do it using PHP attributes.

```php
use Tests\TestCase;
use Spiral\Testing\Attribute\Config;

final class SomeTest extends TestCase
{
    #[Config('http.basePath', '/custom')]
    #[Config('http.headers.Content-Type', 'text/plain')]
    public function testSomeFeature(): void
    {
        //
    }
}
````

#### Check if config matches the given value

```php
$this->assertConfigMatches('http', [
    'basePath'   => '/',
    'headers'    => [
        'Content-Type' => 'text/html; charset=UTF-8',
    ],
    'middleware' => [],
]);
```

#### Check if config has the given fragment

```php
$this->assertConfigHasFragments('http', [
    'basePath' => '/'
])
```

#### Get the config

```php
/** @var array $config */
$config = $this->getConfig('http');
```

----

### Interaction with Directories

#### Check if directory alias exists

```php
$this-assertDirectoryAliasDefined('root');
```

#### Check if directory alias matches the given value

```php
$this->assertDirectoryAliasMatches('runtime', __DIR__.'src/runtime');
```

#### Clean up directory

```php
$this->cleanupDirectories(
    __DIR__.'src/runtime/cache',
    __DIR__.'src/runtime/tmp'
);
```

#### Clean up directory by alias

```php
$this->cleanupDirectoriesByAliases(
    'runtime', 'cache', 'logs'
);
```

You can also clean up runtime directory

```php
$this->cleanUpRuntimeDirectory();
```

----

### Interaction with Console

#### Check if a command is registered

```php
$this->assertCommandRegistered('ping');
```

#### Run a console command

You can run a console command in your test case and check the result.

```php
$output = $this->runCommand('ping', ['site' => 'https://google.com']);
$this->assertStringContainsString('Pong', $output);
```

You can also check strings in the output using

```php
$this->assertConsoleCommandOutputContainsStrings(
    'ping',
    ['site' => 'https://google.com'],
    ['Site found', 'Starting ping ...', 'Success!']
);
```

----

### Interaction with Bootloaders

The `Tests\TestCase` class also includes a feature that allows you to interact with bootloaders.

#### Check if a bootloader is registered

```php
$this->assertBootloaderLoaded(\MyPackage\Bootloaders\PackageBootloader::class);
```

#### Check if a bootloader is not registered

```php
$this->assertBootloaderMissed(\MyPackage\Bootloaders\PackageBootloader::class);
```

----

### Interaction with Dispatcher

#### Check if a dispatcher is registered

```php
$this->assertDispatcherRegistered(HttpDispatcher::class);
```

#### Check if a dispatcher is not registered

```php
$this->assertDispatcherMissed(HttpDispatcher::class);
```

#### Run dispatcher

You can run a dispatcher with passing some bindings as second argument. It will be run inside scope with bindings.

```php
$this->serveDispatcher(HttpDispatcher::class, [
    \Spiral\Boot\EnvironmentInterface::class => new \Spiral\Boot\Environment([
        'foo' => 'bar'
    ]),
]);
```

#### Check if a dispatcher can be served

Check if a dispatcher can be served with current environment.

```php
$this->assertDispatcherCanBeServed(HttpDispatcher::class);
```

#### Check if a dispatcher cannot be served

Check if a dispatcher cannot be served with current environment.

```php
$this->assertDispatcherCannotBeServed(HttpDispatcher::class);
```

#### Get registered dispatchers

```php
/** @var class-string<\Spiral\Boot\DispatcherInterface>[] $dispatchers */
$dispatchers = $this->getRegisteredDispatchers();
```

----

### Interaction with Scaffolder

We can provide the ability to test scaffolder commands.

#### Assert generated code is same as expected

```php
$this->assertScaffolderCommandSame(
    'create:command',
    [
        'name' => 'TestCommand',
    ],
    expected: <<<'PHP'
    <?php
    
    declare(strict_types=1);
    
    namespace Spiral\Testing\Command;
    
    use Spiral\Console\Attribute\Argument;
    use Spiral\Console\Attribute\AsCommand;
    use Spiral\Console\Attribute\Option;
    use Spiral\Console\Attribute\Question;
    use Spiral\Console\Command;
    
    #[AsCommand(name: 'test:command')]
    final class TestCommand extends Command
    {
        public function __invoke(): int
        {
            // Put your command logic here
            $this->info('Command logic is not implemented yet');
    
            return self::SUCCESS;
        }
    }
    
    PHP,
    expectedFilename: 'app/src/Command/TestCommand.php',
    expectedOutputStrings: [
        "Declaration of 'TestCommand' has been successfully written into 'app/src/Command/TestCommand.php",
    ],
);
```

#### Assert that generated code contains the given string

```php
$this->assertScaffolderCommandContains(
    'create:command',
    [
        'name' => 'TestCommand',
        '--namespace' => 'App\Command',
    ],
    expectedStrings: [
        'namespace App\Command;',
    ],
    expectedFilename: 'app/src/TestCommand.php',
);
```

#### Assert exception is thrown

```php
$this->expectException(RuntimeException::class);
$this->expectExceptionMessage('Not enough arguments (missing: "name").');

$this->assertScaffolderCommandSame(
    'create:command',
    [],
    '',
);
```

#### Assert run command with additional options

```php
$this->assertScaffolderCommandContains(
    'create:command',
    [
        'name' => 'TestCommand',
        '-o' => 'foo',
    ],
    expectedStrings: [
        "#[Option(description: 'Argument description')]",
        'private bool $foo;'
    ],
);
```

----

## Running Tests

When you run the `vendor/bin/phpunit` command, it will automatically look for test files in the `tests` directory of
your application, and run them using the configuration specified in the `phpunit.xml` file.

**To run the tests in your Spiral application, you can simply execute the command:**

```terminal
./vendor/bin/phpunit
```

Enjoy testing!

<hr>

## What's next?

Now, dive deeper into the fundamentals by reading some articles:

* [Queue and Jobs](../queue/configuration.md)
* [Queue Interceptors](../queue/interceptors.md)