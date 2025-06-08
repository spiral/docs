# Продвинутые возможности — Атрибуты (Attributes)

Атрибуты (Attributes) в PHP — это способ добавления метаданных к классам, свойствам, функциям, константам и параметрам. Эти метаданные могут использоваться для предоставления дополнительной информации об элементе или для изменения поведения элемента в определенных ситуациях.

Компонент `spiral/attributes` предоставляет простой и последовательный способ работы с атрибутами в PHP.

**Преимущества использования атрибутов в PHP приложении:**

- Они предоставляют способ добавления метаданных к элементам отдельно от логики элемента.
- Они могут использоваться для изменения поведения элементов в определенных ситуациях, например, для предоставления дополнительной валидации свойства или изменения поведения функции на основе её атрибутов.
- Они могут использоваться для предоставления дополнительной информации об элементах, например, для предоставления документации для класса или свойства.
- Используя атрибуты в сочетании с перехватчиками (interceptors), разработчики могут реализовать практики аспектно-ориентированного программирования в своей кодовой базе, что приводит к таким преимуществам, как более компактный, быстрый и легко тестируемый код.

**Компонент также служит двум очень важным целям:**

- Возможность объединения различных типов метаданных в одном месте. Вам не нужно заботиться о том, использует ли разработчик [Doctrine Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html) или [PHP Attributes](https://wiki.php.net/rfc/attributes_v2), добавленные в PHP 8.
- Возможность чтения атрибутов в любой версии языка. Это означает, что вы можете использовать PHP 8 Attributes прямо сейчас, даже если вы используете PHP 7.2.

Компонент предоставляет мост для чтения метаданных, позволяющий использовать современные [PHP атрибуты](https://wiki.php.net/rfc/attributes_v2) и [Doctrine аннотации](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html) в одном проекте.

> **Примечание:**
> В этой документации термин "метаданные" используется для обозначения как "атрибутов", так и "аннотаций".

## Установка

Для установки компонента:

```terminal 
composer require spiral/attributes
```

### Интеграция с фреймворком

Чтобы включить компонент, вам просто нужно добавить класс `Spiral\Bootloader\Attributes\AttributesBootloader` в список загрузчиков (bootloaders):

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Bootloader\Attributes\AttributesBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках (bootloaders) в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Attributes\AttributesBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках (bootloaders) в разделе [Фреймворк — Загрузчики](../framework/bootloaders.md).
:::

::::

## Использование

Компонент предоставляет набор интерфейсов и классов для работы с атрибутами.
`Spiral\Attributes\ReaderInterface` предоставляет набор методов для чтения атрибутов из классов, свойств, функций, констант и параметров.

```php
interface ReaderInterface
{
    public function getClassMetadata(\ReflectionClass $class, string $name = null): iterable;
    public function getPropertyMetadata(\ReflectionProperty $property, string $name = null): iterable;
    public function getFunctionMetadata(\ReflectionFunctionAbstract $function, string $name = null): iterable;
    public function getConstantMetadata(\ReflectionClassConstant $constant, string $name = null): iterable;
    public function getParameterMetadata(\ReflectionParameter $parameter, string $name = null): iterable;
    
    public function firstClassMetadata(\ReflectionClass $class, string $name): ?object;
    public function firstPropertyMetadata(\ReflectionProperty $property, string $name): ?object;
    public function firstFunctionMetadata(\ReflectionFunctionAbstract $function, string $name): ?object;
    public function firstConstantMetadata(\ReflectionClassConstant $constant, string $name): ?object;
    public function firstParameterMetadata(\ReflectionParameter $parameter, string $name): ?object;
}
```

### Метаданные класса

Для чтения метаданных класса используйте метод `$reader->getClassMetadata()`. Он принимает на вход `ReflectionClass` требуемого класса и возвращает список доступных объектов метаданных.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection); 
// возвращает iterable<object>
```

Второй необязательный аргумент `$name` метода позволяет указать, какие конкретные объекты метаданных вы хотите получить.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection, Entity::class); 
// возвращает iterable<Entity>
```

Чтобы получить один объект метаданных, вы можете использовать метод `$reader->firstClassMetadata()`.

```php
$reflection = new ReflectionClass(User::class);

$attribute = $reader->firstClassMetadata($reflection, Entity::class); 
// возвращает Entity|null
```

Начиная с **v2.10.0**, поддерживается чтение атрибутов из трейтов, которые используются в классе.

:::: tabs

::: tab Атрибуты

```php
#[Cycle\Entity]
class Entity {
    use TsTrait;
}

#[Behavior\CreatedAt]
#[Behavior\UpdatedAt]
trait TsTrait
{
    #[Cycle\Column(type: 'datetime')]
    private DateTimeImmutable $createdAt;

    #[Cycle\Column(type: 'datetime', nullable: true)]
    private ?DateTimeImmutable $updatedAt = null;
}
```

:::

::: tab Аннотации

```php
/**
 * @Cycle\Entity
 */
class Entity {
    use TsTrait;
}

/**
 * @Behavior\CreatedAt
 * @Behavior\UpdatedAt
 */
trait TsTrait
{
    #[Cycle\Column(type: 'datetime')]
    private DateTimeImmutable $createdAt;

    #[Cycle\Column(type: 'datetime', nullable: true)]
    private ?DateTimeImmutable $updatedAt = null;
}
```

:::

::::

### Метаданные свойств

Для чтения метаданных свойства используйте метод `$reader->getPropertyMetadata()`. Он принимает на вход `ReflectionProperty` требуемого свойства и возвращает список доступных объектов метаданных.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection); 
// возвращает iterable<object>
```

Второй необязательный аргумент `$name` метода позволяет указать, какие конкретные объекты метаданных вы хотите получить.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection, Column::class); 
// возвращает iterable<Column>
```

