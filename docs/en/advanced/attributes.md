# Advanced — Attributes

PHP Attributes (introduced in PHP 8.0) provide a structured, native way to add metadata to classes, methods, properties,
constants, and parameters. This metadata enables declarative configuration, behavior modification, and enhanced code
documentation without cluttering your business logic.

The `spiral/attributes` component provides a powerful interface for reading and processing PHP attributes throughout
your application.

## Key Benefits

**Declarative Configuration**
Define behavior through metadata rather than procedural configuration, making code self-documenting and reducing
boilerplate.

**Native PHP Support**
Use PHP 8.0+ native attribute syntax with full language-level support, IDE integration, and static analysis.

**Reflection Integration**
Seamlessly work with PHP's reflection API to discover and process attributes at runtime.

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

## Quick Start

The `ReaderInterface` provides methods to read metadata from any PHP reflection target:

```php
use Spiral\Attributes\ReaderInterface;

class AttributeProcessor
{
    public function __construct(
        private readonly ReaderInterface $reader
    ) {}
    
    public function processClass(\ReflectionClass $class): void
    {
        // Read all attributes from the class
        foreach ($this->reader->getClassMetadata($class) as $attribute) {
            // Process each attribute
        }
    }
}
```

**Core Interface Methods**

| Method                     | Return Type        | Description                                            |
|----------------------------|--------------------|--------------------------------------------------------|
| `getClassMetadata()`       | `iterable<object>` | Returns all attributes from a class (including traits) |
| `getPropertyMetadata()`    | `iterable<object>` | Returns all attributes from a property                 |
| `getFunctionMetadata()`    | `iterable<object>` | Returns all attributes from a method/function          |
| `getConstantMetadata()`    | `iterable<object>` | Returns all attributes from a class constant           |
| `getParameterMetadata()`   | `iterable<object>` | Returns all attributes from a function parameter       |
| `firstClassMetadata()`     | `?object`          | Returns first matching class attribute or null         |
| `firstPropertyMetadata()`  | `?object`          | Returns first matching property attribute or null      |
| `firstFunctionMetadata()`  | `?object`          | Returns first matching function attribute or null      |
| `firstConstantMetadata()`  | `?object`          | Returns first matching constant attribute or null      |
| `firstParameterMetadata()` | `?object`          | Returns first matching parameter attribute or null     |

## Reading Attributes

### Class Attributes

Read attributes from classes using reflection. The reader automatically includes attributes from traits used by the
class.

**Read all attributes:**

```php
$reflection = new \ReflectionClass(User::class);

// Get all attribute objects
$attributes = $reader->getClassMetadata($reflection); 
foreach ($attributes as $attribute) {
    // Process each attribute object
}
```

**Filter by specific type:**

```php
use Cycle\Annotated\Annotation\Entity;

$reflection = new \ReflectionClass(User::class);

// Get only Entity attributes
$entities = $reader->getClassMetadata($reflection, Entity::class); 
foreach ($entities as $entity) {
    echo $entity->getTable(); // Access Entity properties
}
```

**Get single attribute instance:**

```php
use Cycle\Annotated\Annotation\Entity;

$reflection = new \ReflectionClass(User::class);

// Get first Entity attribute or null
$entity = $reader->firstClassMetadata($reflection, Entity::class); 
if ($entity !== null) {
    echo $entity->getTable();
}
```

**Trait attribute support (since v2.10.0):**

Attributes from traits are automatically included when reading class attributes:

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

// Returns Entity, CreatedAt, UpdatedAt attributes
$attributes = $reader->getClassMetadata($reflection);
```

### Property Attributes

Read attributes from class properties to configure field behavior:

```php
use Cycle\Annotated\Annotation\Column;

$reflection = new \ReflectionProperty(User::class, 'email');

// Get all property attributes
$attributes = $reader->getPropertyMetadata($reflection);

// Get specific attribute type
$columns = $reader->getPropertyMetadata($reflection, Column::class);

// Get first matching attribute
$column = $reader->firstPropertyMetadata($reflection, Column::class);
if ($column !== null) {
    echo $column->getType(); // string
    var_dump($column->isNullable()); // false
}
```

### Function Attributes

Read attributes from methods and functions to configure behavior or route registration:

```php
use Spiral\Router\Annotation\Route;

