# Advanced — Static analysis

The tokenizer component is a tool for discovering and analyzing code in specified directories. The most used
functionality of locating class declarations is provided by `Spiral\Tokenizer\ClassesInterface`.

> **Note**
> The tokenizer component is pre-installed with all framework bundles.

## Class Locator

To use the tokenizer component, you will need to use the `Spiral\Tokenizer\ClassesInterface` interface. This interface
provides a getClasses method which allows you to search for classes by name, interface, or trait.

Here is an example of how to use this method to search for classes that implement the
`\Psr\Http\Server\MiddlewareInterface` interface:

```php
public function findClasses(ClassesInterface $classes): void
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

The `getClasses` method will then return an array of `ReflectionClass` objects representing the classes found.

By default, the tokenizer component will only search for classes in the `app` directory. However, you can add additional
directories to be searched using the `Spiral\Bootloader\TokenizerBootloader`.

```php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function boot(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addDirectory($directories->get('vendor') . 'spiral/validator/src');
    }
}
```

And here is an example of how to add an additional directory using the `app\config\tokenizer.php` config file:
```php
return [
    'directories' => [
        directory('app'),
        directory('vendor') . 'spiral/validator/src',
    ],
];
```

You can also specify directories to be excluded from the search using the exclude option:

```php
return [
    'directories' => [
        //...
    ],
    'exclude' => [
        directory('resources'),
        directory('config'),
        'tests',
        'migrations',
    ],
];
```

> **Note**
> The class lookup is not a fast process, only add necessary directories.

## Scoped Class Locator

If you need to search for classes in a large number of directories, the tokenizer component may suffer from poor
performance. In this case, you can use the scoped class locator to improve performance.

With the scoped class locator, you can define directories to be searched within named scopes. This allows you to
selectively search only the directories that are relevant to your current task.

To use the scoped class locator, you will need to define your scopes in the `app\config\tokenizer.php` config file. Here
is an example of how to define a scope named `scopeName` that searches the `app/Directory` directory:

```php
// file app/config/tokenizer.php
return [
    'scopes' => [
        'scopeName' => [
            'directories' => [
                directory('app') . 'Directory',
            ],
            'exclude' => [
                directory('app') . 'Directory/Excluded',
            ]
        ],
    ]
];
```

> **Note**
> With the `exclude` parameter, we can exclude some directories from the search.

The `Spiral\Tokenizer\ScopedClassesInterface` interface provides a `getScopedClasses` method that allows you to search
for classes within a specific scope.

To use the method, you will need to pass in the name of the `scope` as an argument. The method will then return an
array of `ReflectionClass` objects representing the classes found within that scope.

```php
use Spiral\Tokenizer\ScopedClassesInterface;

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

## Class Listeners

The Tokenizer Class Listeners is a way to use the `Spiral\Tokenizer\ClassesInterface` interface in a more efficient
manner, particularly when working with large codebases.

Normally, when you use the `ClassesInterface` to search for classes, the tokenizer component will scan the specified
directories every time the `getClasses` method is called. This can be slow, particularly if you have many components
that make repeated calls to the method.

To improve performance, it allows you to register listeners that will be notified when a class is found by
the `ClassesInterface`. This means that the tokenizer component will only need to scan the directories
once, during application bootstrapping. After the initial scan, the tokenizer will iterate over all the found classes
and send information about each class to all registered listeners.

### Usage

To use this feature, you will need to include `Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader`
bootloader in your project at the top of bootloader's list:

```php app/src/Application/Kernel.php
protected const LOAD = [
    \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
    // ...
];
```

Now you can create a listener class that implements the `Spiral\Tokenizer\TokenizationListenerInterface` interface.
This interface requires you to implement a `listen` method, which will be called for each class that is found by
the `ClassesInterface`.

In addition `TokenizationListenerInterface` also includes a `finalize` method. This method is called after all classes
have been iterated, and can be used to perform any final processing on the classes that have been found during
listening.

```php
use Spiral\Attributes\ReaderInterface;

class RouteAttributeListener implements TokenizationListenerInterface
{
    private array $attributes = [];

    public function __construct(
        private readonly ReaderInterface $reader,
        private readonly RouterInterface $router
    ) {
    }
    
    public function listen(\ReflectionClass $class): void
    {
        foreach ($class->getMethods() as $method) {
            $route = $this->reader->firstFunctionMetadata($method, Route::class);

            if ($route === null) {
                continue;
            }

            $this->attributes[] = [$method, $route];
        }
    }

    public function finalize(): void
    {
        foreach ($this->attributes as [$method, $route]) {
            $this->router->addRoute(...);
        }
    }
}
```

To register your listener, you will need to use the `Spiral\Tokenizer\TokenizerListenerRegistryInterface`.

Here is an example of how to register a listener:

```php
use Spiral\Tokenizer\TokenizerListenerRegistryInterface;

class AppBootloader extends Bootloader
{
    public function init(
        TokenizerListenerRegistryInterface $listenerRegistry,
        RouteAttributeListener $listener
    ): void {
        $listenerRegistry->addListener($listener);
    }
}
```

> **Warning**
> To ensure that your listeners are called correctly, make sure to register them in bootloaders from within the `LOAD`
> section of the application Kernel. Listeners will not be called if you register them  within the `APP` kernel section.

Overall, the Tokenizer Listeners are a useful way to improve performance when using the `ClassesInterface` to search
for classes in a large codebase. By registering listeners and allowing the tokenizer component to scan the directories
only once, you can reduce the number of times the directories are scanned and improve the overall performance of your
application.