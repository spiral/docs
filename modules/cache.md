# Cache Bridges
Spiral utilizes PSR-16 (ORM) and PSR-6 (psr7-middlewares) protocols to allow your application to communicate with cache engines.

You are able to choose any caching library which support this mechanisms. To enable cache support in application create PSR interface binding to your implementation or factory:

```php
use Psr\SimpleCache\CacheInterface;

class CacheBootloader extends Bootloader {
    const SINGLETONS = [
        CacheInterface::class => [self::class, 'makeCache']
    ];  

    protected function makeCache(): CacheItemPoolInterface
    {
        return new MyCache();
    }
}
```

## Existed Bridges
Take a look at existed module `spiral/phpfastcache` which creates bridge to [https://github.com/PHPSocialNetwork/phpfastcache](https://github.com/PHPSocialNetwork/phpfastcache) library and adds cache configuration into your application.

`composer require spiral/phpfastcache`
`spiral register spiral/phpfastcache`

Add bootloader `Spiral/PhpFastCache/CacheBootloader` to enable caching. 