$reflection = new \ReflectionMethod(UserController::class, 'list');

// Get all method attributes
$attributes = $reader->getFunctionMetadata($reflection);

// Get specific attribute type
$routes = $reader->getFunctionMetadata($reflection, Route::class);

// Get first matching attribute
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
$attributes = $reader->getFunctionMetadata($methodReflection);

// For standalone functions
$functionReflection = new \ReflectionFunction('someFunction');
$attributes = $reader->getFunctionMetadata($functionReflection);
```

### Constant Attributes

Read attributes from class constants (PHP 8.0+):

```php
use App\Metadata\Deprecated;

class StatusCodes
{
    #[Deprecated(since: '2.0', alternative: 'STATUS_ACTIVE')]
    public const STATUS_OK = 1;
    
    public const STATUS_ACTIVE = 1;
}

$reflection = new \ReflectionClassConstant(StatusCodes::class, 'STATUS_OK');

// Get all constant attributes
$attributes = $reader->getConstantMetadata($reflection);

// Get first matching attribute
$deprecated = $reader->firstConstantMetadata($reflection, Deprecated::class);
if ($deprecated !== null) {
    echo $deprecated->getSince(); // 2.0
    echo $deprecated->getAlternative(); // STATUS_ACTIVE
}
```

### Parameter Attributes

Read attributes from function/method parameters for validation or injection configuration:

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

// Get all parameter attributes
$attributes = $reader->getParameterMetadata($reflection);

// Get specific attribute types
$validators = $reader->getParameterMetadata($reflection, NotEmpty::class);

// Get first matching attribute
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
$attributes = $reader->getParameterMetadata($reflection);
```

## Creating Attribute Classes

Define custom attribute classes for your application:

```php
#[\Attribute(\Attribute::TARGET_CLASS)]
class Table
{
    public function __construct(
        public string $name,
        public ?string $database = null,
    ) {}
}
```

**Usage:**

```php
#[Table(name: 'users', database: 'main')] 
class User {}
```

### Attribute Targets

Specify where your attributes can be applied using the `#[\Attribute]` declaration:

| Target Constant         | Applies To | Example                                            |
|-------------------------|------------|----------------------------------------------------|
| `TARGET_CLASS`          | Classes    | `#[\Attribute(\Attribute::TARGET_CLASS)]`          |
| `TARGET_METHOD`         | Methods    | `#[\Attribute(\Attribute::TARGET_METHOD)]`         |
| `TARGET_PROPERTY`       | Properties | `#[\Attribute(\Attribute::TARGET_PROPERTY)]`       |
| `TARGET_FUNCTION`       | Functions  | `#[\Attribute(\Attribute::TARGET_FUNCTION)]`       |
| `TARGET_PARAMETER`      | Parameters | `#[\Attribute(\Attribute::TARGET_PARAMETER)]`      |
| `TARGET_CLASS_CONSTANT` | Constants  | `#[\Attribute(\Attribute::TARGET_CLASS_CONSTANT)]` |
| `TARGET_ALL`            | All        | `#[\Attribute(\Attribute::TARGET_ALL)]`            |

**Multiple targets example:**

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::TARGET_METHOD | \Attribute::TARGET_PROPERTY)]
class Cached
{
    public function __construct(
        public int $ttl = 3600,
    ) {}
}
```

### Repeatable Attributes

Allow the same attribute to be used multiple times on a single element:

```php
#[\Attribute(\Attribute::TARGET_CLASS | \Attribute::IS_REPEATABLE)]
class Tag
{
    public function __construct(
        public string $name,
    ) {}
}
```

**Usage:**

```php
#[Tag('api')]
#[Tag('v1')]
#[Tag('user-management')]
class UserController {}
```

## Attribute Design Patterns

### Named Arguments Constructor (Recommended)

Use named constructor parameters for better IDE support and type safety:

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Column
{
    public function __construct(
        public string $type,
        public bool $nullable = false,
        public ?int $length = null,
        public bool $unique = false,
    ) {}
}
```

**Usage with named arguments:**

```php
#[Column(type: 'string', length: 255, unique: true)]
private string $email;

#[Column(type: 'integer', nullable: true)]
private ?int $age;
```

