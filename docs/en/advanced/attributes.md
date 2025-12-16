# Advanced — Attributes

PHP Attributes provide a structured way to add metadata to classes, methods, properties, constants, and parameters. This
metadata enables declarative configuration, behavior modification, and enhanced code documentation without cluttering
your business logic.

The `spiral/attributes` component offers a unified interface for reading both native PHP 8 attributes and legacy
Doctrine annotations, making it ideal for projects in transition or working with mixed metadata sources.

## Key Benefits

**Declarative Configuration**
Define behavior through metadata rather than procedural configuration, making code more self-documenting and reducing
boilerplate.

**Metadata Unification**
Work seamlessly with both PHP 8+ attributes and Doctrine annotations through a single API—no need to check which format
developers are using.

**Version Compatibility**
Use modern PHP 8 attribute syntax even in PHP 7.2+ projects by leveraging the Doctrine annotation bridge.

**Aspect-Oriented Programming**
Combine attributes with interceptors to implement cross-cutting concerns (logging, caching, validation) without
modifying business logic.

## Common Use Cases

**ORM Entity Mapping**

```php
#[Entity(table: 'users')]
class User
{
    #[Column(type: 'integer', primary: true)]
    public int $id;
    
    #[Column(type: 'string', length: 255)]
    public string $email;
}
```

**Route Definition**

```php
class UserController
{
    #[Route(path: '/users', methods: ['GET'])]
    public function list(): array
    {
        // Return user list
    }
}
```

**Validation Rules**

```php
class CreateUserRequest
{
    #[NotEmpty, Email]
    public string $email;
    
    #[NotEmpty, MinLength(8)]
    public string $password;
}
```

**Event Listeners**

```php
class UserRegistrationHandler
{
    #[Listener(event: UserRegistered::class)]
    public function sendWelcomeEmail(UserRegistered $event): void
    {
        // Send email
    }
}
```

> **Note**
> Throughout this documentation, "metadata" refers to both PHP attributes and Doctrine annotations interchangeably.

## Installation

Install the component via Composer:

```terminal 
composer require spiral/attributes
```

### Framework Integration

Register the `AttributesBootloader` to enable attribute reading throughout your application:

:::: tabs

::: tab Using method

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

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Attributes\AttributesBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

### Doctrine Annotation Configuration

When using Doctrine annotations alongside PHP attributes, you may encounter errors about unrecognized annotations.
Configure Doctrine to ignore specific annotations that aren't meant to be processed:

```php app/src/Application/Bootloader/AttributesBootloader.php
namespace App\Application\Bootloader;

use Doctrine\Common\Annotations\AnnotationReader;
use Spiral\Boot\Bootloader\Bootloader;

class AttributesBootloader extends Bootloader
{
    public function boot(): void
    {
        // Ignore PHPDoc annotations that aren't Doctrine annotations
        AnnotationReader::addGlobalIgnoredName('note');
        AnnotationReader::addGlobalIgnoredName('mixin');
        AnnotationReader::addGlobalIgnoredName('yield');
        AnnotationReader::addGlobalIgnoredName('type');
    }
}
```

> **Note**
> Place this bootloader early in your bootloader list (in the `SYSTEM` section) to ensure it executes before any
> attribute reading occurs. The Attributes SDK already configures these common ignored names internally, but you may need
> to add project-specific ones.

## Quick Start

The `ReaderInterface` provides methods to read metadata from any PHP reflection target:

```php
use Spiral\Attributes\ReaderInterface;

class AttributeProcessor
{
    public function __construct(
        private readonly ReaderInterface $reader,
    ) {}
    
    public function processClass(\ReflectionClass $class): void
    {
        // Read all metadata from the class
        foreach ($this->reader->getClassMetadata($class) as $metadata) {
            // Process each attribute/annotation
        }
    }
}
```

**Core Interface Methods**

| Method                     | Return Type        | Description                                          |
|----------------------------|--------------------|------------------------------------------------------|
| `getClassMetadata()`       | `iterable<object>` | Returns all metadata from a class (including traits) |
| `getPropertyMetadata()`    | `iterable<object>` | Returns all metadata from a property                 |
| `getFunctionMetadata()`    | `iterable<object>` | Returns all metadata from a method/function          |
| `getConstantMetadata()`    | `iterable<object>` | Returns all metadata from a class constant           |
| `getParameterMetadata()`   | `iterable<object>` | Returns all metadata from a function parameter       |
| `firstClassMetadata()`     | `?object`          | Returns first matching class metadata or null        |
| `firstPropertyMetadata()`  | `?object`          | Returns first matching property metadata or null     |
| `firstFunctionMetadata()`  | `?object`          | Returns first matching function metadata or null     |
| `firstConstantMetadata()`  | `?object`          | Returns first matching constant metadata or null     |
| `firstParameterMetadata()` | `?object`          | Returns first matching parameter metadata or null    |

