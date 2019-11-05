# Static Analysis
A lot of Spiral components are based on automatic code discovery and analysis. The most used functionality of locating class declarations is provided by `Spiral\Tokenizer\ClassesInterface`. 

> Tokenizer component is pre-installed with all framework bundles.

## Class Locator
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

## PHP-Parser
The [nikic/PHP-Parser](https://github.com/nikic/PHP-Parser) is available in Web and GRPC bundle by default. Use this dependency for deeper
analysis of AST-tree.

## ORM Introspection
To get list of all available entity roles:

```php
use Cycle\ORM\ORMInterface;

// ...

public function index(ORMInterface $orm)
{
    dump($orm->getSchema()->getRoles());
}
```

To get all classes for all ORM entities:

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\Schema;

// ...

public function index(ORMInterface $orm)
{
    foreach($orm->getSchema()->getRoles() as $role) {
        dump($orm->getSchema()->define($role, Schema::ENTITY));
    }
}
```

