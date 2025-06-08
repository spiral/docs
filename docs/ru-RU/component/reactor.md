# Компонент — Генерация кода

Вы можете использовать компонент `spiral/reactor` для генерации PHP кода классов, используя текучие декларативные обёртки. Этот компонент основан на `nette/php-generator`.

## Установка

Для установки компонента:

```terminal
composer require spiral/reactor
```

> **Примечание**
> Обратите внимание, что spiral/framework >= 2.7 уже включает этот компонент.

## Объявление класса

Для объявления класса используйте `Spiral\Reactor\ClassDeclaration`.

```php
use Spiral\Reactor\ClassDeclaration;

$class = new ClassDeclaration('MyClass');

dump($class->render()); // или dump((string) $class);
```

Результат:

```php
class MyClass
{
}
```

Вы можете получить доступ к большинству объявлений напрямую из класса.

### Свойство

Для отображения свойства класса:

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

Результат:

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

### Константа

Для отображения константы:

```php
$class = new ClassDeclaration('MyClass');

$class->addConstant('MY_CONSTANT', 'default')
    ->setPublic()
    ->setFinal()
    ->addAttribute('SomeAttribute')
    ->setComment(['My constant']);

dump((string) $class);
```

Результат:

```php
class MyClass
{
    /** My constant */
    #[SomeAttribute]
    final public const MY_CONSTANT = 'default';
}
```

### Трейты

Для добавления объявления трейта:

```php
$class = new ClassDeclaration('MyClass');

$class->addTrait(PrototypeTrait::class);

dump((string) $class);
```

Результат:

```php
class MyClass
{
    use Spiral\Prototype\Traits\PrototypeTrait;
}
```

### Интерфейс и наследование

Для реализации заданного интерфейса или расширения базового класса:

```php
use Spiral\Reactor\ClassDeclaration;
use Cycle\ORM\Select\Repository;

$class = new ClassDeclaration('MyClass');

$class
    ->addImplement(\Countable::class)
    ->setExtends(Repository::class);

dump((string) $class);
```

Результат:

```php
class MyClass extends Cycle\ORM\Select\Repository implements Countable
{
}
```

### Методы

Для генерации метода класса:

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

Результат:

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

## Пространство имён

Для создания класса внутри определённого пространства имён:

```php
use Spiral\Reactor\Partial\PhpNamespace;

$namespace = new PhpNamespace('App\\Some');
$namespace->addClass('MyClass')

dump((string) $namespace);
 ```

Результат:

```php
namespace App\Some;

class MyClass
{
}
```

## Объявление интерфейса

Для объявления интерфейса используйте `Spiral\Reactor\InterfaceDeclaration`.

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

Результат:

```php
/**
 * My interface
 */
interface MyInterface extends Countable
{
    public function someMethod(): int;
}
```

## Объявление перечисления

Для объявления перечисления используйте `Spiral\Reactor\EnumDeclaration`.

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

Результат:

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

## Объявление функции

Для объявления глобальной функции используйте `Spiral\Reactor\FunctionDeclaration`.

```php
$function = new FunctionDeclaration('myFunction');
$function
    ->addBody('return \'Hello world\';')
    ->setReturnType('string')
    ->addAttribute('SomeAttribute')
    ->addComment('Some function');

dump((string) $function);
```

Результат:

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

## Объявление трейта

Для объявления трейта используйте `Spiral\Reactor\TraitDeclaration`.

```php
$trait = new TraitDeclaration('MyTrait');
$trait
    ->setComment('Some trait')
    ->addMethod('myMethod')
        ->setPublic()
        ->setReturnType('void');

dump((string) $trait);
```

Результат:

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

## Файл

Вы можете собрать классы, интерфейсы, трейты, глобальные функции и перечисления в файле:

```php
$namespace = new PhpNamespace('MyNamespace');
$namespace
    ->addUse(\Countable::class)
    ->addUse(Repository::class, 'Repo') // с псевдонимом
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

Результат:

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