**Benefits:**

- IDE autocomplete and type hints
- Clear parameter names at usage site
- Optional parameters with defaults
- Compile-time validation

### Property-Based Pattern

Use public properties for simple data containers:

```php
#[\Attribute(\Attribute::TARGET_METHOD)]
class Route
{
    public string $path;
    public array $methods = ['GET'];
    public ?string $name = null;
    
    public function __construct(
        string $path,
        array $methods = ['GET'],
        ?string $name = null,
    ) {
        $this->path = $path;
        $this->methods = $methods;
        $this->name = $name;
    }
}
```

### Validation in Attributes

Add validation logic in attribute constructors:

```php
#[\Attribute(\Attribute::TARGET_PROPERTY)]
class Range
{
    public function __construct(
        public int $min,
        public int $max,
    ) {
        if ($min > $max) {
            throw new \InvalidArgumentException('Min cannot be greater than max');
        }
    }
}
```

## Reader Implementations

The component provides several reader implementations:

### AttributeReader

The default and recommended reader for PHP 8+ attributes:

```php
use Spiral\Attributes\AttributeReader;

$reader = new AttributeReader();

$attributes = $reader->getClassMetadata(new \ReflectionClass(User::class));
```

### Factory (Recommended)

Use the factory to create a properly configured reader:

```php
use Spiral\Attributes\Factory;

// Create default reader
$reader = (new Factory())->create();

// With cache
$reader = (new Factory())
    ->withCache($cacheImplementation)
    ->create();
```

## Performance Optimization with Caching

Attribute reading involves reflection, which can be expensive. Use caching for production environments.

### PSR-6 Cache

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
$attributes = $reader->getClassMetadata(new \ReflectionClass(User::class));

// Subsequent calls: returns from cache
$attributes = $reader->getClassMetadata(new \ReflectionClass(User::class));
```

### PSR-16 Cache

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

$attributes = $reader->getClassMetadata(new \ReflectionClass(User::class));
```

### Factory with Cache

The simplest approach:

```php
use Spiral\Attributes\Factory;

$reader = (new Factory())
    ->withCache($cacheImplementation) // PSR-6 or PSR-16
    ->create();
```

**Performance considerations:**

- Cache increases read performance by 10-100x for repeated access
- Cache keys automatically invalidate when source files change
- Disable cache in development for immediate reflection of code changes
- Essential for production where classes are stable

## Discovering Classes with Attributes

For automatic discovery of classes with specific attributes, use the [Tokenizer component](./tokenizer.md).

The Tokenizer can scan your codebase to find all classes that use specific attributes, making it perfect for:

- Automatic route registration
- Event listener discovery
- Command registration
- Plugin/module discovery

**Example using Tokenizer:**

```php
use Spiral\Tokenizer\TokenizationListenerInterface;
use Spiral\Attributes\ReaderInterface;

#[\Spiral\Tokenizer\Attribute\TargetAttribute(Route::class)]
class RouteListener implements TokenizationListenerInterface
{
    private array $routes = [];

    public function __construct(
        private readonly ReaderInterface $reader
    ) {}
    
    public function listen(\ReflectionClass $class): void
    {
        foreach ($class->getMethods() as $method) {
            $route = $this->reader->firstFunctionMetadata($method, Route::class);

            if ($route !== null) {
                $this->routes[] = [
                    'path' => $route->path,
                    'handler' => [$class->getName(), $method->getName()],
                ];
            }
        }
    }

    public function finalize(): void
    {
        // Register all discovered routes
        foreach ($this->routes as $route) {
            // Add to router
        }
    }
}
```

> **See also**
> [Component — Static analysis](./tokenizer.md) for detailed information on class discovery and automatic registration.

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
        // Validation logic based on validator type
        $property->setAccessible(true);
        $value = $property->getValue($entity);
        
        return match (true) {
            $validator instanceof NotEmpty => !empty($value),
            $validator instanceof Email => filter_var($value, FILTER_VALIDATE_EMAIL) !== false,
            $validator instanceof MinLength => strlen($value) >= $validator->length,
            default => true,
        };
    }
}
```

### Event Listener Discovery

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
        
        // Sort by priority
        foreach ($listeners as &$eventListeners) {
            usort($eventListeners, fn($a, $b) => $b['priority'] <=> $a['priority']);
        }
        
        return $listeners;
    }
}
```

