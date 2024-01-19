# Container — Attributes

Spiral provides a set of attributes that can be used to provide additional control over dependency resolution:

## Singleton

Using the `#[Singleton]` attribute can be handy in situations where we want to ensure that only one instance of a class
is created and used throughout the application, such as with configuration managers, database connections, or loggers.

#### Example

Suppose you have a configuration manager in your application that reads from a configuration file and provides access to
various configuration settings. It would be wasteful to reload and parse the configuration file every time you need to
access a setting. Using the #[Singleton] attribute ensures that once the configuration file is loaded and parsed, it
remains in memory for the duration of the application's lifetime.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class ConfigurationManager
{
    private readonly array $config;

    public function __construct()
    {
        $this->config = parse_ini_file('config.ini');
    }

    public function get(string $key)
    {
        return $this->config[$key] ?? null;
    }
}
```

## Scope

Allows you to set a specific scope in which a dependency can be resolved. If the dependency is attempted to be resolved
in a different scope, an exception will be thrown, indicating a scope mismatch. This attribute helps enforce strict
scoping rules and prevents dependencies from being mistakenly resolved in unintended scopes.

#### Example

Consider an application where certain resources or operations are restricted to authenticated users. For such resources,
you'd want to:

1. Ensure that the user is authenticated.
2. Make the authenticated user's data available throughout the application during the current request.

This class holds information about the authenticated user. This data is only meant to be available and resolved when the
user is authenticated (i.e., within an auth scope).

```php
use Spiral\Core\Attribute\Scope;

#[Scope('auth')]
final readonly class AuthenticatedUser
{
    public function __construct(
        private int $id,
        private string $name, 
        private string $email,
    ) {
    }
}
```

The middleware checks if the user is authenticated. If they are, it sets up an auth scope in the IoC container and binds
the authenticated user's data to the AuthenticatedUser class.

```php;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\RequestHandlerInterface as Handler;

final class AuthMiddleware implements MiddlewareInterface
{
    public function __construct(
        private readonly ContainerInterface $container
    ) {
    }

    public function process(Request $request, Handler $handler): Response
    {
        // This is a simplified authentication check.
        // In a real application, this might involve checking session data, JWT tokens, etc.
        if ($request->hasHeader('Authorization')) {
            // Fetch user data based on the authorization header. 
            // For simplicity, we're hardcoding user data here.
            $user = new AuthenticatedUser(1, 'John Doe', 'john.doe@example.com');

            // Set up the auth scope
            return $this->container->runScoped(
                closure: function (ContainerInterface $container) use ($next, $request) {
                    // Now, within this scope, you can get the authenticated user instance.
                    $authenticatedUser = $container->get(AuthenticatedUser::class);
                    
                    // Continue processing the request.
                    return $handler($request);
                },
                bindings: [AuthenticatedUser::class => $user],
                name: 'auth'
            );

        } else {
            // No authentication header found.
            // Return an unauthorized response or simply continue processing.
            return $handler($request);
        }
    }
}
```

Let's say we have a controller or service that requires the authenticated user's data. By attempting to resolve the
`AuthenticatedUser` class from the container, we can be sure we are either getting the authenticated user or that we are
within the auth scope (thanks to the #[Scope('auth')] attribute).

```php
final class UserProfileController
{
    public function getProfile(AuthenticatedUser $user)
    {
        // Use the $user data to fetch and return the profile.
    }
}
```

> **Note**
> Read more about container scopes in the [Framework — IoC Scopes](../framework/scopes.md) section.

## Finalize

Allows you to define a finalize method for a class. When a dependency is resolved within a scope, and that scope is
destroyed, the finalize method specified by this attribute will be called. The purpose of the finalize method is to
perform any necessary cleanup or finalization actions before the scope is destroyed. This attribute provides a
convenient way to handle resource cleanup and ensure proper destruction of objects when a scope is no longer needed.

#### Example

Consider you're working with a database connection. Once you're done with it, especially within a specific scope, you
might want to close the connection or release other resources.

```php
use Spiral\Core\Attribute\Finalize;

#[Finalize(method: 'closeConnection')]
final class DatabaseConnection
{
    private $connection;

    public function __construct()
    {
        // Initialize the database connection
    }

    public function query($sql)
    {
        // Execute the query on the database
    }

    public function closeConnection(): void
    {
        // Close the connection
    }
}
```

When using this class within a scope, once the scope ends, the closeConnection method will be invoked, ensuring that
resources are released:

```php
$container->runScoped(
    closure: function (DatabaseConnection $db) {
        // Execute some database operations
        $users = $db->query('SELECT * FROM users');
        // ...  
    },
    bindings: [DatabaseConnection::class => new DatabaseConnection()],
);

// Once the scope is destroyed, the connection is automatically closed.
```

> **Warning**
> The object can be leaked but finalized. You should avoid such situations.

```php
$root = new Container();
$obj = $root->get(Foo::class);
unset($root); // The Foo finalizer will be called

// Here we have a leaked finalized object. It is `$obj`.
```

## Combining Attributes

All the attributes — `#[Finalize]`, `#[Singleton]`, and `#[Scope]` — are fully compatible with each other. This means
that you can use these attributes simultaneously on the same class, allowing for fine-grained control over the behavior
and lifecycle of your dependencies.

#### Example

Imagine you have a caching service where:

1. Only a single instance of the caching service should be created throughout the application's lifecycle (singleton).
2. The service should only be available within certain parts of your application, like within an HTTP request handling (
   scoped).
3. When the application is shutting down, or when the scope ends, you want to ensure all pending cache operations (like
   write-backs) are finalized and any resources (like open file handles or network connections) are closed.

```php
namespace App\Services;

use Psr\Log\LoggerInterface;
use Spiral\Core\Attribute\Finalize;
use Spiral\Core\Attribute\Scope;
use Spiral\Core\Attribute\Singleton;

#[Singleton]
#[Scope('http')]
#[Finalize(method: 'shutdown')]
final class CacheService
{
    private array $cache = [];
    public function __construct(
        private readonly LoggerInterface $logger,
    ) {
    }

    public function get(string $key): ?string
    {
        return $this->cache[$key] ?? null;
    }

    public function set(string $key, string $value): void
    {
        $this->cache[$key] = $value;
    }

    public function shutdown(): void
    {
        $this->logger->info("CacheService is finalizing.");
        
        // Flush the cache to a persistent storage, close any resources, etc.
        $this->cache = [];
    }
}
```
