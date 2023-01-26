# Websockets — Event broadcasting

Being able to broadcast data in real-time from servers to clients is a requirement for many modern web and mobile
applications. When some data is updated on the server, a message is typically sent over a WebSocket connection to be
handled by the client. WebSockets provide a more efficient alternative to continually poll your application's server for
data changes that should be reflected in your UI.

## Installation

To enable the component, you just need to add `Spiral\Broadcasting\Bootloader\BroadcastingBootloader` class to the
bootloaders list, which is located in the class of your application.

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Broadcasting\Bootloader\BroadcastingBootloader::class,
    // ...
];
```

## Configuration

The configuration for broadcasting in the Spiral Framework is stored in the `app/config/broadcasting.php` file. The
framework includes several built-in broadcast drivers, including a `log` driver for local development and debugging,
and a `null` driver for disabling broadcasting during testing.

The default driver can be changed using the `BROADCAST_CONNECTION` environment variable:

```dotenv .env
# Broadcasting
BROADCAST_CONNECTION=log
```

or by modifying the `default` setting in the configuration file.

```php app/config/broadcasting.php
use Spiral\Broadcasting\Driver\LogBroadcast;
use Spiral\Broadcasting\Driver\NullBroadcast;

return [
    'default' => 'log',
    'authorize' => [],
    'aliases' => [],
    'connections' => [
        'log' => [
            'driver' => 'log',
        ],
        'null' => [
            'driver' => NullBroadcast::class,
        ],
    ],
    'driverAliases' => [
        'log' => LogBroadcast::class,
    ],
];
```

## Usage

The Spiral Framework provides the `Spiral\Broadcasting\BroadcastInterface` interface for sending events to the default
connection using the `publish method`.

```php
use Spiral\Broadcasting\BroadcastInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function purchase(string $orderUuid): void
    {
        // ...

        $this->broadcast->publish(
            "order.{$orderUuid}",
            $this->serializer->serialize(['status' => 'purchased'])
        );
    }
}
```

In some cases, you may want to send an event to a specific broadcasting driver rather than the default connection. The
Spiral Framework provides the `Spiral\Broadcasting\BroadcastManagerInterface` for this purpose. It allows you to obtain
a connection instance for a specific driver and use it to publish events.

```php
use Spiral\Broadcasting\BroadcastManagerInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastManagerInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(string $orderUuid): void
    {
        $this->broadcast
            ->connection('log')
            ->publish(
               "order.{$orderUuid}",
                $this->serializer->serialize(['status' => 'purchased'])
            );
    }
}
```

The `publish` method of the `BroadcastInterface` can also accept a `\Stringable` type as the topic argument. This allows
you to create a topic using an object that implements it.

```php
namespace App\Broadcast\Topic;

final class Order implements \Stringable
{
    public function __construct(
        public readonly string $orderUuid
    ) {
    }

    public function __toString(): string
    {
        return \sprintf('order.%s', $this->orderUuid);
    }
}
```

Here is an example of how the `Order` class could be used with the `publish` method:

```php
use Spiral\Broadcasting\BroadcastManagerInterface;
use Spiral\Serializer\SerializerInterface;

class OrderService
{
    public function __construct(
        private readonly BroadcastManagerInterface $broadcast,
        private readonly SerializerInterface $serializer
    ) {
    }

    public function send(Order $order): void
    {
        $this->broadcast
            ->connection('log')
            ->publish(
               $order,
               $this->serializer->serialize(['status' => 'purchased'])
            );
    }
}
```

## Drivers

### Centrifugo

> **See more**
> Read more about integration with Centrifugo Websocket server [here](./configuration.md). 

After installation, you can activate driver using the `BROADCAST_CONNECTION` environment variable:

```dotenv .env
# Broadcasting
BROADCAST_CONNECTION=centrifugo
```

### Custom driver

You can create your own driver by implementing the `Spiral\Broadcasting\BroadcastInterface` interface. The driver should
implement the `publish` method, which accepts a topic and a payload.

```php
namespace App\Broadcast;

use Pusher\Pusher;
use Spiral\Broadcasting\Driver\AbstractBroadcast;

final class PusherBroadcast extends AbstractBroadcast
{ 
    public function __construct(
        private readonly Pusher $pusher
    ){
    }
    
