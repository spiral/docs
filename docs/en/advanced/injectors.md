# Advanced — Injectors

The Spiral Framework provides a way to control the creation process of any interface or abstract class children using an
injection interface. The `Spiral\Core\Container\InjectorInterface` is implemented by the Injector class which provides
a method that will be used every time when a specific class is requested from the container.

**There are several benefits to using the Spiral Framework's injection interface:**

- **Flexibility:** The injection interface allows for the creation of a specific class based on a specific context,
  providing a high degree of flexibility in the instantiation of classes.
- **Decoupling:** The injection interface allows for the decoupling of classes from their dependencies, making the code
  more modular and easier to maintain.
- **Control over instantiation:** The injection interface provides a way to control the instantiation of classes, making
  it easy to manage the lifecycle of objects and to ensure that they are properly initialized.

This guide demonstrates how to create a class instance and assign a unique value to it, no matter what children
implement it.

## Class injectors

Let's imagine that we have an interface `Psr\SimpleCache\CacheInterface` that provides a simple cache interface.

### Creating an injector

When we want to get a specific cache implementation based on a context. The injector class should implement the
`Spiral\Core\Container\InjectorInterface` which provides a method called `createInjection`. This method is used every
time a specific class is requested from the container.

In oue example we can combine a bootloader and an injector into one instance:

```php
namespace App\Application\Bootloader;

use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\InjectorInterface;

class CacheBootloader extends Bootloader implements InjectorInterface
{
    public function __construct(
        private readonly ContainerInterface $container,
    ) {}

    public function boot(BinderInterface $binder): void
    {
        $binder->bindInjector(CacheInterface::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
    {
        return match ($context) {
            'redis' => new RedisCache(...),
            'memcached' => new MemcachedCache(...),
            default => new ArrayCache(...),
        };
    }
}
```

> **Note**
> Do not forget to activate the bootloader.

When the container resolves the `CacheInterface`, it will request it from the injector using the `createInjection`
method. The method takes in two arguments, `$class` and `$context`. The `$class` argument returns the `ReflectionClass`
object for the requested class and the `$context` argument returns a Parameter or alias name (for example, the argument
name of the method or function that requested the injectable class).

### Using the injector

Now we can use the injector to get a specific cache implementation based on the context:

```php
namespace App\Application\Controller;

use Psr\SimpleCache\CacheInterface;

class BlogController
{
    public function __construct(
        private readonly CacheInterface $redis,
        private readonly CacheInterface $memcached,
        private readonly CacheInterface $cache,
    } {
        
        \assert($redis instanceof RedisCache);

        \assert($memcached instanceof MemcachedCache);
        
        \assert($cache instanceof ArrayCache);
    }
}
```

In this example, the `BlogController` class takes in three properties, `$redis`, `$memcached`,
and `$cache`. These properties are all of type `CacheInterface`, but they correspond to different implementations of the
interface based on the context that was passed to the injector's `createInjection` method.

### Class inheritance

Yes, class inheritance is possible with the injector. Currently, the injector only supports classes that extend a base
class, but future releases of the Spiral Framework will also support interface inheritance.

```php
abstract class RedisCacheInterface implements CacheInterface
{
}
```

In this example, the `RedisCacheInterface` is an abstract class that implements the `CacheInterface`.

For example, in the `createInjection` method, we can check if the requested class is a subclass of `RedisCacheInterface`
and return a `RedisCache` instance, or check if the requested class is a subclass of `MemcachedCacheInterface` and
return a `MemcachedCache` instance.

```php
public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
{
    if ($class->isSubclassOf(RedisCacheInterface::class)) {
        return new RedisCache(...);
    }
    
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        default => new ArrayCache(...),
    };
}
```

## Enum Injectors

The `spiral/boot` component provides a convenient way for resolving enum classes using the
`Spiral\Boot\Injector\InjectableEnumInterface` interface. When the container requests an Enum, it will call a specified
method to determine the current value of the Enum and inject any required dependencies. This allows for easy management
of Enums throughout the application and allows for the dynamic resolution of Enum values based on the current state of
the application. This feature can be very useful for managing application-wide variables such as the current environment
or the current user's role.

Let's create an `Enum`, with which we can easily determine the environment of our application.

```php
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

    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

The `ProvideFrom` attribute is used to specify a detect method which is used to determine the current value
of the Enum. The method will be called when the container requests the Enum and any required dependencies from the
container will be injected into it.

### Usage

The Container will automatically inject an Enum with the correct value by using the method specified in
the `ProvideFrom`
attribute.

```php
final class MigrateCommand extends Command 
{
     const NAME = '...';

     public function perform(AppEnvironment $appEnv): int
     {
           if ($appEnv->isProduction()) {
                 // Deny
           }
           // ...
     }
}
```

This makes it easy to manage application-wide variables such as the current environment, without having to pass them
around or hardcode them in multiple places throughout the application.
