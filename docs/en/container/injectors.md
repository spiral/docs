# Container — Injectors

Spiral provides a powerful injection system that allows you to control the creation process of any interface or abstract
class children using the injector interface. This feature enables dynamic resolution of dependencies based on context,
making it highly flexible for complex dependency scenarios.

## What are Injectors?

Injectors are special components that intercept the container's resolution process for specific interfaces or classes.
Instead of relying solely on the container's auto-wiring, injectors give you fine-grained control over how instances
are created and configured based on the calling context.

**Key Benefits:**

- **Context-Aware Resolution**: Create different implementations based on the parameter name, type, or attributes
- **Flexibility**: Dynamically choose implementations at runtime without changing client code
- **Decoupling**: Keep client code independent of specific implementations
- **Lifecycle Control**: Manage instantiation logic, initialization, and configuration in one place

## Creating Class Injectors

### Basic Injector Implementation

An injector must implement the `Spiral\Core\Container\InjectorInterface`, which provides the `createInjection` method.
This method is called every time the container needs to resolve a specific class or interface.

Let's create a simple cache injector that provides different cache implementations based on context:

```php app/src/Application/Bootloader/CacheBootloader.php
namespace App\Application\Bootloader;

use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\InjectorInterface;

final class CacheBootloader extends Bootloader implements InjectorInterface
{
    public function __construct(
        private readonly ContainerInterface $container,
    ) {
    }

    public function boot(BinderInterface $binder): void
    {
        // Register this bootloader as the injector for CacheInterface
        $binder->bindInjector(CacheInterface::class, self::class);
    }

    public function createInjection(
        \ReflectionClass $class,
        \Stringable|string|null $context = null
    ): CacheInterface {
        return match ($context) {
            'redis' => new RedisCache(...),
            'memcached' => new MemcachedCache(...),
            default => new ArrayCache(...),
        };
    }
}
```

> **Note**
> Don't forget to activate the bootloader in your application.

### Understanding Context Parameters

The `createInjection` method receives two important parameters:

| Parameter  | Type                        | Description                                                      |
|------------|-----------------------------|------------------------------------------------------------------|
| `$class`   | `\ReflectionClass`          | Reflection object of the requested class or interface            |
| `$context` | `\Stringable\|string\|null` | Context information about where the injection is being requested |

The `$context` parameter can contain different types of information:

- **String**: Parameter name from constructor or method arguments
- **ReflectionParameter**: Full parameter reflection (for advanced scenarios)
- **Stringable**: Custom context objects
- **null**: No context available

### Using String Context

The simplest form of context-aware injection uses parameter names:

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use Psr\SimpleCache\CacheInterface;

final class BlogController
{
    public function __construct(
        private readonly CacheInterface $redis,
        private readonly CacheInterface $memcached,
        private readonly CacheInterface $cache,
    ) {
        // $redis will be an instance of RedisCache
        // $memcached will be an instance of MemcachedCache  
        // $cache will be an instance of ArrayCache
    }
}
```

The injector receives the parameter name as the `$context` and returns the appropriate implementation.

## Advanced Context Handling

### Using ReflectionParameter Context

Starting from `3.16.x`, injectors can receive `ReflectionParameter` objects as context, enabling more
sophisticated resolution strategies based on parameter attributes, types, or other metadata.

To use `ReflectionParameter` context, your `createInjection` signature should accept it:

```php
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): object {
    // Now $context can be a ReflectionParameter
}
```

#### Attribute-Based Injection

This is particularly powerful when combined with PHP attributes:

```php app/src/Application/Attribute/DatabaseDriver.php
namespace App\Application\Attribute;

#[\Attribute(\Attribute::TARGET_PARAMETER)]
final readonly class DatabaseDriver
{
    public function __construct(
        public string $name,
    ) {
    }
}
```

Create an injector that reads parameter attributes:

```php app/src/Application/Injector/DatabaseInjector.php
namespace App\Application\Injector;

use App\Application\Attribute\DatabaseDriver;
use Spiral\Core\Container\InjectorInterface;

final class DatabaseInjector implements InjectorInterface
{
    public function createInjection(
        \ReflectionClass $class,
        \ReflectionParameter|string|null $context = null
    ): object {
        // Extract attribute from parameter
        $driver = $context instanceof \ReflectionParameter
            ? $context->getAttributes(DatabaseDriver::class)[0]?->newInstance()?->name ?? 'mysql'
            : 'mysql';

        return match ($driver) {
            'sqlite' => new Sqlite(),
            'mysql' => new Mysql(),
            'postgres' => new PostgreSQL(),
            default => throw new \InvalidArgumentException("Unknown database driver: {$driver}"),
        };
    }
}
```

Register the injector in a bootloader:

```php app/src/Application/Bootloader/DatabaseBootloader.php
namespace App\Application\Bootloader;

