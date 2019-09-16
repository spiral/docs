# Dumping Variables
Use `Spirl\Debug\Dumper` class or function `dump` to view content of your variables and instances without xDebug:

```php
protected function indexAction(Dumper $dumper, MemcacheStore $memcacheStore)
{
    $dumper->dump($memcacheStore);
}
```
```php
protected function indexAction(MemcacheStore $memcacheStore)
{
    dump($memcacheStore);
}
```

![Dump](https://raw.githubusercontent.com/spiral/guide/master/resources/dumps.png)

The Spiral `Dumper` supports the `__debugInfo` [method](http://php.net/manual/en/language.oop5.magic.php) which allows you to dump only important information. 

## Usage
In your code (works in web and CLI SAPIs):

```php
use Spiral\Debug;

$d = new Debug\Dumper();

$d->dump($variable);
```

Dump to Log:

```php
use Spiral\Debug;

$d = new Debug\Dumper($loggerInterface);

$d->dump($variable, Debug\Dumper::LOGGER);
```

Dump to STDERR:

```php
use Spiral\Debug;

$d = new Debug\Dumper($loggerInterface);

$d->dump($variable, Debug\Dumper::STDERR);
```

Force dump to STDERR with color support:

```php
use Spiral\Debug;

$d = new Debug\Dumper($loggerInterface);
$d->setRenderer(Debug\Dumper::STDERR, new Debug\Renderer\ConsoleRenderer());

$d->dump($variable, Debug\Dumper::STDERR);
```

> You can use function `dump` as a shortcut. Use `dumprr` to dump to RoadRunner error log.

## Internal Fields
To hide certain fields of your objects without `__debugInfo` method, add an "@internal" annotation to your property.

```php
class TestService
{
    /**
     * @internal
     * @var ContainerInterface
     */
    protected $container = null;
}
```
