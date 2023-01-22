# The Basics — Cache

Caching is a technique that can significantly improve the performance of an application by storing frequently accessed
data in a faster storage medium, such as memory. This reduces the need to recalculate or fetch the data from a slower
source, such as a database. By implementing a caching layer, the application can quickly retrieve the data from the
cache instead of recalculating it or fetching it from the slower source, which can improve the overall response time and
reduce the load on the slower source.

The Spiral Framework's `spiral/cache` component allows for efficient storage and retrieval of data. It is compliant
with the [PSR-16 standard](https://www.php-fig.org/psr/psr-16/).

In the Spiral Framework, the RoadRunner is used as the underlying technology for the cache data layer, which provides
several benefits. RoadRunner is written in Go, a language known for its performance and efficiency, and is designed to
handle high-performance and concurrent workloads. Additionally, the
RoadRunner [Key-Value plugin](https://roadrunner.dev/docs/plugins-kv/) provides a wide range of storage options,
including popular solutions such as Redis Server or Memcached, as well as options that do not require a separate server,
such as BoltDB. By using RoadRunner, the framework can take advantage of its performance and efficiency to handle
caching more efficiently, which can ultimately improve the overall performance of the application.

## Installation

To enable the component, you just need to add `Spiral\Cache\Bootloader\CacheBootloader` to the bootloader's list:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cache\Bootloader\CacheBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
    // ...
];
```

> **Note**
> Make sure that you have the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package installed.
> This package provides the necessary classes to integrate Cache component with RoadRunner KV plugin.

## Configuration

The configuration file should be located at `app/config/cache.php`. It allows for configuring different storage options
and setting the default storage.

For example, the configuration file might look like this:

```php app/config/cache.php
use Spiral\Cache\Storage\ArrayStorage;
use Spiral\Cache\Storage\FileStorage;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default storage
     * -------------------------------------------------------------------------
     * 
     * The key of one of the registered cache storages to use by default.
     */
    'default' => env('CACHE_STORAGE', 'redis'),
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases
     * -------------------------------------------------------------------------
     * 
     * Aliases, if you want to use domain specific storages.
     */
    'aliases' => [
        'user-data' => 'rr-redis'
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Storages
     * -------------------------------------------------------------------------
     * 
     * Here you may define all of the cache "storages" for your application as well as their types.
     */
    'storages' => [
        'rr-redis' => [
            'type' => 'roadrunner',
            'driver' => 'redis',
        ],

        'rr-local' => [
            'type' => 'roadrunner',
            'driver' => 'local',
        ],
        
        'local' => [
            'type' => 'array',
        ],
        
        'file' => [
            'type' => 'file',
            'path' => directory('runtime') . 'cache',
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases for storage types
     * -------------------------------------------------------------------------
     */
    'typeAliases' => [
        'array' => ArrayStorage::class,
        'file' => FileStorage::class,
    ],
];
```

The `aliases` option allows you to give domain-specific names to storages that are defined in the `storages` section
of the configuration file. This makes it easier to manage and maintain the cache data for different parts of the
application.

For example, you can have a storage called `array` in the `storages` section, and then define an alias
called `user-data` in the aliases section, which refers to the `array` storage. This way, you can use the `user-data`
alias throughout your application to refer to the `array` storage.

The `aliases` section also allows you to request cache storage `Spiral\Cache\CacheStorageProviderInterface` by alias
name, which will be resolved to the actual storage instance. This allows you to reference the cache storage by a more
meaningful name throughout your application, rather than using the internal storage name. If you need to change the
storage in the future, you just need to update the aliases section of the configuration file, and it will affect all the
instances where the alias is used in the application, making it easy to maintain and change the cache storage without
affecting the rest of the code.

And the configuration file for the RoadRunner KV plugin:

```yaml .rr.yaml
version: '2.7'

kv:
  local:
    driver: memory
    config:
      interval: 60
  redis:
    driver: redis
    config:
      addrs:
        - localhost:6379
...
```

> **Note**
> Read more about the configuration of the RoadRunner KV plugin in
> the [RoadRunner documentation](https://roadrunner.dev/docs/plugins-k).

## Usage

The Spiral Framework's cache component allows you to work with the cache using two interfaces:
`Psr\SimpleCache\CacheInterface` and `Spiral\Cache\CacheStorageProviderInterface`.

> **Note**
> You can use the `cache` and `cacheManager` prototype properties. Read more about the
> prototypes in the [The Basics — Prototyping](../basics/prototype.md) section.

### Default storage

By using `Psr\SimpleCache\CacheInterface`, you can access the default cache storage that is defined in the configuration
file. This can be injected into a class using dependency injection, as shown in the example below:

```php
namespace App\Service;

use Psr\SimpleCache\CacheInterface;

final class UserService
{
    public function __construct(
        private readonly CacheInterface $cache,
    ) {
    }

    public function find(int $id): User
    {
        // ...
    }
}
```

### Cache storage provider

On the other hand, by using `Spiral\Cache\CacheStorageProviderInterface`, you can get a specific storage by its string
key from the configuration file. This allows you to access the cache storage for a specific domain or use case.

```php
namespace App\Service;

use Spiral\Cache\CacheStorageProviderInterface;

class UserService
{
    private readonly CacheInterface $cache;
  
    public function __construct(CacheStorageProviderInterface $provider) 
    {
        $this->cache = $provider->storage('user-data');
    }

    public function find(int $id): User
    {
        //...
    }
}
```

In the example provided, a domain-specific storage `user-data` is injected into the `UserService` class. This allows you
to access the cache storage that is specifically tailored to the needs of the user data, and use it throughout the
class to store, retrieve, and delete user data from the cache.

By using this approach, you can easily manage and maintain the cache data for different parts of the application, and
easily switch between different storage options as needed.

### Retrieving items

To get an item from the cache, you can use the `get` method. If the item doesn't exist, it'll return `null`. You can
also specify a default value to be returned if the item doesn't exist.

```php
$data = $this->cache->get('key');

$data = $this->cache->get('key', 'default');
```

If you need to get multiple items at once, you can use the `getMultiple` method and pass in an array of keys.

```php
$data = $this->cache->getMultiple(['key', 'other']);
```

### Checking for item existence

If you want to check if an item exists in the cache, you can use the `has` method. It'll return `true` if the item
exists, and `false` if it doesn't.

```php
if ($this->cache->has('key')) {
    // ...
}
```

### Storing items

To store an item in the cache, you can use the `set` method.

```php
$this->cache->set(key: 'key', value: ['some' => 'data'], ttl: 3600);
```

You can also specify how long the item should be stored for (TTL) by passing in the number of `seconds` or a
`DateInterval` or `DateTimeInterface` object.

```php
$this->cache->set(key: 'key', value: ['some' => 'data'], ttl: \Carbon\Carbon::now()->addHour());
```

If no value is specified, the cache value will be stored indefinitely (or for as long as the underlying storage driver
allows).

```php
$this->cache->set(key: 'key', value: ['some' => 'data']);
```

You can also store multiple items at once by using the `setMultiple` method and passing in an array of key/value pairs.

```php
$this->cache->setMultiple(values: ['key' => ['some' => 'data'], 'other' => ['foo' => 'bar']], ttl: 3600);
```

### Removing items

If you want to remove an item from the cache, you can use the `delete` method.

```php
$this->cache->delete('key');
```

You can also remove multiple items at once by using the `deleteMultiple` method and passing in an array of keys.

```php
$this->cache->deleteMultiple(['key', 'other']);
```

### Clearing the cache

To remove all items from the cache, you can use the `clear` method.

```php
$this->cache->clear();
```

## Custom storage

The component allows for the integration
of [custom storage](https://packagist.org/providers/psr/simple-cache-implementation) methods through configuration
files, making it versatile and easy to use. So you can use any of the available PSR-16 implementations.

In order to use a custom storage you need to register it in the `storages` section of the configuration file, and
specify the type of storage using the `type` key, and any necessary configuration options for that storage.

You will also need to register the class in the `typeAliases` section of the configuration file, so that Spiral knows
how to instantiate your custom storage.

## Events

| Event                            | Description                                                                 |
|----------------------------------|-----------------------------------------------------------------------------|
| `Spiral\Cache\Event\CacheHit`    | The Event will be fired when data is successfully retrieved from the cache. |
| `Spiral\Cache\Event\CacheMissed` | The Event will be fired if the requested data is not found in the cache.    |
| `Spiral\Cache\Event\KeyDeleted`  | The Event will be fired `after` the data is removed from the cache.         |
| `Spiral\Cache\Event\KeyWritten`  | The Event will be fired `after` the data is stored in the cache.            |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
