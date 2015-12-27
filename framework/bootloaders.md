# Bootloaders
In many cases, when you would like to move or split your code from application files, bootloader will becomed required.

> Make sure you read about [container and di](/framework/container.md).

## What is bootloader
Bootloader is special class which used to declare needed container bindings and factory methods to make your component operate well. On other hand you can use bootloader to exectute some code at moment when your environment is loading.

## Booting bindings
Let's try to imagine situation when your code provides set of interfaces and default implementation for them. To make sure that your interfaces work in code you have to configure container first, this can be done in App bootstrap method:

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

Let's try to archieve same goal using bootloader (this is only example code):

```php
class SomeBootloader extends Bootloader
{
    protected $bindings = [
        SomeInterface::class => SomeClass::class
    ];
    
    //Only constructed once
    protected $singletons = [
        OtherInterface::class => [self::class, 'createOther']
    ];
    
    //Automatically resolved by container
    public function createOther(SomeInterface $class, ...)
    {
        return new OtherClass($class);
    }
}
```

The only thing we have to do now is to add such bootloader class into our application, you can simply modify property `$load` in your App class.

```php
protected $load = [
    //...
    Bootloaders\SomeBootloader::class
];
```

Once done you will be able to use declared bindings and factories in your application. 

> Framework bundle caches bootloader bindings so there is no need to load such classes on every request, this means that any modifications in already registred bootloaders has to be declared to application using console command "app:reload". On other 
hand, bootloading cache provides ability to connect multiple extensions or modules without getting much performance penalty on that.

## Booting code
Often you might want to execute your code at moment of application bootloading, you can do that by simply declaring specific constant in your bootloader `BOOT` and creating boot method with supported method injection:

```php
class AppBootloader extends Bootloader 
{
    protected $bindings = [
        SomeInterface::class => SomeClass::class
    ];
    
    protected $singletons = [
        OtherInterface::class => [self::class, 'createOther']
    ];
    
    //Automatically resolved by container
    public function createOther(SomeInterface $class, ...)
    {
        return new OtherClass($class);
    }

    /**
     * @param HttpDispatcher       $http
     */
    public function boot(HttpDispatcher $http)
    {
        $http->addRoute(new Route(
            'route', 'route/<section:[a-z\-]*>', 'Vendor\Controllers\SomeController::action'
        ));
    }
}
```

Given Bootloader will automatically register http route at moment of application initialization. You can declare any needed depencies in your boot method arguments.

> Attention, booted Bootloaders are not cached. If you want to add boot method to existed bootloaded - do not forget to execture 'app:reload' after doing that.

## Bootloading outside of core
As you see in default application, using `$load` property is not the only way to mount your bootloaders as you can get access to BootloadManager at any moment:

```php
public function indexAction()
{
    $this->app->bootloader()->bootload([
        \Vendor\Module\VendorBootloader::class
    ]);
}
```

> Attention, this bootloaded will not be cached in memory by default.

## Short bindings and Shortcuts
Sometimes you might need to have simplified access to some of your code, for example database or specific part of request. As you might notice in [Container, Factory, DI](/framework/cotainer.md) section you can create short bindings for some of you classes and services.

However in some cases we can additionally combine short binding with factory methods in Bootloader:

```php
class MyBootloader extends Bootloader
{
    /**
     * @return array
     */
    protected $bindings = [
        'someTable' => [self::class, 'someTable']
    ];

    /**
     * @param DatabaseManager $databases
     * @return \Spiral\Database\Entities\Table
     */
    public function someTable(DatabaseManager $databases)
    {
        return $databases->database('default')->table('some');
    }
}
```

Now you can use such table shortcut in your controllers and services:

```php
public function indexAction()
{
    dump($this->someTable);
}
```

If you would like to get automatic tooltips in your IDE for such shortcut - simply add comment into SharedTrait twin which you can put into your module or find one in app/classes/Bootloaders/Virtual:

```php
/**
 * @property-read \Spiral\Databases\Entities\Table $someTable Binded in MyBootloader
 */
trait SharedTrait 
{
    //...
}
```

> You can treat shortcuts as inline functions in C but in a context of active container.
