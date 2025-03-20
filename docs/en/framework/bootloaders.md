# Framework — Bootloaders

Spiral uses bootloader classes during the bootstrapping process to handle application configuration. One of the key
features is that bootloaders are executed only once during the application bootstrapping, ensuring that adding
additional code to them does not negatively impact runtime performance.

Bootloader classes play a crucial role in configuring various aspects of the application.

**Here are some examples:**

- **Container Configuration**: Bootloaders can be utilized to configure the dependency injection container. This
  includes tasks such as registering services and binding interfaces to their respective implementations.
- **Application Configuration**: Bootloaders are responsible for setting up various application-level settings,
  including the environment, debug mode, and error handling.
- **Database Configuration**: Bootloaders are involved in setting up and configuring the application's database
  connection. This includes specifying the database driver, connection details, and any necessary migrations.
- **Routing Configuration**: The bootloaders can be used to set up the application's routing rules, such as defining
  which controllers should handle which URLs.
- **Services Initialization**: Bootloaders take care of initializing any services required by the application, such as
  caching, logging, and event systems.

![Application Control Phases](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## Simple Bootloader

To create a simple bootloader effortlessly, use the scaffolding command:

```terminal
php app.php create:bootloader GithubClient
```

> **Note**
> Read more about scaffolding in the [Basics — Scaffolding](../basics/scaffolding.md#bootloader) section.

After executing this command, the following output will confirm the successful creation:

```output
Declaration of '[32mGithubClientBootloader[39m' has been successfully written into '[33mapp/src/Application/Bootloader/GithubClientBootloader.php[39m'.
```

Now you can find the `GithubClientBootloader` class in the `app/src/Application/Bootloader` directory.

Currently, a new Bootloader doesn't perform any actions. A little later, we will add some functionality to it.

## Registering bootloader

Every bootloader must be activated in your application kernel `app/src/Application/Kernel.php`.

In Spiral, you can register bootloaders in your kernel using two different approaches: constants and methods.

1. **Constants:** The kernel provides constants such as `Kernel::SYSTEM`, `Kernel::LOAD`, and `Kernel::APP`. These
   constants allow you to specify which bootloaders should be executed at different stages of the application's
   initialization process. By using these constants, you can achieve a clear separation of concerns and easily
   understand which bootloaders handle specific tasks.
2. **Methods:** The kernel also offers methods like `defineBootloaders`, `defineAppBootloaders`,
   and `defineSystemBootloaders`. These methods allow for more complex logic and registration of objects, anonymous
   classes, or other advanced use cases. With these methods, you have greater flexibility and control over the
   initialization process of your application.

> **Warning**
> You cannot use both methods and constants simultaneously. If you choose to use the method approach, the registration
> through constants will not take effect.

:::: tabs

::: tab Using methods

Add the class reference into `defineBootloaders` or `defineAppBootloaders` methods of your `App\Application\Kernel`

class:

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
           RoutesBootloader::class,
        ];
    }

    public function defineAppBootloaders(): array
    {
        return [
           LoggingBootloader::class,
           MyBootloader::class,
           
           // anonymous bootloader via object instance
           new class extends Bootloader {
               // ...
           },
       ];
    }
}
```

> **Note**
> Bootloaders in the `defineAppBootloaders` method are always loaded after bootloaders in the `defineBootloaders`
> method. Keep domain-specific bootloaders in it.

:::

::: tab Using constants

Add the class reference into `LOAD` or `APP` constants of your `App\Application\Kernel` class:

```php app/src/Application/Kernel.php
namespace App\Application;

use App\Application\Bootloader\RoutesBootloader;
use App\Application\Bootloader\LoggingBootloader;
use App\Application\Bootloader\MyBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    protected const LOAD = [
        // ...
        RoutesBootloader::class,
    ];

    protected const APP = [
        // ...
        LoggingBootloader::class,
        MyBootloader::class,
    ];
}
```

> **Note**
> Bootloaders in the `APP` constant are always loaded after bootloaders in the `LOAD` constant. Keep domain-specific
> bootloaders in it.

:::

::::

### Bootloader loading control

There is also a feature that lets developers manage how bootloaders are set up and used. This functionality is
particularly advantageous for adapting applications to various environments, such as HTTP, command-line
interfaces, or any other contexts. It allows for the selective activation or deactivation of bootloaders based on the
specific requirements of the environment, enhancing both efficiency and performance.

There is a new DTO class `Spiral\Boot\Attribute\BootloadConfig` which enables the inclusion or exclusion of bootloaders,
passing parameters that will be forwarded to the init and boot methods of the bootloader, and dynamically adjusting the
bootloader loading based on environment variables.

To use it in the Kernel, you should employ the full class name of the bootloader as the key in the array of bootloaders,
with the corresponding value being a `BootloadConfig` object.

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => new BootloadConfig(allowEnv: ['APP_ENV' => ['local', 'dev']]),
            // ...
        ];
    }
}
```

