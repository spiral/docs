# Framework - Auto Wiring
Spiral Framework attempts to hide the container from your domain layer by providing rich auto-wiring functionality. Though, 
auto-wiring rules are very simple it's important to learn them to avoid framework misbehavior.

## Automatic Dependency Resolution
Framework container is able to automatically resolve the constructor or method dependencies by providing instances
of concrete classes.

```php
class MyController
{
    public function __construct(OtherClass $class, SomeInterface $some)
    {
    }
}
```

In a provided example the container will attempt to provide the instance of `OtherClass` by automatically constructing it. However,
`SomeInterface` would not be resolved unless you have the proper binding in your container.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Please note, Container will try to resolve *all* constructor dependencies (unless you manually provide some values). It means that
all class dependencies must be available or parameter must be declared as optional:

```php
// will fail if `value` dependency not provided
__construct(OtherClass $class, $value)

// will use `null` as `value` if no other value provided
__construct(OtherClass $class, $value = null) 

// will fail if SomeInterface does not point to the concrete implemenation
__construct(OtherClass $class, SomeInterface $some) 

// will use null as value of `some` if no conrete implemation is provided
__construct(OtherClass $class, SomeInterface $some = null) 
```

## Injectors
In addition to regular method injections, the container is able to resolve the injection context automatically. Such technique provides us the ability to request multiple databases using the following statement:

```php
protected function index(Database $primary, Database $secondary)
{
    dump($primary);
    dump($secondary);
}
```

> Where `primary` and `secondary` are database names.

Implement `Spiral\Core\Container\InjectorInterface` to create injection factory:

Now we have to define class responsible for such injections:

```php
class Injector implements InjectorInterface
{
    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        return new MyClass($context);
    }
}
```

Where MyClass is:

```php
class MyClass 
{
    public function __constrcut(string $name)
    {
        //...
    }
}
```

Make sure to register inject in the container:

```php
$container->bindInjector(MyClass::class, Injector::class);
```

As result we can request instance of `MyClass` using method arguments:

```php
public function method(Name $john, Name $bob)
{
    dump($john);
    dump($bob);
}
```

You can always bypass contextual injection in `Spiral\Core\FactoryInterface`:

```php
dump($factory->make(Name::class, ['name' => 'abc']));
```