## Reading Metadata

### Class Metadata

Read metadata from classes using reflection. The reader automatically includes metadata from traits used by the class.

**Read all metadata:**

```php
$reflection = new \ReflectionClass(User::class);

// Get all metadata objects
$metadata = $reader->getClassMetadata($reflection); 
foreach ($metadata as $item) {
    // Process each metadata object
}
```

**Filter by specific type:**

```php
use Cycle\Annotated\Annotation\Entity;

$reflection = new \ReflectionClass(User::class);

// Get only Entity metadata
$entities = $reader->getClassMetadata($reflection, Entity::class); 
foreach ($entities as $entity) {
    echo $entity->getTable(); // Access Entity properties
}
```

**Get single metadata instance:**

```php
use Cycle\Annotated\Annotation\Entity;

$reflection = new \ReflectionClass(User::class);

// Get first Entity metadata or null
$entity = $reader->firstClassMetadata($reflection, Entity::class); 
if ($entity !== null) {
    echo $entity->getTable();
}
```

**Trait metadata support (since v2.10.0):**

Metadata from traits is automatically included when reading class metadata:

:::: tabs

::: tab PHP Attributes

```php
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Column;
use App\Behavior\CreatedAt;
use App\Behavior\UpdatedAt;

#[Entity(table: 'entities')]
class User
{
    use TimestampTrait;
}

#[CreatedAt]
#[UpdatedAt]
trait TimestampTrait
{
    #[Column(type: 'datetime')]
    private \DateTimeImmutable $createdAt;

    #[Column(type: 'datetime', nullable: true)]
    private ?\DateTimeImmutable $updatedAt = null;
}
```

```php
$reflection = new \ReflectionClass(User::class);

// Returns Entity, CreatedAt, UpdatedAt metadata
$metadata = $reader->getClassMetadata($reflection);
```

:::

::: tab Doctrine Annotations

```php
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Column;
use App\Behavior\CreatedAt;
use App\Behavior\UpdatedAt;

/**
 * @Entity(table="entities")
 */
class User
{
    use TimestampTrait;
}

/**
 * @CreatedAt
 * @UpdatedAt
 */
trait TimestampTrait
{
    /**
     * @Column(type="datetime")
     */
    private \DateTimeImmutable $createdAt;

    /**
     * @Column(type="datetime", nullable=true)
     */
    private ?\DateTimeImmutable $updatedAt = null;
}
```

```php
$reflection = new \ReflectionClass(User::class);

// Returns Entity, CreatedAt, UpdatedAt annotations
$metadata = $reader->getClassMetadata($reflection);
```

:::

::::

### Property Metadata

Read metadata from class properties to configure field behavior:

**Basic usage:**

```php
use Cycle\Annotated\Annotation\Column;

$reflection = new \ReflectionProperty(User::class, 'email');

// Get all property metadata
$metadata = $reader->getPropertyMetadata($reflection);

// Get specific metadata type
$columns = $reader->getPropertyMetadata($reflection, Column::class);

// Get first matching metadata
$column = $reader->firstPropertyMetadata($reflection, Column::class);
if ($column !== null) {
    echo $column->getType(); // string
    var_dump($column->isNullable()); // false
}
```

### Function Metadata

Read metadata from methods and functions to configure behavior or route registration:

**Basic usage:**

```php
use Spiral\Router\Annotation\Route;

$reflection = new \ReflectionMethod(UserController::class, 'list');

// Get all method metadata
$metadata = $reader->getFunctionMetadata($reflection);

// Get specific metadata type
$routes = $reader->getFunctionMetadata($reflection, Route::class);

// Get first matching metadata
$route = $reader->firstFunctionMetadata($reflection, Route::class);
if ($route !== null) {
    echo $route->getPath(); // /users
    var_dump($route->getMethods()); // ['GET']
}
```

**Works with both methods and functions:**

```php
// For class methods
$methodReflection = new \ReflectionMethod(SomeClass::class, 'someMethod');
$metadata = $reader->getFunctionMetadata($methodReflection);

// For standalone functions
$functionReflection = new \ReflectionFunction('someFunction');
$metadata = $reader->getFunctionMetadata($functionReflection);
```

