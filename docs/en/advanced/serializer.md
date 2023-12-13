# Component — Serializer

Spiral helps you manage and organize your code. One aspect of this is the ability to serialize data, which
involves converting it into a format that can be stored or transmitted, and then later deserialized, or converted back
into its original form. This is useful in a variety of situations, such as when transferring data over a network or
storing it in a database.

Spiral includes some basic serialization tools by default, but these may not always be sufficient for more
complex use cases. In such cases, the framework makes it easy to integrate additional serialization tools or to develop
custom solutions for serializing data within your application. This allows you to choose the best approach for your
specific needs.

The component is available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the serializer component in your Spiral application, you need to add
`Spiral\Serializer\Bootloader\SerializerBootloader` to the bootloaders list.

```php app/src/Application/Kernel.php
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

```php app/config/serializer.php
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

To create a custom serializer in Spiral, you will need to implement the `Spiral\Serializer\SerializerInterface`. This 
interface defines two methods that your serializer class must implement:
`serialize(...)` and `unserialize(...)`.

Here is an example of a custom serializer class that implements the `SerializerInterface`:

```php
<?php

declare(strict_types=1);

namespace App\Application\Serializer;

use Google\Protobuf\Internal\Message;
use Spiral\Serializer\SerializerInterface;

final class ProtoSerializer implements SerializerInterface
{
    public function serialize(mixed $payload): string|\Stringable
    {
        \assert($payload instanceof Message);

        return $payload->serializeToString();
    }

    public function unserialize(\Stringable|string $payload, object|string|null $type = null): mixed
    {
        \assert(
            $type !== null
            && \class_exists($type)
            && \is_a($type, Message::class, true),
        );

        $object = new $type();
        $object->mergeFromString((string)$payload);

        return $object;
    }
}

```

The `serialize` method of your custom serializer class should accept a single parameter `$payload` which represents the
data to be serialized. This method should return the serialized data as a string.

The `unserialize` method of your custom serializer class should accept two parameters: `$payload` which represents the
serialized data as a string, and `$type` which is an optional parameter that specifies the name of the class or object
into which the payload should be deserialized. This method should return the deserialized data.

### Registering a new Serializer

There are two ways to register a custom serializer:

:::: tabs
::: tab Registry
Using the `Spiral\Serializer\SerializerRegistryInterface`:

```php
namespace App\Application\Bootloader;

use App\Application\Serializer\ProtoSerializer;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Serializer\SerializerRegistryInterface;

class SerializerBootloader extends Bootloader
{
    public function boot(SerializerRegistryInterface $registry): void
    {
        $registry->register('proto', new ProtoSerializer());
    }
}
```
:::

::: tab Config
Using the config file:

```php app/config/serializer.php
use App\Application\Serializer\ProtoSerializer;

return [
    'serializers' => [
        'proto' => ProtoSerializer::class,
        // other serializers
    ],
];
```
:::
::::