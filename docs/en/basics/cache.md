# The Basics — Cache

Caching is a technique that can significantly improve the performance of an application by storing frequently accessed
data in a faster storage medium, such as memory. This reduces the need to recalculate or fetch the data from a slower
source, such as a database. By implementing a caching layer, the application can quickly retrieve the data from the
cache instead of recalculating it or fetching it from the slower source, which can improve the overall response time and
reduce the load on the slower source.

The `spiral/cache` component in Spiral provides a streamlined method for storing and retrieving data. This component 
aligns with the [PSR-16 standard](https://www.php-fig.org/psr/psr-16/) standard, making it a versatile choice for 
developers familiar with PHP standards.

## RoadRunner: The Game-Changer for Caching

When we speak of Spiral's caching prowess, RoadRunner sits at its core. Here's what you should know about RoadRunner:

- **Written in Go:** RoadRunner is all about speed and efficiency. Go is revered for its blistering performance metrics, 
  making it a natural choice for high-octane operations.

- **Tailored for High Concurrency:** It doesn't just perform; it performs in parallel. Designed to shoulder 
  high-performance tasks, it can manage concurrent operations without a hiccup.

- **Diverse Storage Choices through the Key-Value Plugin:** With RoadRunner's 
  [Key-Value plugin](https://roadrunner.dev/docs/plugins-kv/), you're spoilt for choice. Opt for household names like 
  Redis Server and Memcached or go the server-less route with choices like in-memory storage.

- **No PHP Extensions Required:** One of the standout features of using RoadRunner is the elimination of any PHP 
  extensions for Redis or Memcache. Instead, you communicate directly with RoadRunner using RPC.


## Installation

To enable the component, you just need to add `Spiral\Cache\Bootloader\CacheBootloader` to the bootloader's list:

> **Warning**
> Make sure that you have the [spiral/roadrunner-bridge](../start/server.md#roadrunner-bridge) package installed.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cache\Bootloader\CacheBootloader::class,
    \Spiral\RoadRunnerBridge\Bootloader\CacheBootloader::class,
    // ...
];
```

> **Note**
> The `spiral/roadrunner-bridge` package allows you to use RoadRunner
> [kv plugin](https://roadrunner.dev/docs/plugins-kv/) with Spiral. This package provides RPC api for KV
> and a bootloader for your application.

## Configuration

To get started, place your configuration file at `app/config/cache.php`. This file acts as your go-to place for 
configuring storage options and setting your default storage.

Here's a sample of what your `cache.php` configuration might look like:

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
        'user-data' => 'rr-redis',
        'blog-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'blog_'
        ],
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

### Working with Aliases

Aliases are your shortcut to referencing cache storages. They simplify the code and make it more readable.

For example, you have a storage named `rr-redis` and you'd prefer to reference it as `user-data` in your application. 
The `aliases` section helps you do just that!

What's more, you can fetch cache storage using the `Spiral\Cache\CacheStorageProviderInterface` by the alias name. It 
then smartly maps to the actual storage, making your code neater and more manageable.

### Prefixing Cache Keys:

Prefixes play a pivotal role when working with cache aliases. They ensure that your cache keys remain unique, 
safeguarding against any accidental overlaps or clashes. By attaching a `prefix` to an alias, each cache key under that 
alias automatically inherits this prefix. This results in a structured, organized, and clash-free cache system.

Incorporating prefixes into your cache.php is straightforward:

```php app/config/cache.php
return [
    // ...

    'aliases' => [
        'user-data' => [
            'storage' => 'rr-redis',
            'prefix' => 'user_'
        ],
    ],
];
```

In the example above, any cache key under the `user-data` alias will automatically be prefixed with `user_`. So, a cache
key like `profile` would be stored as `user_profile`.

### RoadRunner KV Plugin configuration

And the configuration file for the RoadRunner KV plugin:

```yaml .rr.yaml
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

> **See more**
> Read more about the configuration of the RoadRunner KV plugin in
> the [RoadRunner documentation](https://roadrunner.dev/docs/plugins-kv).

## Usage

Spiral's cache component allows you to work with the cache using two interfaces:
`Psr\SimpleCache\CacheInterface` and `Spiral\Cache\CacheStorageProviderInterface`.

> **See more**
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
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: 3600
);
```

You can also specify how long the item should be stored for (TTL) by passing in the number of `seconds` or a
`DateInterval` or `DateTimeInterface` object.

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data'], 
    ttl: \Carbon\Carbon::now()->addHour()
);
```

If no value is specified, the cache value will be stored indefinitely (or for as long as the underlying storage driver
allows).

```php
$this->cache->set(
    key: 'key', 
    value: ['some' => 'data']
);
```

You can also store multiple items at once by using the `setMultiple` method and passing in an array of key/value pairs.

```php
$this->cache->setMultiple(
    values: [
        'key' => ['some' => 'data'], 
        'other' => ['foo' => 'bar']
    ], 
    ttl: 3600
);
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
