# Attributes

Component `spiral/attributes` provides a metadata reader bridge allowing both modern 
[PHP attributes](https://wiki.php.net/rfc/attributes_v2) and 
[Doctrine annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html) 
to be used in the same project.

This documentation uses the term "metadata" to refer to both "attributes" and "annotations".

## Installation

To install the component:

```bash
$ composer require spiral/attributes
```

> Please note that the spiral/framework >= 2.7 already includes this component.

Make sure to add `Spiral\Bootloader\AttributesBootloader` to your App class:

```php
protected const LOAD = [
    // ...
    Spiral\Bootloader\AttributesBootloader::class
];
```

After that, the following interfaces will become available to you for access through the container:
- `Spiral\Attributes\ManagerInterface`
- `Spiral\Attributes\ReaderInterface`

## Usage

To get a metadata reader, use the `Spiral\Attributes\Manager`. It allows you to correctly initialize available 
readers and provides a method `$manager->get()` to get them.

```php
$manager = new Spiral\Attributes\Manager();

$reader = $manager->get();
```

After receiving the reader, you can use it to get the required metadata.

### Reading Metadata

The reader provides a set of following methods:

```php
interface ReaderInterface
{
    public function getClassMetadata(\ReflectionClass $class, string $name = null): iterable;
    public function getPropertyMetadata(\ReflectionProperty $property, string $name = null): iterable;
    public function getFunctionMetadata(\ReflectionFunctionAbstract $function, string $name = null): iterable;
    
    public function firstClassMetadata(\ReflectionClass $class, string $name): ?object;
    public function firstPropertyMetadata(\ReflectionProperty $property, string $name): ?object;
    public function firstFunctionMetadata(\ReflectionFunctionAbstract $function, string $name): ?object;
}
```

#### Class Metadata

To read the class metadata, use the `$reader->getClassMetadata()` method. It receives as 
input the `ReflectionClass` of the required class and returns a list of available metadata 
objects.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata 
objects you want to retrieve.

```php
$reflection = new ReflectionClass(User::class);

$attributes = $reader->getClassMetadata($reflection, Entity::class); 
// returns iterable<Entity>
```

To get one metadata object, you can use the method `$reader->firstClassMetadata()`.

```php
$reflection = new ReflectionClass(User::class);

$attribute = $reader->firstClassMetadata($reflection, Entity::class); 
// returns Entity|null
```

#### Property Metadata

To read the property metadata, use the `$reader->getPropertyMetadata()` method. It receives as
input the `ReflectionProperty` of the required property and returns a list of available metadata
objects.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata
objects you want to retrieve.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$attributes = $reader->getPropertyMetadata($reflection, Column::class); 
// returns iterable<Column>
```

To get one metadata object, you can use the method `$reader->firstPropertyMetadata()`.

```php
$reflection = new ReflectionProperty(User::class, 'name');

$column = $reader->firstPropertyMetadata($reflection, Column::class); 
// returns Column|null
```

#### Function Metadata

To read the function metadata, use the `$reader->getFunctionMetadata()` method. It receives an 
argument of the `ReflectionFunction` or `ReflectionMethod` type of the required function and 
returns a list of available metadata objects.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getFunctionMetadata($reflection); 
// returns iterable<object>
```

The second optional argument `$name` of the method allows you to specify which specific metadata
objects you want to retrieve.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$attributes = $reader->getPropertyMetadata($reflection, DTOGetter::class); 
// returns iterable<DTOGetter>
```

To get one metadata object, you can use the method `$reader->firstFunctionMetadata()`.

```php
$reflection = new ReflectionMethod(RequestData::class, 'getEmail');

$getter = $reader->firstFunctionMetadata($reflection, DTOGetter::class); 
// returns DTOGetter|null
```

### Create Annotations

> For details on using doctrine annotations, please see 
> the [doctrine documentation](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/index.html#create-an-annotation-class).

You should use "hybrid" syntax to create metadata classes that will work on any version of PHP.

```php
/**
 * @Annotation
 * @Target({ "CLASS" })
 */
#[Attribute(Attribute::TARGET_CLASS)]
class Entity
{
    public string $table;
}
```

In this case you can use this metadata class on any PHP version.

```php
/**
 * PHP 8.0 syntax
 */
#[Entity(table: 'users')] class User {}

/**
 * PHP 7.x syntax
 * @Entity(table="users")
 */
class User {}
```

### Drivers

The `Spiral\Attributes\Manager` encapsulates several implementations behind it and returns a 
[selective reader](#attributes-usage-drivers-selective-reader) implementation by default, which is 
suitable for most cases. However, you can require a specific implementation  if available on your 
platform and/or in your application.

```php
use Spiral\Attributes\Manager;
use Spiral\Attributes\Reader\DoctrineReader;

$reader = (new Manager())->get(DoctrineReader::class);

// The "$reader" will be the implementation of the doctrine's reader
// or throws "Spiral\Attributes\Exception\NotFoundException" in the
// case that no implementation is available.
```

#### Doctrine Reader

> Please note that in order for this reader to be available in the application, you 
> need to require "doctrine/annotations" component.

This reader implementation allows reading doctrine annotations.

```php
/** @ExampleAnnotation */
class Example {}

$annotations = $manager->get(Spiral\Attributes\Reader\DoctrineReader::class)
    ->getClassMetadata(new ReflectionClass(Example::class))
;
// returns iterable<ExampleAnnotation>
```

#### Native Reader

> This implementation is only available since PHP 8.0

This reader implementation allows reading native PHP attributes.

```php
#[ExampleAttribute]
class Example {}

$attributes = $manager->get(Spiral\Attributes\Reader\NativeReader::class)
    ->getClassMetadata(new ReflectionClass(Example::class))
;
// returns iterable<ExampleAttribute>
```

#### Selective Reader

The implementation automatically selects the correct reader based on the syntax used in the 
application. This behavior is required if you use both syntaxes at the same time in the same 
project. For example, in the case of an already working project, which is refactored and the 
syntax of annotations is translated into the modern syntax of attributes.

```php
/** @ExampleAnnotation */
class ClassWithAnnotations {}
#[ExampleAttribute]
class ClassWithAttributes {}

$reader = $manager->get(Spiral\Attributes\Reader\SelectiveReader::class);

$annotations = $reader->getClassMetadata(new ReflectionClass(ClassWithAnnotations::class));
// returns iterable<ExampleAnnotation>

$attributes = $reader->getClassMetadata(new ReflectionClass(ClassWithAttributes::class));
// returns iterable<ExampleAttribute>
```

> When using both annotations and attributes in the same place, the 
> behavior of this reader is non-deterministic.

#### Merge Reader

The reader's implementation combines several syntaxes in one. This behavior is required 
if you are working with multiple libraries at the same time that support either only the
old or only the new syntax.

```php
/** @DoctrineAnnotation */
#[NativeAttribute]
class ExampleClass {}

$reader = $manager->get(Spiral\Attributes\Reader\MergeReader::class);

$metadata = $reader->getClassMetadata(new ReflectionClass(ExampleClass::class));
// returns iterable { DoctrineAnnotation, NativeAttribute }
```