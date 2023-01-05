# Cache

The Spiral Framework provides a component for data caching. The component provides implementation of `PSR-16` 
cache storage in `files` and `PHP arrays`.
It also provides an easy way to integrate your own storages by configuration file.

The component is available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\Cache\Bootloader\CacheBootloader` to the bootloader's list, 
which is located in the class of your application.

```php
protected const LOAD = [
    // ...
    \Spiral\Cache\Bootloader\CacheBootloader::class,
    // ...
];
```

## Configuration

The configuration file for this component should be located at `app/config/cache.php`. Within this file, you may
configure available storages and default storage.

For example, the configuration file might look like this:

```php
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
    'default' => env('CACHE_STORAGE', 'array'),
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases
     * -------------------------------------------------------------------------
     * 
     * Aliases, if you want to use domain specific storages.
     */
    'aliases' => [
        'user-data' => 'array'
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Storages
     * -------------------------------------------------------------------------
     * 
     * Here you may define all of the cache "storages" for your application as well as their types.
     */
    'storages' => [
        'array' => [
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

## Usage

To work with the cache, you can use `Psr\SimpleCache\CacheInterface` or `Spiral\Cache\CacheManager`.

### CacheInterface

The cache storage can be injected from a container using the `Psr\SimpleCache\CacheInterface`. It will refer to the 
default storage.

```php
namespace App\Service;

use Psr\SimpleCache\CacheInterface;

class MyService
{
    public function __construct(
        private readonly CacheInterface $cache,
    ) {
    }

    public function someMethod(): void
    {
        // save data in the cache
        $this->cache->set(key: 'key', value: ['some' => 'data'], ttl: 3600);

        \dump($this->cache->has('key')); // true

        // retrieve data from the cache
        $data = $this->cache->get('key');

        // delete data from the cache
        $this->cache->delete('key');
        
        // delete all data 
        $this->cache->clear();
        
        // work with multiple items
        $this->cache->setMultiple(values: ['key' => ['some' => 'data'], 'other' => ['foo' => 'bar']], ttl: 3600);
        
        $data = $this->cache->getMultiple(['key', 'other']);
        
        $this->cache->deleteMultiple(['key', 'other']);
    }
}
```

### CacheManager

Using the `Spiral\Cache\CacheManager`, you can get a specific storage by storage string key from config.

```php
namespace App\Service;

use Spiral\Cache\CacheManager;

class MyService
{
    public function __construct(
        private readonly CacheManager $cache,
    ) {
    }

    public function someMethod(): void
    {
        // save data in the cache
        $this->cache
            ->storage('array')
            ->set(key: 'key', value: ['some' => 'data'], ttl: 3600);

        \dump($this->cache->storage('array')->has('key')); // true

        // retrieve data from the cache
        $data = $this->cache
            ->storage('array')
            ->get('key');

        // delete data from the cache
        $this->cache
            ->storage('array')
            ->delete('key');

        // delete all data
        $this->cache
            ->storage('array')
            ->clear();

        // work with multiple items
        $this->cache
            ->storage('array')
            ->setMultiple(values: ['key' => ['some' => 'data'], 'other' => ['foo' => 'bar']], ttl: 3600);
        
        $data = $this->cache
            ->storage('array')
            ->getMultiple(['key', 'other']);
        
        $this->cache
            ->storage('array')
            ->deleteMultiple(['key', 'other']);
    }
}
```

> **Note**
> You can use the `cache` and `cacheManager` prototype properties.

## Events

| Event                          | Description                                                                 |
|--------------------------------|-----------------------------------------------------------------------------|
| Spiral\Cache\Event\CacheHit    | The Event will be fired when data is successfully retrieved from the cache. |
| Spiral\Cache\Event\CacheMissed | The Event will be fired if the requested data is not found in the cache.    |
| Spiral\Cache\Event\KeyDeleted  | The Event will be fired `after` the data is removed from the cache.         |
| Spiral\Cache\Event\KeyWritten  | The Event will be fired `after` the data is stored in the cache.            |

> **Note**
> To learn more about dispatching events, see the [Events](../component/events.md) section in our documentation.