### Constant Metadata

Read metadata from class constants (PHP 8.0+):

**Basic usage:**

```php
use App\Metadata\Deprecated;

class StatusCodes
{
    #[Deprecated(since: '2.0', alternative: 'STATUS_ACTIVE')]
    public const STATUS_OK = 1;
    
    public const STATUS_ACTIVE = 1;
}

$reflection = new \ReflectionClassConstant(StatusCodes::class, 'STATUS_OK');

// Get all constant metadata
$metadata = $reader->getConstantMetadata($reflection);

// Get first matching metadata
$deprecated = $reader->firstConstantMetadata($reflection, Deprecated::class);
if ($deprecated !== null) {
    echo $deprecated->getSince(); // 2.0
    echo $deprecated->getAlternative(); // STATUS_ACTIVE
}
```

### Parameter Metadata

Read metadata from function/method parameters for validation or injection configuration:

**Basic usage:**

```php
use App\Validation\Email;
use App\Validation\NotEmpty;

function sendEmail(
    #[NotEmpty, Email] string $to,
    #[NotEmpty] string $subject,
    string $body
): void {
    // Send email
}

$reflection = new \ReflectionParameter('sendEmail', 'to');

// Get all parameter metadata
$metadata = $reader->getParameterMetadata($reflection);

// Get specific metadata types
$validators = $reader->getParameterMetadata($reflection, NotEmpty::class);

// Get first matching metadata
$emailValidator = $reader->firstParameterMetadata($reflection, Email::class);
```

**Works with method parameters:**

```php
class EmailService
{
    public function send(
        #[NotEmpty, Email] string $to,
        #[NotEmpty] string $subject
    ): void {
        // Send email
    }
}

$reflection = new \ReflectionParameter([EmailService::class, 'send'], 'to');
$metadata = $reader->getParameterMetadata($reflection);
```

## Creating Metadata Classes

Define custom metadata classes that work with both PHP attributes and Doctrine annotations:

**Hybrid syntax (PHP 7.2 - 8.x compatible):**

```php
/**
 * @Annotation
 * @Target({"CLASS"})
 */
#[\Attribute(\Attribute::TARGET_CLASS)]
class Table
{
    public function __construct(
        public string $name,
        public ?string $database = null,
    ) {}
}
```

This metadata class works on any PHP version:

:::: tabs

::: tab PHP Attributes (8.0+)

```php
#[Table(name: 'users', database: 'main')] 
class User {}
```

:::

::: tab Doctrine Annotations (7.2+)

```php
/**
 * @Table(name="users", database="main")
 */
class User {}
```

:::

::::

**Attribute targets:**

Specify where your metadata can be applied using the `@Target` annotation and `#[\Attribute]` flags:

| Target Constant         | Applies To | Doctrine Annotation      | PHP Attribute                       |
|-------------------------|------------|--------------------------|-------------------------------------|
| `TARGET_CLASS`          | Classes    | `@Target({"CLASS"})`     | `\Attribute::TARGET_CLASS`          |
| `TARGET_METHOD`         | Methods    | `@Target({"METHOD"})`    | `\Attribute::TARGET_METHOD`         |
| `TARGET_PROPERTY`       | Properties | `@Target({"PROPERTY"})`  | `\Attribute::TARGET_PROPERTY`       |
| `TARGET_FUNCTION`       | Functions  | `@Target({"FUNCTION"})`  | `\Attribute::TARGET_FUNCTION`       |
| `TARGET_PARAMETER`      | Parameters | `@Target({"PARAMETER"})` | `\Attribute::TARGET_PARAMETER`      |
| `TARGET_CLASS_CONSTANT` | Constants  | N/A                      | `\Attribute::TARGET_CLASS_CONSTANT` |
| `TARGET_ALL`            | All        | `@Target({"ALL"})`       | `\Attribute::TARGET_ALL`            |

**Multiple targets example:**

```php
/**
 * @Annotation
 * @Target({"CLASS", "METHOD", "PROPERTY"})
 */
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD | \Attribute::TARGET_PROPERTY)]
class Cached
{
    public function __construct(
        public int $ttl = 3600,
    ) {}
}
```

## Instantiation Strategies

The Attributes SDK supports multiple ways to instantiate metadata classes. Understanding these patterns helps you choose
the right approach for your use case.

### Property-Based Instantiation (Doctrine Style)

