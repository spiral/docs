# Bootloaders
In many cases, when you would like to move or split your code from application files, bootloader will becomed required.

> Make sure you read about [container and di](/framework/container.md).

## What is bootloader
Bootloader is special class which used to declare needed container bindings and factory method. On other hand you can use bootloader to exectute some code at moment when your environment is loading.

## Bootloader bindings
Let's try to image situation that your code provides set of interfaces and default implementation for them. To make your interfaces work in code you have to configure container first, this can be done in App bootstrap method:

```php
$this->container->bind(SomeInterface::class, SomeClass::class);

//or if concrete object needed
$this->container->bind(SomeInterface::class, function(...){
    return new SomeClass(...);
});
```

Now we can use SomeInterface in our constructor and methods injections. Let's try to archieve same goal using bootloader:

```php
class SomeBootloader extends Bootloader
{
    protected $bindings = [
        SomeInterface::class => SomeClass::class
    ];
    
    protection $singletons = [
        OtherInterface::class => [self::class, 'createOther']
    ];
    
    //Automatically resolved by container
    public function createOther(SomeInterface $class, ...)
    {
        return new OtherClass($class);
    }
}
```

The only thing we have to do now is add such bootloader class into our application, you can simply modify property $load in your App class.

```php
protected $load = [
    //...
    Bootloaders\SomeBootloader::class
];
```

Once done you will be able to use declared bindings and factories in your application. 

> Framework bundle caches bootloader bindings so there is no need to load such classes on every request, this means that any modifications in already registred bootloaders has to be decalread to application using console command "app:reload". On other bootloading cache provides ability to connect multiple extensions or modules without getting much performance penalty on that.

## Booting
TODO
