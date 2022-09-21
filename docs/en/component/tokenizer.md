# Tokenizer

A lot of Spiral components based on automatic code discovery and analysis. The most used functionality of locating class
declarations is provided by `Spiral\Tokenizer\ClassesInterface`.

> **Note**
> Tokenizer component is pre-installed with all framework bundles.

## Class Locator

Use `Spiral\Tokenizer\ClassesInterface` to find available classes by their name, interface or trait:

```php
public function findClasses(ClassesInterface $classes): void
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

By default, the component will be looking for classes available in the `app` directory only. You can add any other
directory using `Spiral\Bootloader\TokenizerBootloader`:

```php
public function boot(TokenizerBootloader $tokenizer)
{
    $tokenizer->addDirectory(directory('vendor') . 'name/extension/src');
}
```

> **Note**
> Attention, class lookup is not a fast process, only add necessary directories.

## Scoped Class Locator

To improve the performance of class searching, you can configure the search scope.

First of all, let's add `scopes` in the configuration file:

```php
// file app/config/tokenizer.php
return [
    'scopes' => [
        'scopeName' => [
            'directories' => [
                directory('app/Directory')
            ],
            'exclude' => [
                directory('app/Directory/Other')
            ]
        ],
    ]
];
```

> **Note**
> With the `exclude` parameter, we can exclude some directories from the search.

Use `Spiral\Tokenizer\ScopedClassesInterface` to find available classes by the scope name in scope directories:

```php
final class SomeLocator
{
    public function __construct(
        private readonly ScopedClassesInterface $locator
    ) {
    }

    public function findDeclarations(): array
    {
        foreach ($this->locator->getScopedClasses('scopeName') as $class) {
            // ...
        }
    }
}
```

## Listeners

For even more performance we recommend you to use tokenizer listeners instead of class locator. 

There is a new feature Tokenizer Listener in the Spiral Framework 3.0. 

### How it works

The Spiral Framework used to scan registered directories in tokenizer every time when you request them via
`Spiral\Tokenizer\ClassesInterface::getClasses` method. For example, if an application has 6 components that request
classes from class locator via `getClasses` method, it will scan rectories 6 times.

With the tokenizer listeners it will start iterating all found classes by `Spiral\Tokenizer\ClassesInterface` after 
bootloading all application bootloaders only once and then send information about every found class to all registered 
tokenizer listeners. In this case 6 components will register their listeners and will filter received class by their 
rules and will handle them.

To start using Tokenizer listeners you just need to include `Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader` 
bootloader in your project at the top of bootloaders list:

```php
protected const LOAD = [
    Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
    // ...
];
```

Create a listener:

```php

use Spiral\Attributes\ReaderInterface;

class MyAttributeListener implements TokenizationListenerInterface
{
    private array $attributes = [];

    public function __construct(
        private readonly ReaderInterface $reader,
        private readonly MyAttributeRegistry $registry
    ) {
    }
    
    /**
     * The method will be fired for every found class by a class locator.
     */
    public function listen(\ReflectionClass $class): void
    {
        // Find MyAttribute attribute in the given class
        $attr = $this->reader->firstClassMetadata($class, MyAttribute::class);

        if ($attr === null) {
            continue;
        }
    
        $this->attributes[] = $attr;
    }

    /**
     * The method will be fired after a class locator will finish iterating of found classes.
     */
    public function finalize(): void
    {
        // Register found attributes in your registry
        $this->registry->registerAttributes(
            $this->attributes
        );
    }
}
```

And then register it in your bootloader

```php
use Spiral\Tokenizer\TokenizerListenerRegistryInterface;

class AppBootloader extends Bootloader
{
    public function init(
        TokenizerListenerRegistryInterface $listenerRegistry,
        MyAttributeListener $listener
    ): void {
        $listenerRegistry->addListener($listener);
    }
}
```