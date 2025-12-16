# Framework — Bootloaders

Bootloaders are classes that configure your application during startup. They run once during bootstrapping, so adding
code to them doesn't affect runtime performance.

**Common uses:**

- Register services and bind interfaces
- Configure environment, debugging, and error handling
- Set up database connections
- Define routing rules
- Initialize caching, logging, and event systems

![Application Control Phases](https://user-images.githubusercontent.com/773481/180768689-c711e6f0-3523-4330-a496-f78088504b29.png)

## Creating a Bootloader

Generate a bootloader using the scaffolding command:

```terminal
php app.php create:bootloader GithubClient
```

This creates `app/src/Application/Bootloader/GithubClientBootloader.php`.

> **See also**
> [Basics — Scaffolding](../basics/scaffolding.md#bootloader)

## Registering Bootloaders

Register bootloaders in `app/src/Application/Kernel.php`:

```php app/src/Application/Kernel.php
namespace App\Application;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            RoutesBootloader::class,
            // Framework bootloaders
        ];
    }

    public function defineAppBootloaders(): array
    {
        return [
            LoggingBootloader::class,
            MyBootloader::class,
            
            // You can even use anonymous classes
            new class extends Bootloader {
                // ...
            },
        ];
    }
}
```

> **Note**
> `defineAppBootloaders()` runs after `defineBootloaders()`. Keep your application-specific bootloaders there.

## Conditional Loading

The `BootloadConfig` class allows you to control when and how bootloaders are loaded. This is useful for
environment-specific features, optional components, or dynamic configuration.

### Configuration Options

| Parameter  | Type    | Default | Description                                                            |
|------------|---------|---------|------------------------------------------------------------------------|
| `args`     | `array` | `[]`    | Arguments passed to bootloader's constructor                           |
| `enabled`  | `bool`  | `true`  | Enable/disable the bootloader                                          |
| `allowEnv` | `array` | `[]`    | Environment variables that must match specified values (whitelist)     |
| `denyEnv`  | `array` | `[]`    | Environment variables that must NOT match specified values (blacklist) |
| `override` | `bool`  | `true`  | Whether runtime config (in Kernel) can override attribute config       |

### Basic Usage

Control bootloader loading based on environment:

```php app/src/Application/Kernel.php
use Spiral\Boot\Attribute\BootloadConfig;

class Kernel extends \Spiral\Framework\Kernel
{
    public function defineBootloaders(): array
    {
        return [
            // Load only in local/dev environments
            PrototypeBootloader::class => new BootloadConfig(
                allowEnv: ['APP_ENV' => ['local', 'dev']]
            ),
            
            // Disable in production
            DebugBootloader::class => new BootloadConfig(
                denyEnv: ['APP_ENV' => 'production']
            ),
            
            // Pass constructor arguments
            CacheBootloader::class => new BootloadConfig(
                args: ['driver' => 'redis', 'ttl' => 3600]
            ),
        ];
    }
}
```

### Using Closures

Use a closure to access container services when building configuration:

```php
use Spiral\Boot\Environment\AppEnvironment;

PrototypeBootloader::class => static fn(AppEnvironment $env) => 
    new BootloadConfig(
        enabled: $env->isLocal(),
        args: ['debug' => $env->get('DEBUG')]
    ),
```

**Use case:** Dynamic configuration based on complex application state or multiple services.

### Using Attributes

Define configuration directly on the bootloader class:

```php app/src/Application/Bootloader/DevToolsBootloader.php
use Spiral\Boot\Attribute\BootloadConfig;
use Spiral\Boot\Bootloader\Bootloader;

#[BootloadConfig(
    allowEnv: ['APP_ENV' => ['local', 'development']],
    denyEnv: ['TESTING' => true]
)]
final class DevToolsBootloader extends Bootloader
{
    public function __construct(
        private readonly bool $profiling = false
    ) {}
}
```

**Use case:** Self-contained bootloaders with built-in loading conditions.

### Configuration Priority

When a bootloader has both attribute and runtime configuration, the `override` parameter controls precedence:

```php
// In the bootloader
#[BootloadConfig(
    args: ['debug' => true],
    override: false  // Prevent runtime override
)]
final class MyBootloader extends Bootloader
{
    public function __construct(
        private readonly bool $debug
    ) {}
}

// In the Kernel - this will be IGNORED due to override: false
MyBootloader::class => new BootloadConfig(args: ['debug' => false]),
```

**Use case:** Enforce specific configuration for critical bootloaders that shouldn't be modified at runtime.

### Custom Configuration Classes

Create reusable configuration by extending `BootloadConfig`:

```php app/src/Application/Attribute/TargetRRWorker.php
use Spiral\Boot\Attribute\BootloadConfig;

/**
 * Load bootloader only for specific RoadRunner workers
 */
class TargetRRWorker extends BootloadConfig 
{
    public function __construct(array $modes)
    {
        parent::__construct(allowEnv: ['RR_MODE' => $modes]);
    }
}
```

Use in bootloaders:

```php
#[TargetRRWorker(modes: ['http', 'grpc'])]
final class ApiBootloader extends Bootloader {}
```

Or in Kernel:

```php
public function defineBootloaders(): array
{
    return [
        HttpBootloader::class => new TargetRRWorker(['http']),
        GrpcBootloader::class => new TargetRRWorker(['grpc']),
        TemporalBootloader::class => new TargetRRWorker(['temporal']),
    ];
}
```

**Use case:** Microservices or multi-protocol applications where different workers need different bootloaders.

### Environment Matching

Both `allowEnv` and `denyEnv` support multiple values:

```php
#[BootloadConfig(
    // Load if APP_ENV is local OR development
    allowEnv: ['APP_ENV' => ['local', 'development']],
    
    // Don't load if any of these are true
    denyEnv: [
        'TESTING' => [true, 1, 'true', 'yes'],
        'CI' => true
    ]
)]
final class DevToolsBootloader extends Bootloader {}
```

**How it works:**

- `allowEnv`: At least one condition must match (OR logic)
- `denyEnv`: If any condition matches, bootloader is skipped (OR logic)
- If both specified: `allowEnv` is checked first, then `denyEnv`

## Initialization Methods

Bootloaders provide multiple ways to execute initialization code. You can use traditional methods, attribute-based
methods, or combine both approaches.

### Traditional Methods

#### init()

The `init()` method runs first, before any bootloader's `boot()` method executes.

**Parameters:**

- Supports dependency injection - request any service through method parameters

**Use cases:**

- Set configuration defaults before other bootloaders read them
- Register container bindings that don't depend on configuration
- Initialize services that other bootloaders might need

```php
final class GithubClientBootloader extends Bootloader
{
    public function __construct(
        private readonly ConfiguratorInterface $config
    ) {}

    public function init(EnvironmentInterface $env): void 
    {
        // Set defaults before configuration is accessed
        $this->config->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
            'secret' => $env->get('GITHUB_SECRET'),
            'timeout' => 30,
        ]);
    }
}
```

> **See also**
> [Configuration](../start/configuration.md) • [Kernel](../framework/kernel.md) • [Config Objects](../framework/config.md)

#### boot()

The `boot()` method runs after all `init()` methods complete across all bootloaders.

**Parameters:**

- Supports dependency injection - request any service through method parameters
- Can receive compiled configuration objects

**Use cases:**

- Configure services using finalized configuration
- Register routes, middleware, event listeners
- Set up integrations between different components

```php
final class GithubClientBootloader extends Bootloader
{
    public function boot(
        GithubConfig $config,
        HttpBootloader $http
    ): void {
        // Configuration is now compiled and ready
        $client = new GithubClient($config->getAccessToken());
        
        // Other bootloaders are initialized
        $http->addMiddleware(GithubAuthMiddleware::class);
    }
}
```

### Attribute-Based Methods

Attribute-based methods give you fine-grained control over execution order and allow multiple initialization/boot
methods in a single bootloader.

#### #[InitMethod]

Methods marked with `#[InitMethod]` run during the initialization phase.

**Parameters:**

| Parameter  | Type  | Default | Description                            |
|------------|-------|---------|----------------------------------------|
| `priority` | `int` | `0`     | Execution priority (higher runs first) |

**Features:**

- Supports dependency injection
- Multiple `#[InitMethod]` methods per bootloader
- Execution order: priority 10 → 5 → 0 → -5 → -10

**Use cases:**

- Split initialization logic into focused methods
- Control exact initialization order across bootloaders
- Register groups of related bindings

```php
use Spiral\Boot\Attribute\InitMethod;

final class DatabaseBootloader extends Bootloader
{
    // Critical: runs first
    #[InitMethod(priority: 10)]
    public function registerDrivers(DatabaseManager $manager): void
    {
        $manager->addDriver('mysql', MySQLDriver::class);
        $manager->addDriver('postgres', PostgresDriver::class);
    }
    
    // Normal: runs after high priority
    #[InitMethod]
    public function registerConnections(Container $container): void
    {
        $container->bindSingleton(
            ConnectionInterface::class,
            DefaultConnection::class
        );
    }
    
    // Low priority: runs last
    #[InitMethod(priority: -10)]
    public function registerExtensions(): void
    {
        // Optional extensions that depend on core setup
    }
}
```

#### #[BootMethod]

Methods marked with `#[BootMethod]` run during the boot phase, after all initialization completes.

**Parameters:**

| Parameter  | Type  | Default | Description                            |
|------------|-------|---------|----------------------------------------|
| `priority` | `int` | `0`     | Execution priority (higher runs first) |

**Features:**

- Supports dependency injection
- Multiple `#[BootMethod]` methods per bootloader
- Access to fully configured services and compiled configuration

**Use cases:**

- Configure routes, middleware, or event listeners
- Set up cross-cutting concerns
- Register application-level services

```php
use Spiral\Boot\Attribute\BootMethod;

final class ApplicationBootloader extends Bootloader
{
    // Critical services first
    #[BootMethod(priority: 10)]
    public function configureErrorHandling(ErrorHandler $handler): void
    {
        $handler->addRenderer(new JsonErrorRenderer());
    }
    
    // Standard configuration
    #[BootMethod]
    public function configureRoutes(RouterInterface $router): void
    {
        $router->addRoute('home', new Route('/', HomeController::class));
    }
    
    // Non-critical features last
    #[BootMethod(priority: -10)]
    public function registerEventListeners(
        EventDispatcherInterface $dispatcher
    ): void {
        $dispatcher->addListener(
            ApplicationStarted::class, 
            fn() => $this->onStart()
        );
    }
}
```

### Execution Order

Understanding the complete execution sequence:

```
1. #[InitMethod(priority: 10)]  ← Highest priority init methods
2. #[InitMethod(priority: 0)]   ← Default priority init methods  
3. #[InitMethod(priority: -10)] ← Lowest priority init methods
4. init()                       ← Traditional init method
5. #[BootMethod(priority: 10)]  ← Highest priority boot methods
6. #[BootMethod(priority: 0)]   ← Default priority boot methods
7. #[BootMethod(priority: -10)] ← Lowest priority boot methods
8. boot()                       ← Traditional boot method
```

**This applies across ALL bootloaders**, meaning:

- All `#[InitMethod(priority: 10)]` methods execute before any `#[InitMethod(priority: 0)]`
- All `init()` methods execute before any `#[BootMethod]`
- You can mix traditional and attribute-based methods in the same bootloader

## Container Configuration

Bootloaders provide multiple ways to configure the dependency injection container. Choose the approach that best fits
your needs.

### Direct Configuration

Use the `BinderInterface` directly for maximum flexibility:

```php
use Spiral\Core\BinderInterface;

final class GithubClientBootloader extends Bootloader
{
    public function boot(BinderInterface $binder, GithubConfig $config): void 
    {
        // Singleton - created once, reused
        $binder->bindSingleton(
            ClientInterface::class, 
            static fn(GithubConfig $config) => new Client(
                $config->getAccessToken(),
                $config->getSecret(),
            )
        );
        
        // Factory - created each time
        $binder->bind(
            RequestInterface::class,
            static fn() => new Request()
        );
    }
}
```

> **See also**
> [Container and Factories](../container/overview.md)

### Declarative Bindings

Define bindings in a structured, declarative way.

#### Using Methods

**Available methods:**

| Method               | Returns | Description                               |
|----------------------|---------|-------------------------------------------|
| `defineBindings()`   | `array` | Factory bindings (new instance each time) |
| `defineSingletons()` | `array` | Singleton bindings (created once, reused) |

**Binding formats:**

```php
return [
    // Simple class binding
    InterfaceA::class => ClassA::class,
    
    // Method callback
    InterfaceB::class => [self::class, 'createServiceB'],
    
    // Closure with dependencies
    InterfaceC::class => static fn(Config $config) => new ServiceC($config),
];
```

**Example:**

```php
final class ServicesBootloader extends Bootloader
{
    public function defineBindings(): array
    {
        return [
            // New instance each resolution
            RequestInterface::class => Request::class,
            TokenGeneratorInterface::class => [self::class, 'createTokenGenerator'],
        ];
    }

    public function defineSingletons(): array
    {
        return [
            // Shared instance
            CacheInterface::class => RedisCache::class,
            
            // Lazy initialization with dependencies
            LoggerInterface::class => static fn(Config $config) => 
                new Logger($config->get('logging.channel')),
        ];
    }
    
    private function createTokenGenerator(): TokenGeneratorInterface
    {
        return new TokenGenerator(hash_algo: 'sha256', length: 32);
    }
}
```

**Use cases:**

- Clean, scannable binding definitions
- Group related bindings together
- Simple class-to-class or class-to-factory bindings

#### Using Attributes

Use PHP attributes for type-safe, self-documenting bindings with additional features like aliases and scopes.

##### #[SingletonMethod]

Creates a singleton binding - the method is called once, and the result is cached and reused.

**Parameters:**

| Parameter               | Type           | Default | Description                                      |
|-------------------------|----------------|---------|--------------------------------------------------|
| `alias`                 | `string\|null` | `null`  | Bind to this alias instead of return type        |
| `aliasesFromReturnType` | `bool`         | `false` | Also bind to return type when alias is specified |

**Use cases:**

- Services that maintain state
- Expensive-to-create objects (database connections, HTTP clients)
- Shared resources (cache, event dispatcher)

```php
use Spiral\Boot\Attribute\SingletonMethod;

final class ServicesBootloader extends Bootloader
{
    // Binds to return type: HttpClientInterface
    #[SingletonMethod]
    public function createHttpClient(GithubConfig $config): HttpClientInterface
    {
        return new Client(
            $config->getAccessToken(),
            $config->getSecret(),
        );
    }
    
    // Binds to DbFactory, NOT DatabaseFactory
    #[SingletonMethod(alias: DbFactory::class)]
    public function createDatabaseFactory(): DatabaseFactory
    {
        return new DatabaseFactory();
    }
    
    // Binds to BOTH LogManagerInterface AND LogManager
    #[SingletonMethod(
        alias: LogManagerInterface::class, 
        aliasesFromReturnType: true
    )]
    public function createLogManager(): LogManager
    {
        return new LogManager();
    }
}
```

##### #[BindMethod]

Creates a factory binding - the method is called each time the dependency is resolved, creating a new instance.

**Parameters:**

| Parameter               | Type           | Default | Description                                      |
|-------------------------|----------------|---------|--------------------------------------------------|
| `alias`                 | `string\|null` | `null`  | Bind to this alias instead of return type        |
| `aliasesFromReturnType` | `bool`         | `false` | Also bind to return type when alias is specified |

**Use cases:**

- Stateless services
- Request-scoped objects
- Objects that shouldn't be shared between requests

```php
use Spiral\Boot\Attribute\BindMethod;

final class ServicesBootloader extends Bootloader
{
    // New instance each time
    #[BindMethod]
    public function createHttpClient(): HttpClientInterface
    {
        return new HttpClient();
    }
    
    // New request factory each time
    #[BindMethod(alias: RequestFactory::class)]
    public function createRequestFactory(): RequestFactoryInterface
    {
        return new RequestFactory();
    }
}
```

##### #[InjectorMethod]

Creates a custom injector that controls how a type is resolved. The method itself becomes the resolver.

**Parameters:**

| Parameter | Type     | Required | Description        |
|-----------|----------|----------|--------------------|
| `alias`   | `string` | Yes      | The type to inject |

**Use cases:**

- Types that need context-aware creation
- Services that accept creation-time parameters
- Complex initialization logic that varies per resolution

```php
use Spiral\Boot\Attribute\InjectorMethod;

final class LoggingBootloader extends Bootloader
{
    // Each logger resolution can specify a different channel
    #[InjectorMethod(LoggerInterface::class)]
    public function createLogger(string $channel = 'default'): LoggerInterface
    {
        return new Logger($channel);
    }
    
    // Connection name can be provided at resolution time
    #[InjectorMethod(ConnectionInterface::class)]
    public function createConnection(?string $name = null): ConnectionInterface
    {
        return $name === null
            ? new DefaultConnection()
            : $this->connectionPool->get($name);
    }
}

// Later, in another class:
class UserRepository
{
    public function __construct(
        // Will call createConnection(null) -> DefaultConnection
        ConnectionInterface $connection,
        
        // Will call createLogger('user') -> Logger with 'user' channel
        #[LogChannel('user')] LoggerInterface $logger
    ) {}
}
```

##### #[BindAlias]

Adds additional binding aliases to a method. Can be applied multiple times.

**Parameters:**

| Parameter     | Type       | Required | Description                                 |
|---------------|------------|----------|---------------------------------------------|
| `...$aliases` | `string[]` | Yes      | Additional class/interface names to bind to |

**Use cases:**

- Binding one implementation to multiple interfaces
- Legacy interface support
- Multiple ways to resolve the same service

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindAlias};

final class LoggingBootloader extends Bootloader
{
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindAlias(MonologLoggerInterface::class)]
    public function createLogger(): Logger
    {
        return new Logger();
    }
}
```

**This binds to ALL of these:**

- `Logger` (return type)
- `LoggerInterface`
- `PsrLoggerInterface`
- `MonologLoggerInterface`

Any of these types can now be injected and will receive the same Logger instance.

##### #[BindScope]

Binds a service to a specific container scope. Can be applied multiple times for multiple scopes.

**Parameters:**

| Parameter | Type                  | Required | Description        |
|-----------|-----------------------|----------|--------------------|
| `scope`   | `string\|\BackedEnum` | Yes      | Scope name or enum |

**Use cases:**

- HTTP-only services (request, response, session)
- Console-only services (input, output)
- Worker-specific services
- Environment-specific bindings

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindScope};

final class ServicesBootloader extends Bootloader
{
    // Only available in 'http' scope
    #[SingletonMethod]
    #[BindScope('http')]
    public function createHttpClient(): HttpClientInterface
    {
        return new HttpClient();
    }
    
    // Only in 'console' scope
    #[SingletonMethod]
    #[BindScope('console')]
    public function createConsoleOutput(): OutputInterface
    {
        return new ConsoleOutput();
    }
    
    // Available in BOTH 'http' and 'grpc' scopes
    #[SingletonMethod]
    #[BindScope('http')]
    #[BindScope('grpc')]
    public function createSharedService(): SharedServiceInterface
    {
        return new SharedService();
    }
}
```

