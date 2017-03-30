# Dumps
Use `Dumper` class or function `dump` to view content of your variables and instances without xDebug:

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

To hide certain fields of your objects without __debugInfo method, add an "@invisible" annotation to your property.

```php
class TestService extends Service
{
    /**
     * Dumping this property will give us nothing, so we can hide it.
     *
     * @invisible
     * @var ContainerInterface
     */
    protected $container = null;
}
```

Dump function support multiple return and destination options.

```php
protected function indexAction(MemcacheStore $memcacheStore)
{
    //Output buffer
    dump($memcacheStore, Dumper::OUTPUT_ECHO);
    
    //Dump information in Spiral\Debug\Debugger log using print_r
    dump($memcacheStore, Dumper::OUTPUT_LOG);

    //Dump information in Spiral\Debug\Debugger log with nice formatting
    dump($memcacheStore, Dumper::OUTPUT_LOG_NICE);

    //Return dump as string
    $dump = dump($memcacheStore, Dumper::OUTPUT_RETURN);

}
```