    public function publish(iterable|string|\Stringable $topics, iterable|string $messages): void
    {
        $topics = $this->formatTopics($this->toArray($topics));
        
        /** @var string $message */
        foreach ($this->toArray($messages) as $message) {
            \assert(\is_string($message), 'Message argument must be a type of string');

            $this->pusher->trigger($topics, $message, []);
        }
    }
}
```

And then you can register it in the `app/config/broadcasting.php` file.

```php app/config/broadcasting.php
use Spiral\Broadcasting\Driver\LogBroadcast;
use Spiral\Broadcasting\Driver\NullBroadcast;
use App\Broadcast\PusherBroadcast;

return [
    'default' => 'log',
    'connections' => [
        'log' => [
            'driver' => 'log',
        ],
        'pusher' => [
            'driver' => 'pusher',
        ],
        'null' => [
            'driver' => NullBroadcast::class,
        ],
    ],
    'driverAliases' => [
        'log' => LogBroadcast::class,
        'pusher' => PusherBroadcast::class,
    ],
];
```

If you have some topics in your application that require authorization, you can use
the `Spiral\Broadcasting\GuardInterface` to implement authorization checks. The `GuardInterface` defines a single
method, `authorize`, which takes a `Psr\Http\Message\ServerRequestInterface`instance as an argument and returns
an `Spiral\Broadcasting\AuthorizationStatus` instance.

```php
use Pusher\Pusher;
use Spiral\Broadcasting\AuthorizationStatus;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\InvokerInterface;
use Spiral\Core\ScopeInterface;
use Spiral\Broadcasting\TopicRegistryInterface;

final class PusherBroadcaster extends AbstractBroadcast implements \Spiral\Broadcasting\GuardInterface
{
    public function __construct(
        private readonly Pusher $pusher,
        private readonly InvokerInterface $invoker,
        private readonly ScopeInterface $scope,
        private readonly TopicRegistryInterface $topics
    ) {
    }

    // ...
    
    public function authorize(ServerRequestInterface $request): AuthorizationStatus
    {
        $topic = $request->getQueryParams()['channel_name'] ?? null;
        
        if (!\is_string($topic)) {
            return new AuthorizationStatus(false, []);
        }
        
        if (!$this->authorizeTopic($request, $topic)) {
            return new AuthorizationStatus(false, [$topic]);
        }

        return new AuthorizationStatus(true, [$topic]);
    }
    
    
    private function authorizeTopic(ServerRequestInterface $request, string $topic): bool
    {
        $parameters = [];
        $callback = $this->topics->findCallback($topic, $parameters);
        if ($callback === null) {
            return false;
        }

        return $this->invoke($request, $callback, $parameters + ['topic' => $topic]);
    }

    private function invoke(ServerRequestInterface $request, callable $callback, array $parameters = []): bool
    {
        return $this->scope->runScope(
            [
                ServerRequestInterface::class => $request,
            ],
            fn (): bool => $this->invoker->invoke($callback, $parameters)
        );
    }
}
```

The `Spiral\Broadcasting\TopicRegistryInterface` is a central place for registering authorization rules for specific
topics in the Spiral Framework's broadcasting component. You can register authorization rules using
the `app/config/broadcasting.php` file.

```php app/config/broadcasting.php
'authorize' => [
    'path' => env('BROADCAST_AUTHORIZE_PATH'),
    'topics' => [ // <===============
        'topic' => static fn (ServerRequestInterface $request): bool => $request->getHeader('SECRET')[0] == 'secret',
        'order.{uuid}' => static fn (string $uuid, Actor $actor): bool => $actor->getId() === $id
    ],
],
```

Many websocket servers use HTTP authorization, which requires the client to provide credentials in the form of an HTTP
header or query parameter. To handle HTTP authorization in the Spiral Framework's broadcasting component, you can use
the `Spiral\Broadcasting\Middleware\AuthorizationMiddleware` middleware and specify an authorization endpoint in your
application where the websocket server will send authorization requests.


The `AuthorizationMiddleware` middleware is responsible for verifying the client's credentials and returning an
appropriate response indicating whether the client is authorized to access the websocket server. To use the middleware,
you will need to [register](../http/routing.md#middleware) it in your application and specify the authorization endpoint 
in the `app/config/broadcasting.php` configuration file.

```php app/config/broadcasting.php
'authorize' => [
    'path' => '/pusher/user-auth', // <===============
    'topics' => [
        'topic' => static fn (ServerRequestInterface $request): bool => $request->getHeader('SECRET')[0] == 'secret',
        'order.{uuid}' => static fn (string $uuid, Actor $actor): bool => $actor->getId() === $id
    ],
],
```

or using the `BROADCAST_AUTHORIZE_PATH` environment variable.

```dotenv .env
# Broadcasting
BROADCAST_AUTHORIZE_PATH=/pusher/user-auth
```