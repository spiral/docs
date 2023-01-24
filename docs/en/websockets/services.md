# Websockets — Services

In a Spiral application, a service is a class that is responsible for processing incoming messages, performing
actions based on the message content, and returning a response to the sender. There are several types of events that can
be handled by services:

- **Connection request**: When a client establishes a WebSocket connection to the Centrifugo server, a connection
  request event is sent to RoadRunner.
- **Refresh connection request**: Centrifugo sends a refresh event to RoadRunner when it is time to refresh the
  connection between the client and the server.
- **RPC call request**: Centrifugo can send RPC (Remote Procedure Call) events to RoadRunner over a client connection,
  allowing you to invoke server-side functions from the client.
- **Subscribe request**: When a client subscribes to a channel, Centrifugo sends a subscribe request event to
  RoadRunner.
- **Publish request**: This request occurs before a message is published to a channel, allowing your backend to validate
  whether a client can publish data to a channel.

## Create a service

To create a service, you will need to implement the `Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface`
interface and provide the `handle` method for incoming requests.

The `handle` method will receive a request object that implements the `RoadRunner\Centrifugo\Request\RequestInterface`
interface. The specific type of request object that is received will depend on the type of event being handled. For
example, a connection request event will be passed a `RoadRunner\Centrifugo\Request\Connect` object, while a subscribe
request event will be passed a `RoadRunner\Centrifugo\Request\Subscribe` object.

The request object has the `respond` method that should be used to send a response to the Centrifugo server. The
response data will be passed as an object, which implements `RoadRunner\Centrifugo\Payload\ResponseInterface`.

To register a service, you will need to specify the event type and the class name of the service. The
event type is specified using the `RoadRunner\Centrifugo\Request\RequestType` enum, which has constants for each
supported event type.

### Service registration

Here is an example of how to register an event handler service in the `app/config/centrifugo.php` configuration file:

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;
use App\Centrifuge;

return [
    'services' => [
        RequestType::Connect->value => Centrifuge\ConnectService::class,
        //...
    ],
    'interceptors' => [
        //...
    ],
];
```

> **Note**
> For more information on interceptors and how to use them in a Spiral application, you can refer to
> the documentation section on [Interceptors](./interceptors.md). This page provides additional details and examples to
> help you get started with Interceptors.

### Connection request

This service receives a `RoadRunner\Centrifugo\Request\Connect` object and performs some action based on the connection
request. It should respond to the Centrifugo server with `RoadRunner\Centrifugo\Payload\ConnectResponse` object.

#### Simple example

Here is an example of a service that accepts all connection requests:

```php app/src/Interfaces/Centrifuge/ConnectService.php
namespace App\Interfaces\Centrifuge;

use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

class ConnectService implements ServiceInterface
{
    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                // Return an empty string for accepting unauthenticated requests
                new ConnectResponse(user: '')
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

The Centrifugo JavaScript SDK allows you to pass additional data to the server when connecting via WebSocket. This data
can be used for a variety of purposes, such as client authentication.

#### Service that accepts only authenticated requests

Here is an example of a service that accepts only authenticated connection requests:

```php app/src/Interfaces/Centrifuge/ConnectService.php
namespace App\Interfaces\Centrifuge;

use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;
use Spiral\Auth\TokenStorageInterface;

class ConnectService implements ServiceInterface
{
    public function __construct(
        private readonly TokenStorageInterface $tokenStorage,
    ) {
    }
    
    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $userId = null;
            
            // Authenticate user with a given token from the connection request
            $authToken = $request->getData()['authToken'] ?? null;
            if ($authToken && $token = $this->tokenStorage->load($authToken)) {
                $userId = $token->getPayload()['userID'] ?? null;
            }
            
            // You can also disconnect connection
            if (!$userId) {
                $request->disconnect('403', 'Connection is not allowed.');
                return;
            }
        
            $request->respond(
                new ConnectResponse(user: (string) $userId)
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **Note**
> Read more about connection requests in
> the [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#connect-request-fields).


Here is an example of how to pass an `authToken` for client authentication using the Centrifugo JavaScript SDK:

```javascript
import {Centrifuge} from 'centrifuge';

const centrifuge = new Centrifuge('http://127.0.0.18000/connection/websocket', {
    data: {
        authToken: 'my-app-auth-token'
    }
});
```

> **Note**
> For more information on using JavaScript SDK and passing additional data to the server, refer to the
> [documentation](https://github.com/centrifugal/centrifuge-js#data).

### Subscribe request

This service receives a `RoadRunner\Centrifugo\Request\Subscribe` object and performs some action based on the
connection
request. It should respond to the Centrifugo server with `RoadRunner\Centrifugo\Payload\SubscribeResponse` object.

In this example, we will create a service that will allow users to subscribe to channels only if they are authenticated
with rules provided by the `Spiral\Broadcasting\TopicRegistryInterface` interface.

```php app/src/Interfaces/Centrifuge/SubscribeService.php
namespace App\Interfaces\Centrifuge;

use RoadRunner\Centrifugo\Payload\SubscribeResponse;
use RoadRunner\Centrifugo\Request\Subscribe;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;
use Spiral\Broadcasting\TopicRegistryInterface;

final class SubscribeService implements ServiceInterface
{
    public function __construct(
        private readonly InvokerInterface $invoker,
        private readonly ScopeInterface $scope,
        private readonly TopicRegistryInterface $topics,
    ) {
    }

