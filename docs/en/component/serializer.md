# Serializer

The Spiral Framework provides a component for serializing and deserializing data. Serialization is a complex topic. This
component provides only the simplest serializers out of the box and may not cover all your use cases. But it provides an
easy way to integrate serialization tools into the Spiral Framework or develop your solutions for serializing data in
your application.

The component is available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\Serializer\Bootloader\SerializerBootloader` to the bootloaders
list, which is located in the class of your application.

```php
namespace App;

use Spiral\Serializer\Bootloader\SerializerBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        SerializerBootloader::class,
        // ...
    ];
}
```

## Configuration

The configuration file for this component should be located at `app/config/serializer.php`. Within this file, you may 
configure an array of available serializers and default serializer.

For example, the configuration file might look like this:

```php
use Spiral\Core\Container\Autowire;
use Spiral\Serializer\Serializer\JsonSerializer;
use Spiral\Serializer\Serializer\PhpSerializer; 
use Spiral\Serializer\Serializer\CallbackSerializer;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default serializer
     * -------------------------------------------------------------------------
     * 
     * The key of one of the registered serializers to use by default.
     */
    'default' => 'json',
    
    /**
     * -------------------------------------------------------------------------
     *  Available serializers
     * -------------------------------------------------------------------------
     * 
     * Array of all available serializers.  
     */
    'serializers' => [
        // via fully qualified class name
        'json' => JsonSerializer::class,
        
        // via Autowire 
        'serializer' => new Autowire(PhpSerializer::class),
        
        // or manual instantiating object
        'callback' => new CallbackSerializer()
    ],
];
```

## Usage

Three serializers are available by default.

- `Spiral\Serializer\Serializer\JsonSerializer` - uses PHP functions `json_encode` and `json_decode`.
  It does not support data hydration to an object.
- `Spiral\Serializer\Serializer\PhpSerializer` - uses PHP functions `serialize` and `unserialize`.
- `Spiral\Serializer\Serializer\CallbackSerializer`- uses callbacks for serializing and deserializing data.

### SerializerInterface

The serializer can be injected from a container using the `Spiral\Serializer\SerializerInterface`. It will refer to the
default serializer.

```php
namespace App\Service;

use Spiral\Serializer\SerializerInterface;

class MyService
{
    public function __construct(
        private readonly SerializerInterface $serializer,
    ) {
    }

    public function someMethod(): void
    {
        // serialize
        $serialized = $this->serializer->serialize(['some' => 'data']);
        
        // unserialize
        $array = $this->serializer->unserialize($serialized);
    }
}
```

### SerializerManager

Using the `Spiral\Serializer\SerializerManager`, you can get a specific serializer by serializer string key from config.

```php
namespace App\Service;

use Spiral\Serializer\SerializerManager;

class MyService
{
    public function __construct(
        private readonly SerializerManager $serializer,
    ) {
    }

    public function someMethod(): void
    {
        $serialized = $this->serializer->getSerializer('json')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('json')->unserialize($serialized);

        $serialized = $this->serializer->getSerializer('serializer')->serialize(['some' => 'data']);
        $array = $this->serializer->getSerializer('serializer')->unserialize($serialized);
    }
}
```

## Creating a serializer

### Serializer class

Let's create a serializer that will implement the `SerializerInterface`:

```php
namespace App;

use Spiral\Serializer\SerializerInterface;

class CustomSerializer implements SerializerInterface
{
    public function serialize(mixed $payload): string|\Stringable
    {
        // ...
    }

    public function unserialize(\Stringable|string $payload, object|string|null $type = null): mixed
    {
        // ...
    }
}
```

You need to implement the `serialize` method with the `$payload` parameter, and this method should return
the serialized data. And the `unserialize` method, which in the parameters accepts serialized data `$payload`
and optionally the name of the class or object `$type` into which the payload should be deserialized.

### Registering a new Serializer

There are several ways to add a new serializer. Using `Spiral\Serializer\SerializerRegistryInterface`:

```php
namespace App\Bootloader;

use App\CustomSerializer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Serializer\SerializerRegistryInterface;

class SerializerBootloader extends Bootloader
{
    public function boot(SerializerRegistryInterface $registry): void
    {
        $registry->register('someName', new CustomSerializer());
    }
}
```

Using the config file:

```php
use App\Serializer;

return [
    // file app/config/serializer.php
    'serializers' => [
        'someName' => CustomSerializer::class,
        // other serializers
    ],
];
```
