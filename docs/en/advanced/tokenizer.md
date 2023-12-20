# Component — Static analysis

Tokenizer is your go-to tool to quickly find and explore code within specified directories. Its main role is to help you
identify class declarations. Good news:

> **Note**
> Tokenizer comes with the framework, so you won't have to fuss about setting it up.

## Locators

Locators are like your code's searchlight. They help you find specific pieces of code.

### Classes Locator

If you're aiming to locate classes, turn to `Spiral\Tokenizer\ClassesInterface`. Using this, you can search for classes
based on their name, the interfaces they use, or the traits they incorporate.

Here's a quick example. Let's say you want to locate all classes that use the `\Psr\Http\Server\MiddlewareInterface`
interface:

```php
use Spiral\Tokenizer\ClassesInterfac;

public function findClasses(ClassesInterface $classes): void
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

The `getClasses` method will then return an array of `ReflectionClass` objects representing the classes found.

### Enums Locator

In case you're on the lookout for enumerations, the `Spiral\Tokenizer\EnumsInterface` is what you need. It comes with
a `getEnums` method to help you on your hunt:

```php
use Spiral\Tokenizer\EnumsInterface;

public function findEnums(EnumsInterface $enums): void
{
    foreach ($enums->getEnums() as $enum) {
        dump($enum->getFileName());
    }
}
```

### Interfaces Locator

If you wish to find specific interfaces, the `Spiral\Tokenizer\InterfacesInterface` has got you covered. Use
its `getInterfaces` method like so:

```php
use Spiral\Tokenizer\InterfacesInterface;

public function findEnums(InterfacesInterface $interfaces): void
{
    foreach ($interfaces->getInterfaces() as $interface) {
        dump($interface->getFileName());
    }
}
```

> **Warning**
> Tokenizer will ignore all the files that have `inlcude` or `require` statements. This is because it's not safe to
> require such reflections. Please don't use them in your code.

### Customizing search directories

Tokenizer, by default, conducts its search within the app directory. However, you might often want it to consider other
directories as well. Fortunately, customizing this is straightforward using the `Spiral\Bootloader\TokenizerBootloader`.

:::: tabs

::: tab Bootloader

Here's how you can specify additional directories:

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

:::

::: tab Config

Alternatively, you can add directories directly in the `app/config/tokenizer.php` configuration file:

```php app/config/tokenizer.php
return [
    'directories' => [
        directory('app'),
        directory('vendor') . 'spiral/validator/src',
    ],
];
```

:::

::::

### Excluding specific directories

You might also want to exclude specific directories from Tokenizer's search. Here's how:

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
> Remember, expanding the directories for class search can slow down the process. It's recommended to only add
> directories that are essential for your needs.

### Scoped Class Locator

When dealing with vast directories, Tokenizer can slow down a bit. However, you can amp up its speed by using the scoped
class locator. This tool lets you divide and conquer by setting up specific search zones, which we call `scopes`.

If you need to search for classes in a large number of directories, the tokenizer component may suffer from poor
performance. In this case, you can use the scoped class locator to improve performance.

With the scoped class locator, you can define directories to be searched within named scopes. This allows you to
selectively search only the directories that are relevant to your current task.

#### Setting up Scopes

To use the scoped class locator, you will need to define your scopes:

:::: tabs

::: tab Config

Scopes can be defined in the `app\config\tokenizer.php` config file.

Here is an example of how to define a scope named `scopeName` that searches the `app/Directory` directory:

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
> The `exclude` parameter is there for a reason. If there are parts of the directory you know you won't need, just tell
> Tokenizer to skip them. It'll make things even faster!

:::

::: tab Bootloader

Scopes can be defined using the `Spiral\Bootloader\TokenizerBootloader` bootloader:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addScopedDirectory('scopeName', $directories->get('app') . 'Directory');
    }
}
```

:::

::::

Once you've got your scopes ready, you can then instruct Tokenizer to only search within a chosen scope. It's like
telling it which department to go to!

:::: tabs

::: tab Classes

The `Spiral\Tokenizer\ScopedClassesInterface` lets you do this with its `getScopedClasses` method. Simply hand it the
name of the scope, and it'll return all the classes it finds in that zone.

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

:::

::: tab Enums
Just like with classes, you can set up specific scopes for searching enums. Once you have your scopes configured in
the `app\config\tokenizer.php` file, you can utilize the `Spiral\Tokenizer\ScopedEnumsInterface`.

Here's an example of how you can search for enums in a specific scope:

```php
use Spiral\Tokenizer\ScopedEnumsInterface;

final class EnumSearcher
{
    public function __construct(
        private readonly ScopedEnumsInterface $locator
    ) {
    }

    public function pinpointEnums(): array
    {
        $foundEnums = [];

        foreach ($this->locator->getScopedEnums('scopeName') as $enum) {
            $foundEnums[] = $enum;
            // or any other operations you want...
        }

        return $foundEnums;
    }
}
```

:::

::: tab Interfaces
Similar to the above, once your scopes are good to go in the config file,
the `Spiral\Tokenizer\ScopedInterfacesInterface` will help you search for interfaces within those specified zones.

Here's how to scout for interfaces in a designated scope:

```php
use Spiral\Tokenizer\ScopedInterfacesInterface;

final class InterfaceSearcher
{
    public function __construct(
        private readonly ScopedInterfacesInterface $locator
    ) {
    }

    public function findInterfaces(): array
    {
        $identifiedInterfaces = [];

        foreach ($this->locator->getScopedInterfaces('scopeName') as $interface) {
            $identifiedInterfaces[] = $interface;
            // or add your desired actions...
        }

        return $identifiedInterfaces;
    }
}
```

:::

::::

## Efficient scanning with class listeners

For big codebases, regular scans using Tokenizer's locators can slow things down, especially when you're routinely
searching for classes, enums, or interfaces. Class listeners provide a smarter approach, allowing you to listen in on
and react to class discoveries without repeatedly scanning directories.

### Why use class listeners?

Think about having to search through a really big library every time you want a book. It's a lot of work. Now, think
about someone telling you whenever a new book arrives in the library. That's similar to how class listeners work.
Instead of searching the library every time, you get a heads-up when a new book (class) arrives. This is especially
useful during application bootstrapping, where the initial scanning takes place, after which listeners are kept in the
loop.

### Configuring listeners

By default, listeners focus on classes, but you can easily configure them to cast their net wider to encompass enums and
interfaces too. To do this, you will need to add the following configuration to the `app\config\tokenizer.php` file:

```php app/config/tokenizer.php
return [
    'load' => [
        'classes' => true, // Search for classes
        'enums' => true, // Search for enums
        'interfaces' => true, // Search for interfaces
    ],
];
```

> **Note**
> Remember, you don't have to enable all three. Tailor it to suit your project's needs. The more you enable, the slower
> the process will be.

### Usage

To use this feature, you will need to include `Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader`
bootloader in your project at the top of bootloader's list:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

### Crafting a listener

The next step is to create a listener class. This class should implement
the `Spiral\Tokenizer\TokenizationListenerInterface`, which mandates two methods:

- `listen(\ReflectionClass $class)`: This method is called every time a class is discovered. You can add logic to
  process or store information from this class.
- `finalize()`: Think of this as the closing act. Once all classes are scanned and processed, this method is invoked.
  It's a perfect spot for wrapping things up or finalizing operations based on the discovered classes.

**Here's an example of a listener:**

```php
use Spiral\Attributes\ReaderInterface;

final class RouteAttributeListener implements TokenizationListenerInterface
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

This listener, for instance, listens for classes with specific routing attributes and adds them to a router when the
scan is complete.

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

## Console commands

### Info

Want to know how the tokenizer is set up? Use the `tokenizer:info` command.

**Just run the following command:**

```terminal
php app.php tokenizer:info
```

#### What You'll See

1. **Included directories:** Shows which folders the tokenizer looks in.
2. **Excluded directories:** Folders the tokenizer ignores.
3. **Loaders:** Tells you what kind of PHP things (like classes or interfaces) the tokenizer is looking for. It'll also
   show how to turn these on or off.
4. **Tokenizer cache:** Shows if there's a shortcut (cache) used to speed things up. You can turn this on or off too.

#### Example output

```terminal
Included directories:
+------------------------------------+-------+
| Directory                          | Scope |
+------------------------------------+-------+
| /vendor/intruforce/grpc-shared/src |       |
| app/                               |       |
+------------------------------------+-------+

Excluded directories:
+-----------+-------+
| Directory | Scope |
+-----------+-------+

Loaders:
+------------+------------------------------------------------------------------------------+
| Loader     | Status                                                                       |
+------------+------------------------------------------------------------------------------+
| Classes    | enabled                                                                      |
| Enums      | disabled. To enable, add "TOKENIZER_LOAD_ENUMS=true" to your .env file.      |
| Interfaces | disabled. To enable, add "TOKENIZER_LOAD_INTERFACES=true" to your .env file. |
+------------+------------------------------------------------------------------------------+

Tokenizer cache: disabled
To enable cache, add "TOKENIZER_CACHE_TARGETS=true" to your .env file.
```