use App\Application\DatabaseInterface;
use App\Application\Injector\DatabaseInjector;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;

final class DatabaseBootloader extends Bootloader
{
    public function boot(BinderInterface $binder): void
    {
        $binder->bindInjector(DatabaseInterface::class, DatabaseInjector::class);
    }
}
```

Now you can use attributes to specify which database driver to inject:

```php app/src/Application/Service/SomeService.php
namespace App\Application\Service;

use App\Application\Attribute\DatabaseDriver;
use App\Application\DatabaseInterface;

final class SomeService
{
    public function __construct(
        #[DatabaseDriver(name: 'mysql')]
        private readonly DatabaseInterface $mysql,

        #[DatabaseDriver(name: 'sqlite')]  
        private readonly DatabaseInterface $sqlite,
        
        #[DatabaseDriver(name: 'postgres')]
        private readonly DatabaseInterface $postgres,
    ) {
    }
}
```

### Stringable Context

Injectors support custom `Stringable` objects as context, allowing you to pass rich contextual information:

```php
use Spiral\Core\FactoryInterface;

final class MyContextualClass implements \Stringable
{
    public function __construct(
        public readonly string $environment,
        public readonly array $options = [],
    ) {
    }

    public function __toString(): string
    {
        return $this->environment;
    }
}

// Using with FactoryInterface
$factory->make(
    CacheInterface::class, 
    context: new MyContextualClass('production', ['ttl' => 3600])
);
```

## Class Inheritance Support

Injectors support class inheritance, allowing you to create specialized implementations based on the requested class
type.

> **Note**
> Currently, injectors only support classes (not interfaces) that extend a base class. Future Spiral releases may
> support interface inheritance.

### Example with Abstract Classes

Define specialized abstract classes:

```php
use Psr\SimpleCache\CacheInterface;

abstract class RedisCacheInterface implements CacheInterface
{
    // Redis-specific methods
}

abstract class MemcachedCacheInterface implements CacheInterface
{
    // Memcached-specific methods
}
```

Implement the injector to handle inheritance:

```php app/src/Application/Bootloader/CacheBootloader.php
public function createInjection(
    \ReflectionClass $class,
    \Stringable|string|null $context = null
): CacheInterface {
    // Check for specialized abstract classes first
    if ($class->isSubclassOf(RedisCacheInterface::class)) {
        return new RedisCache(...);
    }
    
    if ($class->isSubclassOf(MemcachedCacheInterface::class)) {
        return new MemcachedCache(...);
    }
    
    // Fall back to context-based resolution
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        default => new ArrayCache(...),
    };
}
```

## Container Factory Context Support

The container's `FactoryInterface` now accepts `Stringable` context for the `make` method, enabling custom context
objects to be passed during dependency resolution:

```php
use Spiral\Core\FactoryInterface;

final readonly class MyContext implements \Stringable
{
    public function __construct(
        public string $environment,
    ) {
    }
    
    public function __toString(): string
    {
        return $this->environment;
    }
}

final class MyService
{
    public function __construct(
        private readonly FactoryInterface $factory,
    ) {
    }
    
    public function createWithContext(): mixed
    {
        return $this->factory->make(
            SomeClass::class,
            context: new MyContext('production')
        );
    }
}
```

The injector receives this context and can use it for resolution decisions:

```php
public function createInjection(
    \ReflectionClass $class,
    \Stringable|string|null $context = null
): object {
    if ($context instanceof MyContext) {
        // Use the environment from context
        return match ($context->environment) {
            'production' => new ProductionImplementation(),
            'staging' => new StagingImplementation(),
            default => new DevelopmentImplementation(),
        };
    }
    
    // Handle other context types
    return new DefaultImplementation();
}
```

## Enum Injectors

The `spiral/boot` component provides a convenient way to resolve enum classes using the
`Spiral\Boot\Injector\InjectableEnumInterface` interface. When the container requests an Enum, it calls a specified
method to determine the current value and inject any required dependencies.

### Benefits of Enum Injections

- **Type Safety**: Ensures correct types are passed to methods and classes
- **Dynamic Resolution**: Resolve enum values based on current application state
- **Reusability**: Reuse enum injection logic across different parts of the application
- **Improved Readability**: Makes code more self-explanatory
- **Centralized Configuration**: Manage enum values in one place

### Creating Injectable Enums

Use the `#[ProvideFrom]` attribute to specify a static method that provides the enum value:

```php app/src/Application/Enum/AppEnvironment.php
namespace App\Application\Enum;

use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum AppEnvironment: string implements InjectableEnumInterface
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }

    public function isTesting(): bool
    {
        return $this === self::Testing;
    }

    public function isLocal(): bool
    {
        return $this === self::Local;
    }

    public function isStage(): bool
    {
        return $this === self::Stage;
    }

    /**
     * Detect environment from configuration
     * Dependencies are automatically injected by the container
     */
    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

### Using Injectable Enums

The container automatically injects enums with the correct value:

```php app/src/Endpoint/Console/MigrateCommand.php
namespace App\Endpoint\Console;

