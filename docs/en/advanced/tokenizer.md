# Component — Static analysis

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

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addDirectory($directories->get('vendor') . 'spiral/validator/src');
    }
}
```

And here is an example of how to add a directory using the `app/config/tokenizer.php` config file:

```php app/config/tokenizer.php
return [
    'directories' => [
        directory('app'),
        directory('vendor') . 'spiral/validator/src',
    ],
];
```

You can also specify directories to be excluded from the search using the exclude option:

```php app/config/tokenizer.php
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

```php app/config/tokenizer.php
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

The Tokenizer class listeners are a way to use the `Spiral\Tokenizer\ClassesInterface` interface in a more efficient
manner, particularly when working with large codebases.

Normally, when you use the `ClassesInterface` to search for classes, the tokenizer component will scan the specified
directories every time the `getClasses` method is called. This can be slow, particularly if you have many components
that make repeated calls to the method.

To improve performance, the component allows you to register listeners that will be notified when a class is found by
the `ClassesInterface`. This means that directories will only be scanned once, during application bootstrapping. After
the initial scan, the listeners will iterate over all the found classes.

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

### Caching listener targets

To improve the performance of your application, you can use the `Spiral\Tokenizer\Attribute\TargetAttribute`
and `Spiral\Tokenizer\Attribute\TargetClass` attributes to filter the classes and attributes that are processed by
listeners. This allows you o improve the performance of your code by filtering the classes and attributes that are
processed by listeners.

When you use the attributes to filter the classes that are processed by listeners, the component caches the filtered
classes in the `runtime/cache/listeners` directory after the first bootstrapping of your application.

Caching the filtered classes provides several benefits to your application. It greatly reduces the amount of
time required to process your codebase, since the class locator can load the filtered classes from cache rather than
re-scanning your codebase every time your application starts up. This can help to improve the performance of your
application and reduce the time required for application bootstrapping.

By default, caching of the filtered classes is disabled. If you want to enable caching, you can set
the `TOKENIZER_CACHE_TARGETS` environment variable to `true`.

```dotenv .env
TOKENIZER_CACHE_TARGETS=true
```

#### TargetAttribute

It allows you to filter classes based on their attributes. When you specify a target attribute, the class locator will
only process classes that have that attribute. This can be useful if you have a listener that only needs to analyze a
specific type of class, such as a controller class that has a specific routing attribute.

**Here's an example of how to use it:**

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;

#[TargetAttribute(Route::class, useAnnotations: true)]
final class RouteLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

In this example, the `RouteAttributeListener` will only process classes that have the `Route` attribute. This means
that if the class locator finds a class without this attribute, it won't call the `listen` method of this listener.

You can add multiple attributes to your listener class:

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;
use Spiral\Tokenizer\TokenizationListenerInterface;

#[TargetAttribute(Route::class)]
#[TargetAttribute(SymfonyRoute::class)]
class RouteLocatorListener implements TokenizationListenerInterface
{
    public function listen(\ReflectionClass $class): void
    {
        // Do something with classes that have Route or SymfonyRoute attributes
    }
}
```

You can also pass a second parameter `useAnnotations: true` to the attribute to specify that the Tokenizer should look
for the target attribute in the class annotations as well. For example:

#### TargetClass

It works similarly to `TargetAttribute`, but instead of filtering classes based on their attributes, it filters them
based on their type. This is useful if you have a listener that only needs to analyze a specific type of class, such
as controller, classes that implement a specific interface or extend a specific class.

**Here's an example of how to use**

```php
use Spiral\Tokenizer\Attribute\TargetClass;

#[TargetClass(SymfonyCommand::class)]
final class CommandLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

In this example, the listener will process all classes that extend the `SymfonyCommand`. This means that if the class
locator finds a class that extends it, it will call the `listen` method of this listener.

> **Note**
> You can add multiple attributes to your listener class.

### Listener Registration

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
> section of the application Kernel. Listeners will not be called if you register them within the `APP` kernel section.
