# Serializer

The Spiral Framework helps you manage and organize your code. One aspect of this is the ability to serialize data, which
involves converting it into a format that can be stored or transmitted, and then later deserialized, or converted back
into its original form. This is useful in a variety of situations, such as when transferring data over a network or
storing it in a database.

The Spiral Framework includes some basic serialization tools by default, but these may not always be sufficient for more
complex use cases. In such cases, the framework makes it easy to integrate additional serialization tools or to develop
custom solutions for serializing data within your application. This allows yiut to choose the best approach for your
specific needs, while still taking advantage of the other features and benefits provided by the Spiral Framework.

The component is available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the serializer component in your Spiral Framework application, you need to add
`Spiral\Serializer\Bootloader\SerializerBootloader` to the bootloaders list.

```php
protected const LOAD = [
    // ...
    \Spiral\Serializer\Bootloader\SerializerBootloader::class,
    // ...
];
```

## Configuration

The configuration for the serializer component is stored in the `app/config/serializer.php` file. In this file, you can
configure an array of available serializers and specify the default serializer.

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
     * The key of one of the registered serializers.
     */
    'default' => 'json',
    
    /**
     * -------------------------------------------------------------------------
     *  Available serializers
     * -------------------------------------------------------------------------
     * List of available serializers.  
     */
    'serializers' => [
        // via fully qualified class name
        'json' => JsonSerializer::class,
        
        // via Autowire 
        'serializer' => new Autowire(PhpSerializer::class),
        
        // via Autowire with arguments
        'encrypted_serializer' => new Autowire(EncryptedPhpSerializer::class, ['secret' => env('ENCRYPTION_KEY')]),
        
        // or manual instantiating object
        'callback' => new CallbackSerializer(
            serializeCallback: fn(mixed $payload): string => \json_encode($payload),
            unserializeCallback: fn(string|\Stringable $payload, string|object|null $type = null) => \json_decode($payload, true)
        )
    ],
];
```

## Available serializers

The component comes with a few basic serializers out of the box, but it's also easy to integrate
additional serialization tools or develop custom solutions for your specific needs.

- `Spiral\Serializer\Serializer\JsonSerializer` - uses PHP functions `json_encode` and `json_decode`.
  It does not support data hydration to an object.
- `Spiral\Serializer\Serializer\PhpSerializer` - uses PHP functions `serialize` and `unserialize`.
- `Spiral\Serializer\Serializer\CallbackSerializer`- uses callbacks for serializing and deserializing data.

There are also packages that provide additional serializers:

- [Symfony serializer](https://github.com/spiral-packages/symfony-serializer) - uses the Symfony serializer component.
- [Laravel Serializable Closure](https://github.com/spiral-packages/serializable-closure) - uses the Laravel
  Serializable Closure package.

## Usage

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

You can use the `Spiral\Serializer\SerializerManager` to get a specific serializer by its string key from the
configuration.

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

To create a custom serializer in the Spiral Framework, you will need to implement the
`Spiral\Serializer\SerializerInterface`. This interface defines two methods that your serializer class must implement:
`serialize(...)` and `unserialize(...)`.

Here is an example of a custom serializer class that implements the `SerializerInterface`:

```php
namespace App;

use Spiral\Serializer\SerializerInterface;

class CustomSerializer implements SerializerInterface
{
    public function serialize(mixed $payload): string|\Stringable
    {
        // your serialization logic goes here
    }

    public function unserialize(\Stringable|string $payload, object|string|null $type = null): mixed
    {
        // your deserialization logic goes here
    }
}
```

The `serialize` method of your custom serializer class should accept a single parameter `$payload` which represents the
data to be serialized. This method should return the serialized data as a string.

The `unserialize` method of your custom serializer class should accept two parameters: `$payload` which represents the
serialized data as a string, and `$type` which is an optional parameter that specifies the name of the class or object
into which the payload should be deserialized. This method should return the deserialized data.

### Registering a new Serializer

There are two ways to register a custom serializer in the Spiral Framework:

1. Using the `Spiral\Serializer\SerializerRegistryInterface`:

```php
namespace App\Bootloader;

use App\CustomSerializer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Serializer\SerializerRegistryInterface;

class SerializerBootloader extends Bootloader
{
    public function boot(SerializerRegistryInterface $registry): void
    {
        $registry->register('my-serializer', new CustomSerializer());
    }
}
```

2. Using the config file:

```php
use App\Serializer;

return [
    // file app/config/serializer.php
    'serializers' => [
        'my-serializer' => CustomSerializer::class,
        // other serializers
    ],
];
```
