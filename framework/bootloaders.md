# Bootloaders
The Bootloader classes responsible for pre-initialization on your application parts, configure proper container bindings and environment initialization (i.e. http routes). In many cases bootloaders will be used to represent 3rd party applications.

> Make sure you read about [Container and DI](/framework/container.md).

## Booting bindings
Ordinary your application will need a lot of interfaces and aliases binded to their implementations, you can define them in your code using following constructions:

```php
$this->container->bind(SomeInterface::class, SomeClass::class);

//or if concrete object needed
$this->container->bind(SomeInterface::class, function(...) {
    return new SomeClass(...);
});
```

Now we can use SomeInterface in our constructor and methods injections.

```php
public function indexAction(SomeInterface $some)
{
    dump($some);
    assert($some instanceof SomeClass);
}
```

Bootloaders make such definitions easier, faster and provide ability to merge multiple bindings into one bucket:

```php
class SomeBootloader extends Bootloader
{
    const BINDINGS = [
        SomeInterface::class => SomeClass::class
    ];
    
    //Only constructed once
    protected $singletons = [
        OtherInterface::class => [self::class, 'createOther']
    ];
    
    protected function createOther(SomeInterface $class, ...)
    {
        return new OtherClass($class);
    }
}
```

> Note that you can make your factory methods private and protected, container will bypass this restriction.

The only thing we have to do now is to add such bootloader class into our application, you can simply modify constant `LOAD` in your App class.

```php
const LOAD = [
    //...
    Bootloaders\SomeBootloader::class
];
```

> You can enable bootloaders cache in your .env file to speed up application initialization a bit.

## Booting code
Bootloaders also allows you to load or execute custom code at moment of application initialization, set constant `BOOT` to true and define method `boot` in order to do that:

```php
class AppBootloader extends Bootloader 
{
    const BOOT = true;

    const BINDIGNS = [
        SomeInterface::class => SomeClass::class
    ];
    
    const SINGLETONS = [
        OtherInterface::class => [self::class, 'createOther']
    ];
    
    //Automatically resolved by container
    public function createOther(SomeInterface $class, ...)
    {
        return new OtherClass($class);
    }

    /**
     * @param HttpDispatcher $http
     */
    public function boot(HttpDispatcher $http)
    {
        $http->addRoute(new Route(
            'route', 'route/<section:[a-z\-]*>',
            'Vendor\Controllers\SomeController::action'
        ));
    }
}
```

> Cache is ignored for such bootloaders.

## Bootloading outside of core
If you want to load your bootloader based on some condition simple utilize `getBootloader()` method of your core application:

```php
public function indexAction()
{
    $this->app->bootloader()->bootload([
        \Vendor\Module\VendorBootloader::class
    ]);
}
```

## Short bindings and Shortcuts
Bootloaders can also be used to define shortcuts to your services:

```php
class MyBootloader extends Bootloader
{
    /**
     * @return array
     */
    const BINDINGINS = [
        'someTable' => [self::class, 'someTable']
    ];

    /**
     * @param DatabaseManager $databases
     */
    public function someTable(DatabaseManager $databases): Table
    {
        return $databases->database('default')->table('some');
    }
}
```

Now you can use such table shortcut in your controllers or service:

```php
public function indexAction()
{
    dump($this->someTable);
}
```

> You can treat shortcuts/[factory methods] as inline functions in C but in a context of active container scope.
