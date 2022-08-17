# Static Analysis

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

## PHP-Parser

The [nikic/PHP-Parser](https://github.com/nikic/PHP-Parser) is available in Web and GRPC bundle by default. Use this
dependency for a deeper analysis of AST-tree.