In this example, we specified that the `PrototypeBootloader` should be loaded only if the environment variable `APP_ENV`
is defined and has a value of `local` or `dev`.

Instead of creating a `BootloadConfig` object directly, you can define a function that returns a `BootloadConfig`
object. This function can take arguments, which might be obtained from the container.

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Prototype\Bootloader\PrototypeBootloader;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // ...
            PrototypeBootloader::class => static fn (AppEnvironment $env) => new BootloadConfig(enabled: $env->isLocal()),
            // ...
        ];
    }
}
```

You can also use `BootloadConfig` class as an attribute to control how a bootloader behaves. This method is particularly
useful because it allows you to set up the configuration directly in the bootloader's class, making it more
straightforward and easier to understand.

**Here's a simple example of how you can use an attribute to configure a bootloader:**

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[BootloadConfig(allowEnv: ['APP_ENV' => 'local'])]
final class SomeBootloader extends Bootloader
{
}
```

Attributes are a great choice when you want to keep the configuration close to the bootloader's code. It's a more
intuitive way to set up bootloaders, especially in cases where the configuration is straightforward and doesn't require
complex logic.

#### Extending for Custom Preconditions

By extending `BootloadConfig`, you can create custom classes that encapsulate specific conditions under which
bootloaders should operate. This approach simplifies the usage of bootloaders by abstracting the configuration details
into these custom classes.

```php app/src/Application/Bootloader/TargetRRWorker.php
namespace App\Application\Bootloader;

use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

class TargetRRWorker extends BootloadConfig 
{
    public function __construct(array $modes)
    {
        parent::__construct(
            env: ['RR_MODE' => $modes],
        );
    }
}
```

Now you can use it in your bootloaders

```php app/src/Application/Bootloader/SomeBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[TargetRRWorker(modes: ['http', 'grpc'])]
final class SomeBootloader extends Bootloader
{
}
```

or use in Kernel:

```php app/src/Application/Kernel.php
namespace App\Application;

use Spiral\Framework\Kernel;

class Kernel extends Kernel
{
    public function defineBootloaders(): array
    {
        return [
            HttpBootloader::class => new TargetRRWorker(['http']),
            RoutesBootloader::class => new TargetRRWorker(['http']),

            GrpcBootloader::class => new TargetRRWorker(['grpc']),

            TemporalBootloader::class => new TargetRRWorker(['temporal']),
            
            // Other bootloaders...
        ];
    }
}
```

The ability to extend `BootloadConfig` opens up a world of possibilities for customizing the behavior of bootloaders.

## Available methods

Bootloaders provide two methods `init` and `boot` that are executed when the application is initialized.

### Init method

This method will be run **first**.

It's a good idea to set default values for the config files before proceeding. You can use the special bootloader
methods to modify the config files as needed. After that, you can go ahead and run any other logic that doesn't require
reading the config files and isn't dependent on code execution in the `init` and `boot` methods of other bootloaders.

This is also a good time to add initialization callbacks and configure container bindings, as long as it doesn't require
accessing the app config.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {
    }

    public function init(EnvironmentInterface $env): void 
    {
        $this->config->setDefaults(
            GithubConfig::CONFIG,
            [
                'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
                'secret' => $env->get('GITHUB_SECRET'),
            ]
        );
    }
}
```

> **Note**
> 1. Learn more about the `Spiral\Boot\EnvironmentInterface`, in the [Configuration](../start/configuration.md) section.
> 2. Learn more about the `Spiral\Boot\AbstractKernel` class (also known as the 'Kernel'), in
     the [Kernel and Environment](../framework/kernel.md) section.
> 3. Learn more about the `Spiral\Config\ConfiguratorInterface` class, in the [Config Objects](../framework/config.md)
     section.

### Boot method

This method will be run after the `init` method in all the bootloaders have been executed. The reason for this is that
you might need the results of bootloader initialization in order to proceed. For example, compiled configuration files.

Just keep in mind that it should be run after all those `init` methods have completed.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use App\Service\Github\GithubConfig;

final class GithubClientBootloader extends Bootloader
{
    // See code above ...
    
    public function boot(GithubConfig $config): void 
    {
        $token = $config->getAccessToken();
        // ...
    }
}
```

## Configuring Container