**How scopes work:**

- Bindings without `#[BindScope]` are global (available everywhere)
- Scoped bindings are only available when running in that scope
- Attempting to resolve a scoped binding outside its scope throws an exception

##### Combining Attributes

Attributes can be combined for complex binding scenarios:

```php
use Spiral\Boot\Attribute\{SingletonMethod, BindAlias, BindScope};

final class LoggingBootloader extends Bootloader
{
    // Singleton logger with multiple aliases, only in HTTP scope
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindScope('http')]
    public function createHttpLogger(): Logger
    {
        return new Logger('http');
    }
    
    // Different logger for console scope
    #[SingletonMethod]
    #[BindAlias(LoggerInterface::class, PsrLoggerInterface::class)]
    #[BindScope('console')]
    public function createConsoleLogger(): Logger
    {
        return new Logger('console');
    }
}
```

**Result:**

- HTTP requests get an HTTP logger via `LoggerInterface` or `PsrLoggerInterface`
- Console commands get a console logger via the same interfaces
- Both are singletons within their scope
- They can't be resolved outside their respective scopes

## Bootloader Dependencies

Ensure other bootloaders are loaded and initialized before yours. Dependencies are loaded only once, even if required by
multiple bootloaders.

### Method 1: Inject as Parameter

Request the bootloader directly in your `init()` or `boot()` method:

