# Advanced — Atomic Locks

Spiral has full integration with [RoadRunner locks](https://roadrunner.dev/docs/plugins-locks) plugin, which enables you
to manage resource locks in your application. With this component, you can easily manage critical sections of your
application and prevent race conditions, data corruption, and other synchronization issues that can occur in
multi-process environments.

To enable the integration with RoadRunner, Spiral provides built-in support through
the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package.

> **Warning**
> Locks are available only in the `spiral/roadrunner-bridge` package version `3.2` and higher.

## Installation

To get started, you need to install [Roadrunner bridge](../start/server.md#roadrunner-bridge) package. Once installed,
add the `Spiral\RoadRunnerBridge\Bootloader\LockBootloader` to the list of bootloaders in your Kernel class:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
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
    \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

That's all it takes! No additional configuration is required on the RoadRunner side.

## Usage

Here's a brief example of how you can use locks in your application:

```php
$locks = $container->get(\RoadRunner\Lock\LockInterface::class);
$id = $lock->lock('pdf:create');

// Your logic for creating a PDF file

$lock->release('pdf:create', $id);
```

> **Warning**
> RoadRunner lock plugin uses an in-memory storage to store information about locks at this moment. When multiple
> instances of RoadRunner are used, each instance will have its own in-memory storage for locks. As a result, if a
> process acquires a lock on one instance of RoadRunner, it will not be aware of the lock state on the other instances.

### Acquiring Locks

Locking a resource ensures that only one process can access it at a time, preventing data conflicts and ensuring
consistency.

#### Basic Lock Acquisition

Acquires an exclusive lock on a specified resource.

```php
$id = $lock->lock('pdf:create');
```

This is the simplest form of acquiring a lock. It attempts to lock the resource identified by `pdf:create`. If
successful, it returns an identifier for the lock.

#### Lock with Time-to-Live (TTL)

```php
$id = $lock->lock('pdf:create', ttl: 10);
// or
$id = $lock->lock('pdf:create', ttl: new \DateInterval('PT10S'));
```

These lines demonstrate how to acquire a lock with a TTL (Time-to-Live). The first line sets a TTL of 10 microseconds,
while the second line uses a DateInterval object to set a TTL of 10 seconds. TTL is used to specify how long the lock
should be held before it's automatically released.

#### Lock with Wait Time

```php
$id = $lock->lock('pdf:create', wait: 5);
// or
$id = $lock->lock('pdf:create', wait: new \DateInterval('PT5S'));
```

These lines show how to acquire a lock with a specified wait time. If the lock is not immediately available, the process
will wait for the given time (5 microseconds in the first line, and 5 seconds in the second line) before giving up.

#### Lock with ID

```php
$id = $lock->lock('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

This line demonstrates acquiring a lock with a specific identifier. This can be useful for tracking or logging purposes,
allowing you to specify a custom identifier for the lock.

### Acquiring Read Locks

In concurrent programming, acquiring a read lock allows multiple processes to access a resource simultaneously for
reading, while still preventing exclusive write access.

There are similar methods for acquiring read locks:

```php
$id = $lock->lockRead('pdf:create', ttl: 100000);
// or
$id = $lock->lockRead('pdf:create', ttl: new \DateInterval('PT10S'));

// Acquire lock and wait 5 microseconds until lock will be released
$id = $lock->lockRead('pdf:create', wait: 5);
// or
$id = $lock->lockRead('pdf:create', wait: new \DateInterval('PT5S'));

// Acquire lock with id - 14e1b600-9e97-11d8-9f32-f2801f1b9fd1
$id = $lock->lockRead('pdf:create', id: '14e1b600-9e97-11d8-9f32-f2801f1b9fd1');
```

### Release lock

Releasing locks is an essential part of working with concurrency control. It ensures that resources are freed up for use
by other processes.

> **Warning**
> You should always release a lock once the task requiring exclusive or shared access to the resource is completed.
> This ensures that other processes can acquire the lock when needed.

To release a previously acquired exclusive or read lock:

```php
$id = $lock->lock('pdf:create');

// Your logic for creating a PDF file

$lock->release('pdf:create', $id);
```

#### Force Release of a Lock

In some cases, you may need to release a lock without knowing the lock identifier. For example, if a process crashes
while holding a lock, the lock will not be released.

```php
$lock->forceRelease('pdf:create');
```

### Check lock

To check if a lock exists on a resource, use the `exists()` method:

```php
$status = $lock->exists('pdf:create');

if ($status) {
    // Lock exists
} else {
    // Lock not exists
}
```

### Update TTL

In some cases, you may need to update the TTL of a lock. For example, if a process is performing a long-running task,
you may want to extend the TTL to prevent the lock from being released before the task is completed.

```php
// Add 10 microseconds to lock ttl
$lock->updateTTL('pdf:create', $id, 10);
// or
$lock->updateTTL('pdf:create', $id, new \DateInterval('PT10S'));
```

## Symfony Integration

We provide also a package that adds lock driver to
the [Symfony lock](https://symfony.com/doc/current/components/lock.html) component.

To install the package, run the following command:

```terminal
composer require roadrunner-php/symfony-lock-driver
```

Once installed, you need to create a bootloader class that will register the lock driver in the container:

```php app/src/Application/Bootloader/LockBootloader.php
<?php

declare(strict_types=1);

namespace App\Application\Bootloader;

use RoadRunner\Lock\LockInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\Environment\AppEnvironment;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Goridge\RPC\RPCInterface;
use Spiral\RoadRunner\Symfony\Lock\RoadRunnerStore;
use Spiral\RoadRunnerBridge\Bootloader\RoadRunnerBootloader;
use Symfony\Component\Lock\LockFactory;
use Symfony\Component\Lock\PersistingStoreInterface;
use Symfony\Component\Lock\Store\InMemoryStore;
use Symfony\Component\Lock\Store\RedisStore;

final class LockBootloader extends Bootloader
{
    public function defineDependencies(): array
    {
        return [
            \Spiral\RoadRunnerBridge\Bootloader\LockBootloader::class
        ];
    }

    public function defineSingletons(): array
    {
        return [
            LockFactory::class => [self::class, 'initLockFactory'],
        ];
    }

    protected function initLockFactory(LockInterface $rrLock, EnvironmentInterface $env): LockFactory
    {
        $driver = $env->get('LOCK_DRIVER', 'roadrunner');
        $defaultTtl = $env->get('LOCK_DRIVER_TTL', 100);

        $store = match ($driver) {
            'memory' => new InMemoryStore(), // for testing purposes
            'roadrunner' => new RoadRunnerStore($rrLock, initialWaitTtl: $defaultTtl),
            default => throw new \InvalidArgumentException("Unknown lock driver: {$driver}"),
        };

        return new LockFactory($store);
    }
}
```

Now you can use the Symfony lock component in your application:

```php
use Symfony\Component\Lock\LockFactory;

$factory = $container->get(LockFactory::class);
$lock = $factory->createLock('pdf-creation');

if ($lock->acquire()) {
    // The resource "pdf-creation" is locked.
    // You can compute and generate the invoice safely here.

    $lock->release();
}
```

> **Note**
> Read more about the Symfony lock component in
> the [Symfony documentation](https://symfony.com/doc/current/components/lock.html).