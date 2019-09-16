# Static Analysis
A lot of Spiral components are based on automatic code discovery and analysis. The most important functionality or locating class declarations
is provided by `Spiral\Tokenizer\ClassesInterface`.

> Tokenizer component is pre-installed with all framework bundles.

## ClassLocator
Use `Spiral\Tokenizer\ClassesInterface` to find available classes by their name, interface or trait:

```php
public function findClasses(ClassesInterface $classes)
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

By default, the component will be looking for classes available in the `app` directory only. You can add any other directory
using `Spiral\Bootloader\TokenizerBootloader`:

```php
public function boot(TokenizerBootloader $tokenizer)
{
    $tokenizer->addDirectory(directory('vendor') . 'name/extension/src');
}
```

> Attention, class lookup is not a fast process, only add necessary directories.
