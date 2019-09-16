# Framework - Finalizers
Most of the framework components does not require any resetting after the request completion. However, there is multiple
use-cases when you might want to reset your library after user request is complete.

> Prioritize usage of IoC scopes over finalizers.

## FinalizerInterface
Use `Spiral\Boot\FinalizerInterface`

```php
/**
 * Used to close resources and connections for long running processes.
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

The finalizer will be invoked by all of the application dispatchers. During:
* HTTP request complete
* HTTP request failed with error
* job is complete
* job failed with error
* GRPC call complete
* GRPC call failed with error
* console command is complete

> Attention, the finalizer will only be invoked if the specific dispatcher has been started. You can freely invoke app commands
and HTTP methods without using dispatcher directly and without resetting your services after each request.

Your handler will receive first bool argument which specifies if the app is going to terminate after the request.

> Note, avoid resetting IoC setting in finalizer as it might lead to some singleton service cache previous service version.

## Example Finalizer
We can use a finalizer to demonstrate how to automatically close the database connection after every request. It can be useful if you run a lot of workers (or lambda functions) and do not want to consume all of the database sockets.


```php
// in bootloader
/**
 * @param FinalizerInterface $finalizer
 * @param ContainerInterface $container
 */
public function boot(FinalizerInterface $finalizer, ContainerInterface $container)
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

> You can find such bootloader already included with framework and available as `Spiral\Bootloader\Database\DisconnectsBootloader`.
