# Websockets — Interceptors

Spiral provides interceptors for WebSockets services that allow you to intercept and modify requests and
responses at various points in the request lifecycle.

> **See more**
> Read more about interceptors in the [Framework — Interceptors](../framework/interceptors.md) section.

They are typically used to add cross-cutting functionality such as logging, authentication, or monitoring to the server.

## Example

### Authentication interceptor

The following example shows how to create an interceptor that checks the user's authentication token and provides the 
user's identity to the service. Where `authToken` is the name of the field in the request data that contains the 
authentication token.

```php app/src/Entrypoint/Centrifugo/Interceptor/AuthenticatorInterceptor.php
namespace App\Entrypoint\Centrifugo\Interceptor;

use Psr\EventDispatcher\EventDispatcherInterface;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\AuthContext;
use Spiral\Auth\AuthContextInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\ScopeInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class AuthenticatorInterceptor implements CoreInterceptorInterface
{
    use PrototypeTrait;

    public function __construct(
        private readonly ScopeInterface $scope,
        private readonly ActorProviderInterface $actorProvider,
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $request = $parameters['request'];
        \assert($request instanceof RequestInterface);

        $authToken = $request->getData()['authToken'] ?? null;

        if (!$authToken || !$token = $this->authTokens->load($authToken)) {
            $request->error(403, 'Unauthorized');
            return null;
        }

        $auth = new AuthContext($this->actorProvider, $this->eventDispatcher);
        $auth->start($token);

        return $this->scope->runScope([
            AuthContextInterface::class => $auth,
        ], fn () => $core->callAction($controller, $action, $parameters));
    }
}
```

And example of how to use it in a service:

```php app/src/Entrypoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;

use App\Database\User;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class ConnectService implements ServiceInterface
{
    use PrototypeTrait;

    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            $user = $this->auth->getActor();

            $request->respond(
                new ConnectResponse(
                    user: (string)$user->getId(),
                    data: ['user' => $user->jsonSerialize()],
                    channels: ['chat'],
                ),
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

### Error handling interceptor

The following example shows how to create an interceptor that handles errors.

```php app/src/Entrypoint/Centrifugo/Interceptor/ExceptionHandlerInterceptor.php
namespace App\Entrypoint\Centrifugo\Interceptor;

use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Exceptions\ExceptionReporterInterface;
use Spiral\RoadRunner\GRPC\Exception\GRPCException;
use Spiral\RoadRunner\GRPC\Exception\GRPCExceptionInterface;

final class ExceptionHandlerInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly ExceptionReporterInterface $reporter
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        try {
            \assert($parameters['request'] instanceof RequestInterface);

            return $core->callAction($controller, $action, $parameters);
        } catch (\Throwable $e) {
            $this->reporter->report($e);

            $request->error($e->getCode(), $e->getMessage());
        }
    }
}
```

After that, you don't need to code into try/catch blocks in your services:

```php app/src/Entrypoint/Centrifugo/ConnectService.php
namespace App\Endpoint\Centrifugo;
/**
 * @param Connect $request
 */
public function handle(RequestInterface $request): void
{
    if (!$this->auth->isAuthenticated()) {
        thorw new \Exception('Unauthorized', 403);
    }
    
    $request->respond(
        ...
    );
}
```

## Registering interceptors

To use this interceptor, you will need to register them in the configuration file `app/config/centrifugo.php`.

```php app/config/centrifugo.php
use RoadRunner\Centrifugo\Request\RequestType;
use App\Centrifuge;

return [
    'services' => [
        //...
    ],
    'interceptors' => [
        RequestType::Connect->value => [
            Centrifuge\Interceptor\AuthInterceptor::class,
        ],
        //...
        '*' => [
            Centrifuge\Interceptor\ExceptionHandlerInterceptor::class,
            Centrifuge\Interceptor\TelemetryInterceptor::class,
        ],
    ],
];
```

You can register interceptors for specific requests or for all requests using the `*` wildcard.

