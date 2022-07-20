# Code Generation

You can use the component `spiral/reactor` to generate PHP classes code using fluent declarative wrappers. This component
is the foundation of `spiral/scaffolder` exception but can be used separately from the framework:

```bash
composer require spiral/reactor
```

> **Note**
> Please note that the spiral/framework >= 2.7 already includes this component.

## Class Declaration
To declare class use `Spiral\Reactor\ClassDeclaration`.

```php
use Spiral\Reactor\ClassDeclaration;

$class = new ClassDeclaration('MyClass');

dump($class->render());
```

The output:

```php
class MyClass
{
}
```

You can get access to most of the declaration directly from the class.

### Property
To render class property:

```php
$class = new ClassDeclaration('MyClass');

$class->property('my_property')
    ->setProtected()
    ->setDefaultValue('default')
    ->setComment(['My property.', '@var string']);

dump($class->render());
```

The output:

```php
class MyClass
{
    /**
     * My property.
     * @var string
     */
    protected $my_property = 'default';
}
```

### Constant
To render constant:

```php
$class = new ClassDeclaration('MyClass');

$class->constant('MY_CONSTANT')
    ->setPublic()
    ->setValue('default')
    ->setComment(['My constant']);

dump($class->render());
```

The output:

```php
class MyClass
{
    /**
     * My constant
     */
    public const MY_CONSTANT = 'default';
}
```

### Traits
To add trait declaration:

```php
$class = new ClassDeclaration('MyClass');

$class->addTrait(PrototypeTrait::class);

dump($class->render());
```

The output:

```php
class MyClass
{
    use Spiral\Prototype\Traits\PrototypeTrait;
}
```

### Interface and Extends
To implement given interface or extend base class:

```php
use Spiral\Reactor\ClassDeclaration;
use Cycle\ORM\Select;

$class = new ClassDeclaration('MyClass');

$class->addInterface(\Countable::class)->setExtends(Select\Repository::class);

dump($class->render());
```

The output:

```php
class MyClass extends Cycle\ORM\Select\Repository implements Countable
{
}
```

### Methods
To generate class method:

```php
$class = new ClassDeclaration('MyClass');

$method = $class->method('ping')
    ->setPublic()
    ->setComment('My method')
    ->setReturn('void');

$method->parameter('a')->setType('string')->setDefaultValue('test');
$method->setSource('echo $a');

dump($class->render());
```

The output:

```php
class MyClass
{
    /**
     * My method
     */
    public function ping(string $a = 'test'): void
    {
        echo $a
    }
}
```

## Namespace
To place generated class into a namespace:

```php
use Spiral\Reactor\ClassDeclaration;
use Spiral\Reactor\NamespaceDeclaration;

$class = new ClassDeclaration('MyClass');

$ns = new NamespaceDeclaration('MyNamespace');
$ns->addElement($class);

dump($ns->render());
```

The output:

```php
namespace MyNamespace {
    class MyClass
    {
    }
}
```

## File
Alternatively you can render whole PHP file (with forced namespace):

```php
use Spiral\Reactor\ClassDeclaration;
use Spiral\Reactor\FileDeclaration;
use Cycle\ORM\Select;

$class = new ClassDeclaration('MyClass');

$file = new FileDeclaration('MyNamespace');

$file->setComment('This is my file');
$file->setDirectives('strict_types=1');

$file->addUse(Select\Repository::class, 'Repo');

$file->addElement($class);

dump($file->render());
```

The output:

```php
<?php

/**
 * This is my file
 */
declare(strict_types=1);

namespace MyNamespace;

use Cycle\ORM\Select\Repository as Repo;

class MyClass
{
}
```