```php
use Spiral\Bootloader\Http\HttpBootloader;

class ApiBootloader extends Bootloader 
{
    public function boot(HttpBootloader $http): void
    {
        // HttpBootloader is guaranteed to be initialized
        $http->addMiddleware(ApiAuthMiddleware::class);
        $http->addMiddleware(RateLimitMiddleware::class);
    }
}
```

**Use case:** When you need to interact with the dependency's methods or properties.

### Method 2: Use DEPENDENCIES Constant

Declare dependencies without accessing them:

```php
class ApiBootloader extends Bootloader 
{
    public function defineDependencies(): void
    {
        return [
            HttpBootloader::class,
            CorsBootloader::class,
            AuthBootloader::class,
        ];
    }
    
    public function boot(): void
    {
        // All dependencies are initialized before this runs
        // Use when you need dependencies loaded but don't interact with them directly
    }
}
```

**Use case:** When dependencies just need to be loaded (for their side effects) but you don't need to call their
methods.

### When to Use Each

```php
// Use injection when you need to call methods
public function boot(HttpBootloader $http): void
{
    $http->addMiddleware(MyMiddleware::class);  // Interacting with dependency
}

// Use defineDependencies when you just need initialization
public function defineDependencies(): void
{
    return [ 
        DatabaseBootloader::class,  // Just needs to set up database
        CacheBootloader::class,     // Just needs to configure cache
    ];
}
```