### Dependency Injection Configuration

```php
#[\Attribute(\Attribute::TARGET_PARAMETER)]
class Inject
{
    public function __construct(
        public ?string $id = null,
    ) {}
}

class ServiceFactory
{
    public function __construct(
        private readonly ReaderInterface $reader,
        private readonly ContainerInterface $container,
    ) {}
    
    public function createInstance(string $class): object
    {
        $reflection = new \ReflectionClass($class);
        $constructor = $reflection->getConstructor();
        
        if ($constructor === null) {
            return new $class();
        }
        
        $args = [];
        foreach ($constructor->getParameters() as $parameter) {
            $inject = $this->reader->firstParameterMetadata($parameter, Inject::class);
            
            if ($inject !== null) {
                $args[] = $this->container->get($inject->id ?? $parameter->getType()->getName());
            } else {
                $args[] = $this->container->get($parameter->getType()->getName());
            }
        }
        
        return $reflection->newInstanceArgs($args);
    }
}
```

## Best Practices

**Use specific attribute filtering:**

```php
// ❌ Avoid: Reading all attributes when you need specific type
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

**Handle missing attributes gracefully:**

```php
// ✅ Always check for null
$entity = $reader->firstClassMetadata($reflection, Entity::class);
if ($entity === null) {
    throw new \RuntimeException('Entity attribute required on ' . $reflection->getName());
}
```

**Validate in attribute constructors:**

```php
#[\Attribute]
class Range
{
    public function __construct(
        public int $min,
        public int $max,
    ) {
        if ($min > $max) {
            throw new \InvalidArgumentException('min must be less than or equal to max');
        }
    }
}
```

**Use descriptive attribute names:**

```php
// ❌ Avoid: Generic names
#[Config('table', 'users')]

// ✅ Prefer: Specific, clear names
#[Table(name: 'users')]
```

**Group related attributes:**

```php
// Create attribute groups for related configuration
#[Entity(table: 'users')]
#[HasTimestamps]
#[SoftDeletes]
class User {}
```

## Troubleshooting

**Problem: "Attribute class not found" errors**

Solution: Ensure attribute classes are autoloaded:

```php
// Verify class exists before reading
if (!class_exists(MyAttribute::class)) {
    throw new \RuntimeException('Attribute class not found: ' . MyAttribute::class);
}

$attributes = $reader->getClassMetadata($reflection);
```

**Problem: Attributes not being detected**

Solution: Verify attribute target matches usage:

```php
// Wrong: Attribute declared for TARGET_CLASS
#[\Attribute(\Attribute::TARGET_CLASS)]
class MyAttribute {}

// Used on method - won't work!
class Example {
    #[MyAttribute] // Error: attribute target mismatch
    public function method() {}
}

// Correct: Use appropriate target
#[\Attribute(\Attribute::TARGET_METHOD)]
class MyAttribute {}
```

**Problem: Performance issues**

Solution: Enable caching and use specific filtering:

```php
// Enable cache
$reader = (new Factory())->withCache($cache)->create();

// Use specific type filtering
$entity = $reader->firstClassMetadata($class, Entity::class);
```

**Problem: Attribute constructor errors**

Solution: Use named arguments for clarity:

```php
// ❌ Avoid: Positional arguments are error-prone
#[Route('/users', ['GET', 'POST'])]

// ✅ Prefer: Named arguments are clear
#[Route(path: '/users', methods: ['GET', 'POST'])]
```

## Migration from Doctrine Annotations

If you're migrating from Doctrine annotations to PHP attributes:

1. **Update PHP version**: Requires PHP 8.0+
2. **Convert annotation syntax**:
   ```php
   // Old (Doctrine annotation)
   /**
    * @Entity(table="users")
    */
   class User {}
   
   // New (PHP attribute)
   #[Entity(table: 'users')]
   class User {}
   ```
3. **Update attribute classes**: Remove `@Annotation` doc comments, add `#[\Attribute]`
4. **Test thoroughly**: Attributes have different parsing rules than annotations

For projects still using Doctrine annotations, consider the legacy compatibility features in previous versions of this
component, though migration to native PHP attributes is recommended.