Чтобы получить один объект метаданных, вы можете использовать метод `$reader->firstPropertyMetadata()`.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$column = $reader->firstPropertyMetadata($reflection, Column::class); 
// возвращает Column|null
```

### Метаданные функций

Для чтения метаданных функции используйте метод `$reader->getFunctionMetadata()`. Он принимает аргумент типа `ReflectionFunction` или `ReflectionMethod` требуемой функции и возвращает список доступных объектов метаданных.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getFunctionMetadata($reflection); 
// возвращает iterable<object>
```

Второй необязательный аргумент `$name` метода позволяет указать, какие конкретные объекты метаданных вы хотите получить.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getPropertyMetadata($reflection, DTOGetter::class); 
// возвращает iterable<DTOGetter>
```

Чтобы получить один объект метаданных, вы можете использовать метод `$reader->firstFunctionMetadata()`.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$getter = $reader->firstFunctionMetadata($reflection, DTOGetter::class); 
// возвращает DTOGetter|null
```

### Метаданные констант

Для чтения метаданных константы класса используйте метод `$reader->getConstantMetadata()`. Он принимает аргумент типа `ReflectionClassConstant` требуемой константы и возвращает список доступных объектов метаданных.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection); 
// возвращает iterable<object>
```

Второй необязательный аргумент `$name` метода позволяет указать, какие конкретные объекты метаданных вы хотите получить.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$attributes = $reader->getConstantMetadata($reflection, Deprecated::class); 
// возвращает iterable<Deprecated>
```

Чтобы получить один объект метаданных, вы можете использовать метод `$reader->firstConstantMetadata()`.

```php
$reflection = new ReflectionClassConstant(Example::class, 'CONSTANT_NAME');

$getter = $reader->firstConstantMetadata($reflection, Deprecated::class); 
// возвращает Deprecated|null
```

### Метаданные параметров

Для чтения метаданных параметров функции/метода используйте метод `$reader->getParameterMetadata()`. Он принимает аргумент типа `ReflectionParameter` требуемой функции и возвращает список доступных объектов метаданных.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection); 
// возвращает iterable<object>
```

Второй необязательный аргумент `$name` метода позволяет указать, какие конкретные объекты метаданных вы хотите получить.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$attributes = $reader->getParameterMetadata($reflection, PreCondition::class); 
// возвращает iterable<PreCondition>
```