## Dynamic Bootloading

Load bootloaders conditionally at runtime based on application state.

**Parameters of `bootload()` method:**

| Parameter          | Type    | Description                              |
|--------------------|---------|------------------------------------------|
| `classes`          | `array` | Array of bootloader class names to load  |
| `bootingCallbacks` | `array` | Callbacks to run before bootloaders boot |
| `bootedCallbacks`  | `array` | Callbacks to run after bootloaders boot  |

**Use cases:**

- Feature flags
- Environment-specific bootloaders
- Plugin systems
- Conditional feature loading

```php
use Spiral\Boot\BootloadManagerInterface;

class AppBootloader extends Bootloader
{
    public function boot(
        BootloadManagerInterface $bootloadManager, 
        EnvironmentInterface $env
    ): void {
        // Load debug tools only when DEBUG is enabled
        if ($env->get('DEBUG')) {
            $bootloadManager->bootload([
                DebugBootloader::class,
                ProfilerBootloader::class,
            ]);
        }
        
        // Load feature bootloaders based on feature flags
        if ($this->isFeatureEnabled('new-api')) {
            $bootloadManager->bootload([
                NewApiBootloader::class,
            ]);
        }
        
        // Load different bootloaders per environment
        match($env->get('APP_ENV')) {
            'production' => $bootloadManager->bootload([
                ProductionCacheBootloader::class,
                ProductionLoggingBootloader::class,
            ]),
            'local' => $bootloadManager->bootload([
                DevToolsBootloader::class,
                LocalCacheBootloader::class,
            ]),
            default => null,
        };
    }
}
```

> **Warning**
> Dynamic bootloading happens during the boot phase, so dynamically loaded bootloaders cannot have `init()` methods or
`#[InitMethod]` attributes - the initialization phase has already completed.

<hr>

## What's Next?

* [HTTP — Interceptors](../http/interceptors.md)
* [Scaffolding](../basics/scaffolding.md)
* [Container — Attributes](../container/attributes.md)