Public properties are automatically populated from metadata arguments:

```php
/**
 * @Annotation
 */
#[\Attribute]
class Route
{
    public string $path;
    public array $methods = ['GET'];
    public ?string $name = null;
}
```

**Usage:**

```php
#[Route(path: '/users', methods: ['GET', 'POST'], name: 'user.list')]
class UserController {}
```

> **See also**
> [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

### Array-Based Constructor Instantiation

When a constructor accepts an array parameter, all arguments are passed as a single array:

```php
/**
 * @Annotation
 */
#[\Attribute]
class Validation
{
    private array $rules;
    
    public function __construct(array $data)
    {
        // $data = ['min' => 5, 'max' => 100]
        $this->rules = $data;
    }
}
```

**Usage:**

```php
#[Validation(min: 5, max: 100)]
private int $age;
```

> **See also**
> [Doctrine Custom Annotations](https://www.doctrine-project.org/projects/doctrine-annotations/en/1.10/custom.html#custom-annotation-classes)

### Named Arguments Constructor (Recommended)

Use named constructor parameters for better IDE support and type safety. Mark the class using either an interface or
metadata attribute.

**Using interface marker (requires spiral/attributes):**

```php
use Spiral\Attributes\NamedArgumentConstructorAttribute;

/**
 * @Annotation
 */
#[\Attribute]
class Column implements NamedArgumentConstructorAttribute
{
    public function __construct(
        public string $type,
        public bool $nullable = false,
        public ?int $length = null,
    ) {}
}
```

**Using metadata marker (framework-independent):**

```php
use Spiral\Attributes\NamedArgumentConstructor;

/**
 * @Annotation
 * @NamedArgumentConstructor
 */
#[\Attribute]
#[NamedArgumentConstructor]
class Column
{
    public function __construct(
        public string $type,
        public bool $nullable = false,
        public ?int $length = null,
    ) {}
}
```

**Usage:**

```php
#[Column(type: 'string', length: 255)]
private string $email;

#[Column(type: 'integer', nullable: true)]
private ?int $age;
```

**Benefits of named arguments:**

- IDE autocomplete and type hints
- Clear parameter names at usage site
- Optional parameters with defaults
- No dependency on `spiral/attributes` when using metadata marker

## Reader Implementations

Choose the appropriate reader based on your project's needs:

### Factory (Recommended)

The `Factory` class automatically selects the best reader implementation and configures caching:

```php
use Spiral\Attributes\Factory;

// Create default reader (SelectiveReader with both annotation and attribute support)
$reader = (new Factory())->create();

// With PSR-6 or PSR-16 cache
$reader = (new Factory())
    ->withCache($cacheImplementation)
    ->create();
```

**Default behavior:**

- Returns `SelectiveReader` that tries attributes first, then annotations
- Automatically includes `AnnotationReader` if `doctrine/annotations` is installed
- Configures common ignored annotation names (e.g., `@mixin`, `@note`)

### AttributeReader

Reads native PHP 8+ attributes. Works on any PHP version but only reads attribute syntax:

```php
use Spiral\Attributes\AttributeReader;

#[ExampleAttribute]
class Example {}

$reader = new AttributeReader();

$attributes = $reader->getClassMetadata(new \ReflectionClass(Example::class));
// Returns: iterable<ExampleAttribute>
```

**Use when:**

- Your project uses PHP 8+ attributes exclusively
- You don't need Doctrine annotation compatibility
- You want maximum performance (no annotation parsing overhead)

### AnnotationReader (Legacy)

Reads Doctrine annotations. Requires `doctrine/annotations` package:

```php
use Spiral\Attributes\AnnotationReader;

/**
 * @ExampleAnnotation
 */
class Example {}

$reader = new AnnotationReader();

$annotations = $reader->getClassMetadata(new \ReflectionClass(Example::class));
// Returns: iterable<ExampleAnnotation>
```

**Use when:**

- Maintaining legacy codebases with Doctrine annotations
- Gradual migration from annotations to attributes
- Working with libraries that only support annotations

> **Note**
> Requires `composer require doctrine/annotations`

### SelectiveReader

Automatically chooses between attributes and annotations based on what's present. Best for migration scenarios:

**Example classes:**

:::: tabs

::: tab PHP Attributes

```php
#[ExampleAttribute]
class ClassWithAttributes {}
```

:::

::: tab Doctrine Annotations

```php
/** 
 * @ExampleAnnotation 
 */
class ClassWithAnnotations {}
```

:::

::::

**Reader setup:**

```php
use Spiral\Attributes\Composite\SelectiveReader;
use Spiral\Attributes\AnnotationReader;
use Spiral\Attributes\AttributeReader;

$reader = new SelectiveReader([
    new AttributeReader(),
    new AnnotationReader(),
]);

// Reads attributes from ClassWithAttributes
$attributes = $reader->getClassMetadata(
    new \ReflectionClass(ClassWithAttributes::class)
);

// Reads annotations from ClassWithAnnotations
$annotations = $reader->getClassMetadata(
    new \ReflectionClass(ClassWithAnnotations::class)
);
```

**Use when:**

- Migrating from Doctrine annotations to PHP attributes
- Different classes use different metadata syntax
- You want automatic detection of metadata format

> **Note**
> If both attributes and annotations are present on the same element, behavior is non-deterministic (first reader wins).

### MergeReader

Combines metadata from multiple readers. Useful when working with mixed libraries:

**Example class with both:**

```php
/**
 * @DoctrineAnnotation
 */
#[NativeAttribute]
class ExampleClass {}
```

**Reader setup:**

```php
use Spiral\Attributes\Composite\MergeReader;
use Spiral\Attributes\AnnotationReader;
use Spiral\Attributes\AttributeReader;

$reader = new MergeReader([
    new AttributeReader(),
    new AnnotationReader(),
]);

$metadata = $reader->getClassMetadata(new \ReflectionClass(ExampleClass::class));
// Returns: iterable containing both DoctrineAnnotation and NativeAttribute
```

**Use when:**

- Working with multiple libraries requiring different metadata formats
- You need all metadata from all sources combined
- Integrating legacy and modern components

**Comparison table:**

| Reader             | Reads Attributes | Reads Annotations | Combines Both   | Best For                 |
|--------------------|------------------|-------------------|-----------------|--------------------------|
| `AttributeReader`  | ✅                | ❌                 | ❌               | Modern PHP 8+ projects   |
| `AnnotationReader` | ❌                | ✅                 | ❌               | Legacy Doctrine projects |
| `SelectiveReader`  | ✅                | ✅                 | ❌ (first found) | Migration scenarios      |
| `MergeReader`      | ✅                | ✅                 | ✅               | Mixed library ecosystems |

## Performance Optimization with Caching

Metadata reading involves reflection and parsing, which can be expensive. Use caching for production environments.

### PSR-6 Cache (Symfony Cache, etc.)

```php
use Spiral\Attributes\Psr6CachedReader;
use Spiral\Attributes\AttributeReader;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new FilesystemAdapter();
$reader = new Psr6CachedReader(
    new AttributeReader(),
    $cache
);

// First call: reads and caches
$metadata = $reader->getClassMetadata(new \ReflectionClass(User::class));

// Subsequent calls: returns from cache
$metadata = $reader->getClassMetadata(new \ReflectionClass(User::class));
```

### PSR-16 Cache (Simple Cache)

```php
use Spiral\Attributes\Psr16CachedReader;
use Spiral\Attributes\AttributeReader;
use Symfony\Component\Cache\Psr16Cache;
use Symfony\Component\Cache\Adapter\FilesystemAdapter;

$cache = new Psr16Cache(new FilesystemAdapter());
$reader = new Psr16CachedReader(
    new AttributeReader(),
    $cache
);

$metadata = $reader->getClassMetadata(new \ReflectionClass(User::class));
```

### Factory with Cache

The simplest approach using the factory:

```php
use Spiral\Attributes\Factory;

$reader = (new Factory())
    ->withCache($cacheImplementation) // PSR-6 or PSR-16
    ->create();
```

### Cache Key Generation

By default, cached readers generate keys using:

- File modification time (for user-defined classes)
- Extension version (for built-in classes)
- Unique reflection identifiers

**Custom key generation:**

```php
use Spiral\Attributes\Psr6CachedReader;
use Spiral\Attributes\Internal\Key\NameKeyGenerator;

$reader = new Psr6CachedReader(
    new AttributeReader(),
    $cache,
    new NameKeyGenerator() // Only class/method names, no modification time
);
```

**Performance considerations:**

- Cache increases read performance by 10-100x for repeated access
- Cache keys automatically invalidate when source files change
- Disable cache in development for immediate reflection of code changes
- Essential for production where classes are stable

## Practical Examples

### Building a Route Registry

```php
use Spiral\Attributes\ReaderInterface;
use Spiral\Router\Annotation\Route;

class RouteRegistrar
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {}
    
    public function registerController(string $class): array
    {
        $routes = [];
        $reflection = new \ReflectionClass($class);
        
        foreach ($reflection->getMethods() as $method) {
            $route = $this->reader->firstFunctionMetadata($method, Route::class);
            
            if ($route !== null) {
                $routes[] = [
                    'path' => $route->getPath(),
                    'methods' => $route->getMethods(),
                    'handler' => [$class, $method->getName()],
                ];
            }
        }
        
        return $routes;
    }
}
```

### Validating Entity Properties

```php
use Spiral\Attributes\ReaderInterface;

class EntityValidator
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {}
    
    public function validate(object $entity): array
    {
        $errors = [];
        $reflection = new \ReflectionClass($entity);
        
        foreach ($reflection->getProperties() as $property) {
            $validators = $this->reader->getPropertyMetadata($property);
            
            foreach ($validators as $validator) {
                if (!$this->checkValidator($validator, $property, $entity)) {
                    $errors[] = sprintf(
                        'Property %s failed validation: %s',
                        $property->getName(),
                        $validator::class
                    );
                }
            }
        }
        
        return $errors;
    }
    
    private function checkValidator(object $validator, \ReflectionProperty $property, object $entity): bool
    {
        // Validation logic
        return true;
    }
}
```

### Event Listener Registration

```php
use Spiral\Attributes\ReaderInterface;
use App\Attributes\Listener;

class ListenerRegistrar
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {}
    
    public function discoverListeners(array $classes): array
    {
        $listeners = [];
        
        foreach ($classes as $class) {
            $reflection = new \ReflectionClass($class);
            
            foreach ($reflection->getMethods() as $method) {
                $listener = $this->reader->firstFunctionMetadata($method, Listener::class);
                
                if ($listener !== null) {
                    $listeners[$listener->event][] = [
                        'handler' => [$class, $method->getName()],
                        'priority' => $listener->priority ?? 0,
                    ];
                }
            }
        }
        
        return $listeners;
    }
}
```

## Best Practices

**Use specific metadata filtering:**

```php
// ❌ Avoid: Reading all metadata when you need specific type
$all = $reader->getClassMetadata($reflection);
foreach ($all as $item) {
    if ($item instanceof Entity) {
        // Process
    }
}

// ✅ Prefer: Filter at read time
$entity = $reader->firstClassMetadata($reflection, Entity::class);
```

**Cache in production:**

```php
// ❌ Avoid: No caching in production
$reader = new AttributeReader();

// ✅ Prefer: Cache for production performance
$reader = (new Factory())
    ->withCache($cache)
    ->create();
```

**Handle missing metadata gracefully:**

```php
// ✅ Always check for null
$entity = $reader->firstClassMetadata($reflection, Entity::class);
if ($entity === null) {
    throw new \RuntimeException('Entity metadata required');
}
```

**Use appropriate reader for your use case:**

```php
// Modern project: AttributeReader
$reader = new AttributeReader();

// Legacy project: AnnotationReader  
$reader = new AnnotationReader();

// Migration phase: SelectiveReader via Factory
$reader = (new Factory())->create();

// Multiple libraries: MergeReader
$reader = new MergeReader([/* readers */]);
```

## Troubleshooting

**Problem: "Class X not found" errors**

Solution: Ensure metadata classes are autoloaded before reading:

```php
// Ensure attribute class is loaded
class_exists(MyAttribute::class, true);

$metadata = $reader->getClassMetadata($reflection);
```

**Problem: Doctrine annotation parsing errors**

Solution: Configure ignored annotations in your bootloader:

```php![img.png](img.png)
use Doctrine\Common\Annotations\AnnotationReader;

AnnotationReader::addGlobalIgnoredName('psalm');
AnnotationReader::addGlobalIgnoredName('phpstan');
AnnotationReader::addGlobalIgnoredName('internal');
```

**Problem: Cache not invalidating**

Solution: Verify file modification detection:

```php
// Default key generator includes modification time
$reader = new Psr6CachedReader(
    new AttributeReader(),
    $cache,
    null // Uses default: NameKeyGenerator + ModificationTimeKeyGenerator
);
```

**Problem: Performance issues**

Solution: Use caching and specific filtering:

```php
// Enable cache
$reader = (new Factory())->withCache($cache)->create();

// Use specific type filtering
$entity = $reader->firstClassMetadata($class, Entity::class);
// Instead of getClassMetadata() with manual filtering
```