Чтобы получить один объект метаданных, вы можете использовать метод `$reader->firstParameterMetadata()`.

```php
$reflection = new ReflectionParameter('send_email', 'email');

$getter = $reader->firstConstantMetadata($reflection, PreCondition::class); 
// возвращает PreCondition|null
```

## Создание аннотаций

> **Примечание**
> Для подробностей по использованию doctrine аннотаций, пожалуйста, смотрите
>
[документацию doctrine](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html#create-an-annotation-class).

Вы должны использовать "гибридный" синтаксис для создания классов метаданных, которые будут работать в любой версии PHP.

```php
/**
 * @Annotation
 * @Target({ "CLASS" })
 */
#[\Attribute(\Attribute::TARGET_CLASS)]
class MyEntityMetadata
{
    public string $table;
}
```

В этом случае вы можете использовать этот класс метаданных в любой версии PHP.

:::: tabs

::: tab Атрибуты

```php
#[MyEntityMetadata(table: 'users')] 
class User {}
```

:::

:::: tab Аннотации

```php
/**
 * @MyEntityMetadata(table="users")
 */
class User {}
```

:::

::::

### Создание экземпляров

Пакет поддерживает различные способы создания экземпляров атрибутов, но по умолчанию использует логику Doctrine для совместимости.

Допустим, вы используете ваш класс метаданных следующим образом, передавая одно поле "`property`" со строковым значением "`value`".

:::: tabs

::: tab Атрибуты

```php
#[CustomMetadataClass(property: "value")]
class AnnotatedClass
{
}
```

:::

::: tab Аннотации

```php
/** @CustomMetadataClass(property="value") */
class AnnotatedClass
{
}
```

:::

::::

В этом случае сам класс аннотации может выглядеть следующим образом:

#### Базовый инстанциатор Doctrine

В этом случае при объявлении класса метаданных будут заполнены свойства атрибута/аннотации.

> **Примечание**
> См. также [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass
{
    public $property;
}
```

#### Конструкторный инстанциатор Doctrine

В случае объявления конструктора все данные при использовании класса метаданных будут переданы в этот конструктор в виде массива.

> **Примечание**
> См. также [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass
{
    public function __construct(array $properties)
    {
        // $properties = [ "property" => "value" ]
    }
}
```

#### Именованные аргументы (маркер интерфейса)

Если вы хотите использовать именованные параметры конструктора (см. также [PHP Руководство — Именованные аргументы](https://www.php.net/manual/en/functions.arguments.php#functions.named-arguments)), то вам нужно добавить интерфейс `Spiral\Attributes\NamedArgumentConstructorAttribute` к классу метаданных, который отметит требуемый класс метаданных как тот, который принимает именованные аргументы.

```php
/** @Annotation */
#[\Attribute]
class CustomMetadataClass implements \Spiral\Attributes\NamedArgumentConstructorAttribute
{
    public function __construct($property)
    {
        // $property = "value"
    }
}
```

#### Именованные аргументы (маркер метаданных)

Обратите внимание, что использование предыдущего метода потребует от вас наличия пакета `spiral/attributes`, и вы не сможете использовать эти классы в других проектах, где этот пакет отсутствует.

Чтобы решить эту проблему, вы можете использовать класс метаданных, который будет означать то же самое (класс метаданных использует именованные аргументы), но не реализует интерфейс напрямую, и поэтому не требует пакета `spiral/attributes` в проекте.

```php
/**
 * @Annotation
 * @Spiral\Attributes\NamedArgumentConstructor
 */
#[\Attribute]
#[\Spiral\Attributes\NamedArgumentConstructor]
class CustomMetadataClass
{
    public function __construct($property)
    {
        // $property = "value"
    }
}
```

## Драйверы

`Spiral\Attributes\Factory` инкапсулирует несколько реализаций и по умолчанию возвращает реализацию [селективного ридера](#selective-reader), которая подходит для большинства случаев. Однако вы можете потребовать конкретную реализацию, если она доступна на вашей платформе и/или в вашем приложении.

```php
use Spiral\Attributes\Factory;

$reader = (new Factory())->create();
```

### Ридер аннотаций

> **Примечание**
> Обратите внимание, что для того, чтобы этот ридер был доступен в приложении, вам нужно подключить компонент "doctrine/annotations".

Эта реализация ридера позволяет читать doctrine аннотации.

```php
/** @ExampleAnnotation */
class Example {}

$reader = new \Spiral\Attributes\AnnotationReader();

$annotations = $reader->getClassMetadata(new ReflectionClass(Example::class));
// возвращает iterable<ExampleAnnotation>
```

### Ридер атрибутов

Эта реализация ридера позволяет читать нативные PHP атрибуты в любой версии PHP.

```php
#[ExampleAttribute]
class Example {}

$reader = new \Spiral\Attributes\AttributeReader();

$attributes = $reader->getClassMetadata(new ReflectionClass(Example::class));
// возвращает iterable<ExampleAttribute>
```

### Селективный ридер

Реализация автоматически выбирает правильный ридер на основе синтаксиса, используемого в приложении. Это поведение необходимо, если вы используете оба синтаксиса одновременно в одном проекте. Например, в случае уже работающего проекта, который рефакторится, и синтаксис аннотаций переводится в современный синтаксис атрибутов.

:::: tabs

::: tab Атрибуты

```php
#[ExampleAttribute]
class ClassWithAttributes {}
```

:::

::: tab Аннотации

```php
/** @ExampleAnnotation */
class ClassWithAnnotations {}
```

:::

::::

```php
$reader = new \Spiral\Attributes\Composite\SelectiveReader([
    new \Spiral\Attributes\AnnotationReader(),
    new \Spiral\Attributes\AttributeReader(),
]);

$annotations = $reader->getClassMetadata(new ReflectionClass(ClassWithAnnotations::class));
// возвращает iterable<ExampleAnnotation>

$attributes = $reader->getClassMetadata(new ReflectionClass(ClassWithAttributes::class));
// возвращает iterable<ExampleAttribute>
```

> **Примечание**
> При использовании как аннотаций, так и атрибутов в одном месте, поведение этого ридера не детерминировано.

### Объединяющий ридер

Реализация ридера объединяет несколько синтаксисов в один. Это поведение необходимо, если вы работаете с несколькими библиотеками одновременно, которые поддерживают либо только старый, либо только новый синтаксис.

```php
/** @DoctrineAnnotation */
#[NativeAttribute]
class ExampleClass {}

$reader = new \Spiral\Attributes\Composite\MergeReader([
    new \Spiral\Attributes\AnnotationReader(),
    new \Spiral\Attributes\AttributeReader(),
]);

$metadata = $reader->getClassMetadata(new ReflectionClass(ExampleClass::class));
// возвращает iterable { DoctrineAnnotation, NativeAttribute }
```

## Кэширование

Некоторые реализации могут до некоторой степени замедлять работу, поскольку они читают метаданные с нуля. Это особенно актуально, когда таких данных много.

Для оптимизации и ускорения работы ридеров рекомендуется использовать кэш. Пакет атрибутов поддерживает спецификации [PSR-6](https://www.php-fig.org/psr/psr-6/) и [PSR-16](https://www.php-fig.org/psr/psr-16/). Для их создания нужно использовать соответствующие классы.

```php
use Spiral\Attributes\Psr6CachedReader;
use Spiral\Attributes\Psr16CachedReader;
use Spiral\Attributes\AttributeReader;

$psr6reader = new Psr6CachedReader(
    new AttributeReader(),
    new SomePsr6CacheImplementation() // Любая PSR-6 реализация кэша
);

$psr16reader = new Psr16CachedReader(
    new AttributeReader(),
    new SomePsr6CacheImplementation() // Любая PSR-16 реализация кэша
);
```

Вы также можете передать экземпляр реализации кэша в класс factory.

```php
use Spiral\Attributes\Factory;

$reader = (new Factory)
    ->withCache($cacheDriver)
    ->create();

// Где $cacheDriver — это реализация драйвера кэша PSR-6 или PSR-16
```