    /**
     * @param Subscribe $request
     */
    public function handle(RequestInterface $request): void
    {
        try {
            if (!$this->authorizeTopic($request->channel, $request->user)) {
                $request->disconnect('403', 'Channel is not allowed.');
                return;
            }
        
            $request->respond(
                new SubscribeResponse()
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
    
    private function authorizeTopic(Subscribe $request): bool
    {
        $parameters = [];
        $callback = $this->topics->findCallback($request->channel, $parameters);
        if ($callback === null) {
            return false;
        }

        return $this->invoke($request, $callback, $parameters + ['topic' => $topic, 'userId' => $request->user]);
    }

    private function invoke(Subscribe $request, callable $callback, array $parameters = []): bool
    {
        return $this->scope->runScope(
            [
                RequestInterface::class => $request,
            ],
            fn (): bool => $this->invoker->invoke($callback, $parameters)
        );
    }
}
```

You can register channel authorization rules in the `app/config/broadcasting.php` file:

```php app/config/broadcasting.php
use RoadRunner\Centrifugo\Request\Subscribe;
'authorize' => [
    'topics' => [
        'topic' => static fn (Subscribe $request): bool => $request->getHeader('SECRET')[0] == 'secret',
        'user.{uuid}' => static fn (string $uuid, string $userId): bool => $userId === $uuid
    ],
],
```

> **Note**
> Read more about subscribe requests in
> the [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#subscribe-request-fields).

### Refresh connection request

This service receives a `RoadRunner\Centrifugo\Request\Refresh` object and performs some action based on the connection
request. It should respond to the Centrifugo server with `RoadRunner\Centrifugo\Payload\RefreshResponse` object.

Here is an example of a service:

```php app/src/Interfaces/Centrifuge/RefreshService.php
namespace App\Interfaces\Centrifuge;

use RoadRunner\Centrifugo\Request\Refresh;
use RoadRunner\Centrifugo\Payload\RefreshResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

class RefreshService implements ServiceInterface
{
    /** @param Refresh $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                new RefreshResponse(...)
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **Note**
> Read more about refresh connection requests in
> the [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#refresh-request-fields).

### RPC call request

This service receives a `RoadRunner\Centrifugo\Request\RPC` object and performs some action based on the connection
request. It should respond to the Centrifugo server with `RoadRunner\Centrifugo\Payload\RPCResponse` object.

#### Simple example

Here is an example of a service that receives a `ping` RPC call and responds with `pong`:

```php app/src/Interfaces/Centrifuge/RPCService.php
namespace App\Interfaces\Centrifuge;

use RoadRunner\Centrifugo\Payload\RPCResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use RoadRunner\Centrifugo\Request\RPC;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class RPCService implements ServiceInterface
{
    /**
     * @param RPC $request
     */
    public function handle(RequestInterface $request): void
    {
        $result = match ($request->method) {
            'ping' => 'pong',
            default => ['error' => 'Not found', 'code' => 404]
        };

        try {
            $request->respond(
                new RPCResponse(
                    data: $result
                )
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **Note**
> Read more about RPC requests in
> the [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#rpc-request-fields).

#### Proxy RPC methods to the HTTP layer

Here is an example of how to proxy RPC methods to the HTTP layer and return the result to the Centrifugo server:

```php app/src/Interfaces/Centrifuge/RPCService.php
namespace App\Interfaces\Centrifuge;

use Psr\Http\Message\ServerRequestFactoryInterface;
use Psr\Http\Message\ServerRequestInterface;
use RoadRunner\Centrifugo\Payload\RPCResponse;
use RoadRunner\Centrifugo\Request\RPC;
use Spiral\Filters\Exception\ValidationException;
use Spiral\Http\Http;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class RPCService implements ServiceInterface
{
    public function __construct(
        private readonly Http $http,
        private readonly ServerRequestFactoryInterface $requestFactory,
    ) {
    }

    /**
     * @param RPC $request
     */
    public function handle(Request\RequestInterface $request): void
    {
        try {
            $response = $this->http->handle($this->createHttpRequest($request));

            $result = \json_decode((string)$response->getBody(), true);
            $result['code'] = $response->getStatusCode();
        } catch (ValidationException $e) {
            $result['code'] = $e->getCode();
            $result['errors'] = $e->errors;
            $result['message'] = $e->getMessage();
        } catch (\Throwable $e) {
            $result['code'] = $e->getCode();
            $result['message'] = $e->getMessage();
        }

        try {
            $request->respond(new RPCResponse(data: $result));
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }

    public function createHttpRequest(Request\RPC $request): ServerRequestInterface
    {
        if(!\str_contains($request->method, ':')) {
            throw new \InvalidArgumentException('Method must be in format "method:uri"');
        }

        [$controller, $action] = \explode('.', $request->method, 2);
        
        // Example of method string: get:users/1 , post:news/store, delete:news/1
        // split received method string to HTTP method and uri  
        [$method, $uri] = \explode(':', $request->method, 2);
        
        $method = \strtoupper($method);

        $httpRequest = $this->requestFactory->createServerRequest($method, \ltrim($uri, '/'))
            ->withHeader('Content-Type', 'application/json');

        return match ($method) {
            'GET', 'HEAD' => $httpRequest->withQueryParams($request->getData()),
            'POST', 'PUT', 'DELETE' => $httpRequest->withParsedBody($request->getData()),
            default => throw new \InvalidArgumentException('Unsupported method'),
        };
    }
}
```

And an example of how to use in JavaScript side:

```javascript
import {Centrifuge} from 'centrifuge';

const centrifuge = new Centrifuge('http://127.0.0.18000/connection/websocket');

// Post request
centrifuge.rpc("post:news/store", {"title": "News title"}).then(function (res) {
    console.log('rpc result', res);
}, function (err) {
    console.error('rpc error', err);
});

// Get request with query params
centrifuge.rpc("get:news/123", {"lang": "en"}).then(function (res) {
    console.log('rpc result', res);
}, function (err) {
    console.error('rpc error', err);
});
```

> **Note**
> For more information on using JavaScript SDK and RPC method, refer to the
> [documentation](https://github.com/centrifugal/centrifuge-js#rpc-method).

### Publish request

This service receives a `RoadRunner\Centrifugo\Request\Publish` object and performs some action based on the connection
request. It should respond to the Centrifugo server with `RoadRunner\Centrifugo\Payload\PublishResponse` object.

```php app/src/Interfaces/Centrifuge/PublishService.php
namespace App\Interfaces\Centrifuge;


use RoadRunner\Centrifugo\Payload\PublishResponse;
use RoadRunner\Centrifugo\Request;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class PublishService implements ServiceInterface
{
    /**
     * @param Request\Publish $request
     */
    public function handle(RequestInterface $request): void
    {
        try {
            $request->respond(
                new PublishResponse(...)
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

> **Note**
> Read more about publish requests in
> the [Centrifugo documentation](https://centrifugal.dev/docs/server/proxy#publish-request-fields).
