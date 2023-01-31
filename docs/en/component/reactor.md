# Component — Code Generation

You can use the component `spiral/reactor` to generate PHP classes code using fluent declarative wrappers. This component
is based on `nette/php-generator`.

## Installation

To install the component:

```terminal
composer require spiral/reactor
```

> **Note**
> Please note that the spiral/framework >= 2.7 already includes this component.

## Class Declaration

To declare a class, use `Spiral\Reactor\ClassDeclaration`.

```php
use Spiral\Reactor\ClassDeclaration;

$class = new ClassDeclaration('MyClass');

dump($class->render()); // or dump((string) $class);
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

$class->addProperty('property', 'default')
    ->setProtected()
    ->setReadOnly()
    ->setType('string')
    ->setComment(['My property.', '@var string'])
    ->addAttribute('SomeAttribute');

dump((string) $class);
```

The output:

```php
class MyClass
{
    /**
     * My property.
     * @var string
     */
    #[SomeAttribute]
    protected readonly string $property = 'default';
}
```

### Constant

To render constant:

```php
$class = new ClassDeclaration('MyClass');

$class->addConstant('MY_CONSTANT', 'default')
    ->setPublic()
    ->setFinal()
    ->addAttribute('SomeAttribute')
    ->setComment(['My constant']);

dump((string) $class);
```

The output:

```php
class MyClass
{
    /** My constant */
    #[SomeAttribute]
    final public const MY_CONSTANT = 'default';
}
```

### Traits

To add trait declaration:

```php
$class = new ClassDeclaration('MyClass');

$class->addTrait(PrototypeTrait::class);

dump((string) $class);
```

The output:

```php
class MyClass
{
    use Spiral\Prototype\Traits\PrototypeTrait;
}
```

### Interface and Extends

To implement a given interface or extend a base class:

```php
use Spiral\Reactor\ClassDeclaration;
use Cycle\ORM\Select\Repository;

$class = new ClassDeclaration('MyClass');

$class
    ->addImplement(\Countable::class)
    ->setExtends(Repository::class);

dump((string) $class);
```

The output:

```php
class MyClass extends Cycle\ORM\Select\Repository implements Countable
{
}
```

### Methods

To generate a class method:

```php
$class = new ClassDeclaration('MyClass');

$class->addMethod('ping')
    ->setPublic()
    ->setComment('My method')
    ->setReturnType('string')
    ->setReturnNullable()
    ->setFinal()
    ->setBody('return $a;')
    ->addAttribute('SomeAttribute')
        ->addParameter('a', null)
        ->setType('string')
        ->setNullable(true);

dump((string) $class);
```

The output:

```php
class MyClass
{
    /**
     * My method
     */
    #[SomeAttribute]
    final public function ping(?string $a = null): ?string
    {
        return $a;
    }
}
```

## Namespace

To create a class inside a specific namespace:

```php
use Spiral\Reactor\Partial\PhpNamespace;

$namespace = new PhpNamespace('App\\Some');
$namespace->addClass('MyClass')

dump((string) $namespace);
 ```

The output:

```php
namespace App\Some;

class MyClass
{
}
```

## Interface Declaration

To declare an interface, use `Spiral\Reactor\InterfaceDeclaration`.

```php
$interface = new InterfaceDeclaration('MyInterface');
$interface
    ->addExtend(\Countable::class)
    ->addComment('My interface')
    ->addMethod('someMethod')
        ->setPublic()
        ->setReturnType('int');

dump((string) $interface);
```

The output:

```php
/**
 * My interface
 */
interface MyInterface extends Countable
{
    public function someMethod(): int;
}
```

## Enum Declaration

To declare enum, use `Spiral\Reactor\EnumDeclaration`.

```php
$enum = new EnumDeclaration('MyEnum');

$enum->addCase('First', 'first');
$enum->addCase('Second', 'second');

$enum
    ->setType('string')
    ->addConstant('FOO', 'bar')
    ->addComment('Description of enum')
    ->addAttribute('SomeAttribute');
$enum
    ->addMethod('getCase')
    ->setReturnType('string')
    ->addBody('return self::First->value;');

dump((string) $enum);
```

The output:

```php
/**
 * Description of enum
 */
#[SomeAttribute]
enum MyEnum: string
{
    public const FOO = 'bar';

    case First = 'first';
    case Second = 'second';

    public function getCase(): string
    {
        return self::First->value;
    }
}
```

## Function Declaration

To declare a global function, use `Spiral\Reactor\FunctionDeclaration`.

```php
$function = new FunctionDeclaration('myFunction');
$function
    ->addBody('return \'Hello world\';')
    ->setReturnType('string')
    ->addAttribute('SomeAttribute')
    ->addComment('Some function');

dump((string) $function);
```

The output:

```php
/**
 * Some function
 */
#[SomeAttribute]
function myFunction(): string
{
    return 'Hello world';
}

```

## Trait Declaration

To declare a trait, use `Spiral\Reactor\TraitDeclaration`.

```php
$trait = new TraitDeclaration('MyTrait');
$trait
    ->setComment('Some trait')
    ->addMethod('myMethod')
        ->setPublic()
        ->setReturnType('void');

dump((string) $trait);
```

The output:

```php
/**
 * Some trait
 */
trait MyTrait
{
    public function myMethod(): void
    {
    }
}
```

## File

You can collect classes, interfaces, traits, global functions, and enums in a file:

```php
$namespace = new PhpNamespace('MyNamespace');
$namespace
    ->addUse(\Countable::class)
    ->addUse(Repository::class, 'Repo') // with alias
    ->addUseFunction('count');

$class = $namespace->addClass('MyClass');
$class
    ->addImplement(\Countable::class)
    ->addMethod('count')
        ->setReturnType('int')
        ->addBody('return 1;');

$file = new FileDeclaration();
$file
    ->setStrictTypes()
    ->addNamespace($namespace);

dump((string) $file);
```

The output:

```php
namespace MyNamespace;

use Countable;
use Cycle\ORM\Select\Repository as Repo;
use function count;

class MyClass implements Countable
{
    public function count(): int
    {
        return 1;
    }
}
```