use App\Application\Enum\AppEnvironment;
use Spiral\Console\Command;

final class MigrateCommand extends Command
{
    protected const NAME = 'migrate';

    public function perform(AppEnvironment $appEnv): int
    {
        if ($appEnv->isProduction()) {
            $this->error('Migrations are disabled in production');
            return self::FAILURE;
        }
        
        // Perform migration
        $this->info('Running migrations...');
        
        return self::SUCCESS;
    }
}
```

Enums can also be injected through constructors:

```php app/src/Application/Service/ConfigService.php
namespace App\Application\Service;

use App\Application\Enum\AppEnvironment;

final readonly class ConfigService
{
    public function __construct(
        private AppEnvironment $environment,
    ) {
    }
    
    public function getCacheTTL(): int
    {
        return match ($this->environment) {
            AppEnvironment::Production, AppEnvironment::Stage => 3600,
            AppEnvironment::Testing => 60,
            AppEnvironment::Local => 0,
        };
    }
}
```

## Best Practices

### Keep Injectors Focused

Each injector should handle a single interface or abstract class. Don't create "mega-injectors" that handle multiple
unrelated types.

```php
// ✅ Good: Focused injector
final class CacheInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
    {
        // Handle only cache-related injection
    }
}

// ❌ Bad: Handles multiple unrelated types
final class MegaInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): object
    {
        if ($class->implementsInterface(CacheInterface::class)) { /* ... */ }
        if ($class->implementsInterface(LoggerInterface::class)) { /* ... */ }
        if ($class->implementsInterface(QueueInterface::class)) { /* ... */ }
        // Too many responsibilities
    }
}
```

### Provide Sensible Defaults

Always provide a default implementation when context-based resolution doesn't match:

```php
public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
{
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        // ✅ Sensible default prevents unexpected failures
        default => new ArrayCache(...),
    };
}
```

### Use Type Hints

Always specify return types and parameter types for better IDE support and type safety:

```php
// ✅ Good: Explicit types
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): CacheInterface {
    // ...
}

// ❌ Bad: Missing types
public function createInjection($class, $context = null) {
    // ...
}
```

### Document Context Expectations

Make it clear what context values your injector expects:

```php
/**
 * Cache injector that resolves implementations based on parameter names.
 * 
 * Supported context values:
 * - 'redis': Returns RedisCache instance
 * - 'memcached': Returns MemcachedCache instance  
 * - 'file': Returns FileCache instance
 * - null or other: Returns ArrayCache instance (default)
 */
final class CacheInjector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, $context = null): CacheInterface
    {
        // Implementation
    }
}
```

### Validate Context

When using `ReflectionParameter` context, validate that you're getting the expected information:

```php
public function createInjection(
    \ReflectionClass $class,
    \ReflectionParameter|string|null $context = null
): DatabaseInterface {
    if ($context instanceof \ReflectionParameter) {
        $attributes = $context->getAttributes(DatabaseDriver::class);
        
        if (empty($attributes)) {
            throw new \InvalidArgumentException(
                "Parameter {$context->getName()} must have DatabaseDriver attribute"
            );
        }
        
        $driver = $attributes[0]->newInstance()->name;
    } else {
        $driver = $context ?? 'mysql';
    }
    
    return $this->createDriver($driver);
}
```

## Troubleshooting

### Injector Not Being Called

If your injector isn't being invoked, verify:

1. The injector is properly registered via `$binder->bindInjector()`
2. The bootloader containing the injector is activated
3. You're requesting the correct interface/class that the injector handles
4. There's no direct binding that takes precedence (bindings are resolved before injectors)

### Context Is Always Null

If `$context` is always `null`, check:

1. You're not requesting the dependency directly from the container: `$container->get(CacheInterface::class)` has no
   context
2. The dependency is being resolved through constructor/method injection where parameter names are available
3. You're using `FactoryInterface::make()` with a context parameter if needed

### Type Mismatches with ReflectionParameter

When using `ReflectionParameter` context, ensure:

1. Your injector signature declares it accepts `ReflectionParameter`: `\ReflectionParameter|string|null $context`
2. You check the context type before treating it as a `ReflectionParameter`
3. Handle cases where context might be a string or null for backward compatibility

## See Also

- [Container — Overview](./overview.md) - Learn about container basics
- [Container — Configuration](./configuration.md) - Configure bindings and dependencies
- [Container — Attributes](./attributes.md) - Use attributes for dependency management
- [Container — IoC Scopes](./scopes.md) - Understand scoped dependency resolution
