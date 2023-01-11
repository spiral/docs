# Framework — Finalizers

Most of the framework components do not require any resetting after the request completion. However, there are
multiple use-cases when you might want to reset your library after the user request is complete.

> **Note**
> Prioritize usage of IoC scopes over finalizers.

## FinalizerInterface

Use `Spiral\Boot\FinalizerInterface`

```php
/**
 * Used to close resources and connections for long-running processes.
 */
interface FinalizerInterface
{
    /**
     * Finalizers are executed after every request and used for garbage collection
     * or to close open connections.
     *
     * @param callable $finalizer
     */
    public function addFinalizer(callable $finalizer);
    
    /**
     * Finalize execution.
     *
     * @param bool $terminate Set to true if finalization is caused on application termination.
     */
    public function finalize(bool $terminate = false);
}
```

All of the application dispatchers will invoke the finalizer. During:

* HTTP request complete
* HTTP request failed with error
* job is complete
* job failed with error
* GRPC call complete
* GRPC call failed with error
* console command is complete

> **Warning**
> The finalizer will only be invoked if a specific dispatcher has been started. You can freely invoke app
> commands and HTTP methods without using a dispatcher directly and without resetting your services after each request.

Your handler will receive the first bool argument, which specifies if the app is going to terminate after the request.

> **Note**
> Avoid resetting IoC setting in the finalizer as it might lead to some singleton service cache previous service
> version.

## Example Finalizer

We can use a finalizer to demonstrate how to close the database connection after every request automatically. It can be
useful if you run a lot of workers (or lambda functions) and do not want to consume all of the database sockets.

```php
// in bootloader
use Spiral\Boot\FinalizerInterface;
use Psr\Container\ContainerInterface;
use Cycle\Database\DatabaseManager;

public function boot(FinalizerInterface $finalizer, ContainerInterface $container): void
{
    $finalizer->addFinalizer(function () use ($container) {
        /** @var DatabaseManager $dbal */
        $dbal = $container->get(DatabaseManager::class);
 
        foreach ($dbal->getDrivers() as $driver) {
            $driver->disconnect();
        }
    });
}
```

> **Note**
> You can find such bootloader already included in the [`spiral\cycle-bridge` package](https://github.com/spiral/cycle-bridge/blob/master/src/Bootloader/DisconnectsBootloader.php) 
> and available as `Spiral\Cycle\Bootloader\DisconnectsBootloader`.
