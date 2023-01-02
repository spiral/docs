# Autowire

## Introduction

Autowire is a class provided by the Spiral Framework that allows you to delegate options to the container. It allows you to pass specific configuration values to your classes without hardcoding them, which is useful for keeping your configuration separate from your code and for making it easier to modify your application's behavior without changing the code itself.

## Usage
To use Autowire, you will need to create an instance of the `Spiral\Core\Container\Autowire` class and pass it a class name or an object. 
You can also specify optional parameters to be passed to the class's constructor.

Here is an example of how to use Autowire:

```php
class MyClass
{
    public function __construct(
        private SomeClass $class, // will be resolved by container automatically
        private string $firstOption,
        private string $secondOption
    ) {  
    }
    
    public function handle()
    {
        \var_dump(
            $this->firstOption, 
            $this->secondOption
        );
    }
}
```

```php
$autowire = new Autowire(
    MyClass::class, 
    [
        'firstOption' => 'foo', 
        'secondOption' => 'bar'
    ]
);

$class = $autowire->resolve($factory);

$class->handle(); // will return `foo` and `bar`
```

Alternatively, you can use the `Autowire::wire()` static method to create an Autowire instance based on a string or array definition:

```php
$autowire = Autowire::wire([
    'class' => MyClass::class,
    'options' => [
        'firstOption' => 'foo', 
        'secondOption' => 'bar'
    ],
]);

$class = $autowire->resolve($factory);

$class->handle(); // will return `foo` and `bar`
```

The `resolve()` method of the `Autowire` class is used to retrieve an instance of the class specified when the `Autowire` instance was created.
It takes two parameters:
- `$factory` (`Spiral\Core\FactoryInterface`): An instance of the `FactoryInterface` class, which is used to create new instances of classes.
- `$parameters` (`array`): An optional array of parameters to be passed to the class's constructor. These parameters will be merged with any parameters specified when the Autowire instance was created.

