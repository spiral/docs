# Cache
Spiral provides easy to use access to cache implementation using the `Spiral\Cache\CacheInterface` (provider) and `Spiral\Cache\StoreInterface` interfaces. Both of this interfaces already pre-mapped in spiral to their implementations and ready to be used.

## Cache Provider
To start let's view CacheInterface to understand how our `CacheProvider` works:

```php
interface CacheInterface
{
    /**
     * Create specified or default cache store. This function will load cache adapter if it
     * was not initiated, or fetch it from memory.
     *
     * @param string $store Keep null, empty or not specified to get default cache adapter.
     * @return StoreInterface
     * @throws CacheException
     */
    public function store($store = null);
}
```

As you can see, provider declares only one method related to creating needed cache store by it's name and configuration options (optional), you can skip store name - in this case you will get default adapter. Inside our code (use Controller action as example) we can get access to cache provider using three different bindings:

```php
protected function indexAction(CacheInterface $cache, CacheManager $provider)
{
    //True
    dump($cache === $provider);
    
    $viaContainer = $this->container->get(CacheInterface::class);
    
    //We can also use short binding defined in default implementation of spiral Core
    //and available in every Service
    $store = $this->cache->store();    
}
```

You can check configuration of cache component and it's adapters in `app/config/cache.php` file:

```php
use Spiral\Cache\Stores;

return [
    /*
     * Class name of default cache store to be used by application. You can all request specific
     * store class in your dependencies.
     */
    'store'  => env('CACHE_STORE', 'files'),
    'stores' => [
        /*
         * Slow cache store to be used in develop environments.
         */
        'files'    => [
            'class'     => Stores\FileStore::class,
            'directory' => directory('runtime') . 'cache/',
            'extension' => 'cache'
        ],
        'xcache'   => [
            'class'  => Stores\XCacheStore::class,
            'prefix' => 'spiral:'
        ],
        'apc'      => [
            'class'  => Stores\APCStore::class,
            'prefix' => 'spiral:'
        ],
        'memcache' => [
            'class'   => Stores\MemcacheStore::class,
            'prefix'  => 'spiral:',
            'options' => [],
            'servers' => [
                ['host' => 'localhost', 'port' => 11211, 'persistent' => true],
                /*{{memcache.servers}}*/
            ]
        ],
        /*{{stores}}*/
    ]
];
```

Once we get an instance of CacheStore we can move to the next step.

## Cache Store
Again, let's review StoreInterface to better understand how it works:

```php
interface StoreInterface
{
    /**
     * Check if store is working properly. Please check if the store drives exists, files are
     * writable, etc.
     *
     * @return bool
     */
    public function isAvailable();

    /**
     * Check if value is present in cache.
     *
     * @param string $name Stored value name.
     * @return bool
     * @throws StoreException
     */
    public function has($name);

    /**
     * Get value stored in cache.
     *
     * @param string $name Stored value name.
     * @return mixed
     * @throws StoreException
     */
    public function get($name);

    /**
     * Save data in cache. Method will replace values created before.
     *
     * @param string $name
     * @param mixed  $data
     * @param int    $lifetime Duration in seconds until the value will expire.
     * @return mixed
     * @throws StoreException
     */
    public function set($name, $data, $lifetime);

    /**
     * Store value in cache with infinite lifetime. Value will only expire when the cache is
     * flushed.
     *
     * @param string $name
     * @param mixed  $data
     * @return mixed
     * @throws StoreException
     */
    public function forever($name, $data);

    /**
     * Delete data from cache.
     *
     * @param string $name Stored value name.
     * @throws StoreException
     */
    public function delete($name);

    /**
     * Increment numeric value stored in cache. Must return incremented value.
     *
     * @param string $name
     * @param int    $delta How much to increment by. Set to 1 by default.
     * @return int
     * @throws StoreException
     */
    public function inc($name, $delta = 1);

    /**
     * Decrement numeric value stored in cache. Must return decremented value.
     *
     * @param string $name
     * @param int    $delta How much to decrement by. Set to 1 by default.
     * @return int
     * @throws StoreException
     */
    public function dec($name, $delta = 1);

    /**
     * Read item from cache and delete it afterwards.
     *
     * @param string $name Stored value name.
     * @return mixed
     * @throws StoreException
     */
    public function pull($name);

    /**
     * Get the item from cache and if the item is missing, set a default value using Closure.
     *
     * @param string   $name
     * @param int      $lifetime
     * @param callback $callback Callback should be called if a value doesn't exist in cache.
     * @return mixed
     * @throws StoreException
     */
    public function remember($name, $lifetime, $callback);

    /**
     * Flush all values stored in cache.
     *
     * @throws StoreException
     */
    public function flush();
}
```

Based on following methods can perform different operations:

```php
protected function indexAction()
{
    $store = $this->cache->store();
    
    //Store value for 1 day
    $store->set('name', 'value', 86400);

    //Check if value presented in cache
    dump($store->has('name'));
    
    //Get value from cache (if no value exists null will be returned)
    dump($store->get('name'));
    
    //Store value forever
    $store->forever('nameB', 1);

    //Increment stored value by 2
    $store->inc('nameB', 2);
    
    //Decrement stored value by 2
    $store->dec('nameB', 2);
    
    //Fetch value from cache and remove it after
    dump($store->pull('nameB'));
    
    //Fetch value from cache or generate it using provided function
    dump($store->remember('nameC', 10, function() {
        dump('we are called!');
        
        return 'ABC';
    }));
}
```

## Supported Cache Stores
At this moment spiral support 4 different cache adapters by default and one adapter provided by external module (Redis):

Adapter        | Description           
---            | ---                   
FileStore      | Default cache using serialized files to store data, you should be using this store only in development.            
MemcacheStore  | Memcache store can work with both "memcache" and "memcached" extensions using set of defined servers. Memcahe is preffered extension.    
APCStore       | Utilizes APC to store data, might be limited in memory.                                                               
XCacheStore    | Utilizes XCache to store data, might be limited in memory.                                                            
RedisStore     | Available only with Redis extension, will use defined predis client to store data into.                               

> Spiral does not support tagged cache at this moment (be free to propose your ideas).