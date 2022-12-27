# Event Broadcasting

Being able to broadcast data in real-time from servers to clients is a requirement for many modern web and mobile 
applications. When some data is updated on the server, a message is typically sent over a WebSocket connection to be 
handled by the client. WebSockets provide a more efficient alternative to continually poll your application's server 
for data changes that should be reflected in your UI.

The component is available by default in the [application bundle](https://github.com/spiral/app).

## Installation

To enable the component, you just need to add `Spiral\Broadcasting\Bootloader\BroadcastingBootloader` and 
`Spiral\Broadcasting\Bootloader\WebsocketsBootloader` classes to the bootloaders list, which is located in the 
class of your application.

```php
namespace App;

use Spiral\Broadcasting\Bootloader\BroadcastingBootloader;
use Spiral\Broadcasting\Bootloader\WebsocketsBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        BroadcastingBootloader::class,
        WebsocketsBootloader::class,
        // ...
    ];
}
```

The `BroadcastingBootloader` bootloader registers required services for event broadcasting. The `WebsocketsBootloader` 
registers `Spiral\Broadcasting\Middleware\AuthorizationMiddleware`.

### Supported Drivers

The Broadcasting component requires a server that will receive events from the application. By default, the Broadcasting
component supports `RoadRunner WebSockets`. The process of configuring the Broadcasting driver is described on the
[RoadRunner Bridge documentation page](https://github.com/spiral/roadrunner-bridge).

## Configuration

You can create a config file `app/config/broadcasting.php` if you want to configure Broadcasting drivers.

```php
use Psr\Log\LogLevel;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Broadcasting\Driver\LogBroadcast;
use Spiral\Broadcasting\Driver\NullBroadcast;

return [
    /**
     * -------------------------------------------------------------------------
     *  Default connection
     * -------------------------------------------------------------------------
     * 
     * Events will be sent by default to this connection.
     */
    'default' => env('BROADCAST_CONNECTION', 'null'),
    
    /**
     * -------------------------------------------------------------------------
     *  Authorize settings
     * -------------------------------------------------------------------------
     */
    'authorize' => [
        'path' => env('BROADCAST_AUTHORIZE_PATH'),
        'topics' => [
            'topic' => static fn (ServerRequestInterface $request): bool => $request->getHeader('SECRET')[0] == 'secret',
            'user.{id}' => static fn ($id, Actor $actor): bool => $actor->getId() === $id
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Connections
     * -------------------------------------------------------------------------
     *  
     * List of available connections to send events.
     */
    'connections' => [
        'null' => [
            'driver' => 'null',
        ],
        'log' => [
            'driver' => 'log',
            'level' => LogLevel::INFO,
        ],
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Aliases
     * -------------------------------------------------------------------------
     */
    'aliases' => [
        'log' => 'alias'
    ],
    
    /**
     * -------------------------------------------------------------------------
     *  Driver aliases
     * -------------------------------------------------------------------------
     */
    'driverAliases' => [
        'null' => NullBroadcast::class,
        'log' => LogBroadcast::class,
    ],
];
```

### TopicRegistry

Using `Spiral\Broadcasting\TopicRegistryInterface`, you can add a new `topic`. To do this, you need to use the `register`
method and pass the `$topic` and the `$callback` function in the parameters.

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Broadcasting\TopicRegistryInterface;

class AppBootloader extends Bootloader
{
    public function boot(TopicRegistryInterface $registry): void
    {
        $registry->register('baz-topic', fn(): string => 'baz');
        $registry->register('baz.{uuid}', fn(string $uuid): string => $uuid);
    }
}
```

Topics from the configuration file are added to the `TopicRegistry` automatically.

## Usage

### Default connection

You can use `Spiral\Broadcasting\BroadcastInterface` to send an event to the default connection.

```php
namespace App;

use Spiral\Broadcasting\BroadcastInterface;
use Spiral\Serializer\SerializerInterface;

class SomeClass
{
    public function __construct(
        private readonly BroadcastInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(): void
    {
        $this->broadcast->publish(
            'baz-topic',
            $this->serializer->serialize(['foo' => 'bar'])
        );
    }
}
```

### BroadcastManager

Using the `Spiral\Broadcasting\BroadcastManagerInterface`, you can send an event to a selected connection.

```php
namespace App;

use Spiral\Broadcasting\BroadcastManagerInterface;
use Spiral\Serializer\SerializerInterface;

class SomeClass
{
    public function __construct(
        private readonly BroadcastManagerInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(): void
    {
        $this->broadcast
            ->connection('log')
            ->publish(
                'baz-topic', 
                $this->serializer->serialize(['foo' => 'bar'])
            );
    }
}
```

### Object as a topic

The `publish` method can accept a `\Stringable` type, this allows us to create a topic via object.

```php
namespace App\Broadcast\Topic;

class User implements \Stringable
{
    public function __construct(
        public readonly int $userId
    ) {
    }

    public function __toString(): string
    {
        return \sprintf('user.%s', $this->userId);
    }
}
```

After that, you can use this object as a topic.

```php
namespace App\Service;

use App\Broadcast\Topic\User;
use Spiral\Auth\AuthContext;
use Spiral\Broadcasting\BroadcastInterface;

class SomeService
{
    public function __construct(
        private readonly AuthContext $auth,
        private readonly BroadcastInterface $broadcast
    ) {
    }

    public function method(): void
    {
        $userId = $this->auth->getActor()->getId();

        $this->broadcast->publish(new User($userId), 'some-message');
    }
}
```

## GuardInterface

A driver that implements `Spiral\Broadcasting\BroadcastInterface` can implement the `Spiral\Broadcasting\GuardInterface` 
interface and implement the `authorize` method. This method will be called by the `Spiral\Broadcasting\Middleware\AuthorizationMiddleware` 
and used to authorize the connection.

```php
namespace App;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Broadcasting\AuthorizationStatus;
use Spiral\Broadcasting\Driver\AbstractBroadcast;
use Spiral\Broadcasting\GuardInterface;

final class Broadcast extends AbstractBroadcast implements GuardInterface
{
    public function publish(iterable|string|\Stringable $topics, iterable|string $messages): void
    {
        // ...
    }

    public function join(iterable|string|\Stringable $topics): TopicInterface
    {
        // ...
    }

    public function authorize(ServerRequestInterface $request): AuthorizationStatus
    {
        // ...
    }
}
```

> **Note**
> This functionality is already implemented in `RoadRunner Bridge`.

## Events

| Event                                | Description                                            |
|--------------------------------------|--------------------------------------------------------|
| Spiral\Broadcasting\Event\Authorized | The Event will be fired `after` success authorization. |

> **Note**
> To learn more about dispatching events, see the [Events](../component/events.md) section in our documentation.