Bootloaders are usually used to set up a container, like if we want to link multiple implementations to their
interfaces or create some service. We can use both `init` or `boot` method for this, which lets us request any services
we need using method injection.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    // See code above ...
    
    public function boot(BinderInterface $binder): void 
    {
        $binder->bindSingleton(
            ClientInterface::class, 
            static fn (GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            )
        );
    }
}
```

> **Note**
> The closure is provided as an argument to the `bindSingleton` method will be called by the dependency injection
> (DI) container when it needs to create an instance of `MyService`. When the closure is called, the DI container will
> automatically resolve and inject any dependencies that are required by the closure.
>
> If you want to learn more about DI, you can check out the [Container and Factories](../container/overview.md) section
> of the documentation. It should have all the info you need.

Bootloaders also provide the ability to simplify container binding definition

:::: tabs

::: tab Using methods

You can use the `defineBindings` and `defineSingletons` methods to define container bindings in a declarative way.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    public function defineSingletons(): array
    {
        return [
            MyInterface::class => MyClass::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            ClientInterface::class => [self::class, 'createClient'],
            
            // or
            
            ClientInterface::class => static fn(GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            );
        ];
    }

    // See code above ...
    
    public function createClient(GithubConfig $config): ClientInterface 
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::: tab Using constants

You can use the `BINDINGS` and `SINGLETONS` constants to define container bindings in a declarative way.

```php app/src/Application/Bootloader/GithubClientBootloader.php
namespace App\Application\Bootloader;

// ...
use Spiral\Core\BinderInterface;
use App\Service\Github\GithubConfig;
use App\Service\Github\ClientInterface;
use App\Service\Github\Client;

final class GithubClientBootloader extends Bootloader
{
    const BINDINGS = [
        MyInterface::class => MyClass::class
    ];
    
    const SINGLETONS = [
        ClientInterface::class => [self::class, 'createClient'],
    ];
    
    // See code above ...
    
    public function createClient(GithubConfig $config): ClientInterface 
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
}
```

:::

::::

## Configuring Application

Another common use case of bootloaders is to configure the framework before the application launch. For example, we can
declare a new route for our application or module:

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;
use Spiral\Router\Route;

final class RoutesBootloader extends Bootloader 
{
    public function boot(RouterInterface $router): void
    {
        $router->setRoute(
            'my-route',
            new Route('/<action>', new Controller(MyController::class))
        );
    }
}
```

> **Note**
> You are only able to use bootloaders to configure your components during the bootstrap phase (a.k.a. via another
> bootloader). The framework would not allow you to change any configuration value after component initialization.

## Depending on other Bootloaders

Depending on other bootloaders can be really useful in certain situations. For example, if you want to make sure a
certain bootloader is initialized before yours, you can use one of two main approaches: injecting the bootloader class
into the init or boot method as an argument, or using the `Bootloader::DEPENDENCIES` constant in your bootloader class.

This can be a good way to manage the initialization of your app and make sure all the necessary resources and
dependencies are available when you need them. Just keep in mind that dependent bootloaders will only be initialized
once, even if they are depended on by multiple other bootloaders.

Some framework bootloaders can be used as a simple way to configure application settings. For example, we can
use `Spiral\Bootloader\Http\HttpBootloader` to add global PSR-15 middleware:

**There are two ways to define dependent bootloaders:**

1. Injecting the bootloader class into the `init` or `boot` method as an argument of your bootloader class

**For example:**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\Http\HttpBootloader;
use App\Middleware\MyMiddleware;

class MyBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        $http->addMiddleware(MyMiddleware::class);
    }
}
```

2. Using the `Bootloader::DEPENDENCIES` constant in your bootloader class. This can be a convenient way to define
   dependent bootloaders when you don't need to access them directly in your bootloader class.

**For example:**

```php app/src/Application/Bootloader/MyBootloader.php
namespace App\Application\Bootloader;

class MyBootloader extends Bootloader 
{
    protected const DEPENDENCIES = [
        \Spiral\Bootloader\Http\HttpBootloader::class
    ];
    
    public function boot(): void
    {
        // ...
    }
}
```

Spiral will automatically resolve and initialize the dependent bootloader before the depending bootloader
is initialized.

Both of these approaches allow you to define dependent bootloaders in a declarative way, which can make it easier to
manage the initialization of your application and ensure that all necessary resources and dependencies are available
when they are needed.

> **Note**
> Dependent bootloaders will be only initialized once, even if multiple other bootloaders depend on them.

## Cascade bootloading

You can control the bootload process using Bootloader itself, simply request `Spiral\Boot\BootloadManager`:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\BootloadManager;
use Spiral\Bootloader\DebugBootloader;
use Spiral\Boot\EnvironmentInterface;

class AppBootloader extends Bootloader
{
    public function boot(BootloadManager $bootloadManager, EnvironmentInterface $env): void
    {
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class
            ]);
        }
    }
}
```

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [HTTP — Interceptors](../http/interceptors.md)
* [Scaffolding](../basics/scaffolding.md)
