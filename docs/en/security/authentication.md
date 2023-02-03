# Security — User Authentication

The framework includes a set of components to authorize users via temporary or permanent tokens from different
sources and safely manages the user context.

> **Note**
> The component does not enforce any specific User entity interface and does not limit the application to HTTP scope
> only (GRPC auth is possible as well).

## Principle of Work

![Auth](https://user-images.githubusercontent.com/773481/210746599-cb43c8ad-8021-4c9a-8a9a-45eea55b4c22.png)

The authentication extension will create an IoC scope for `Spiral\Auth\AuthContextInterface` which points to the
currently authorized actor (User, API Client). The actor is fetched from `Spiral\Auth\ActorProviderInterface`
using `Spiral\Auth\TokenInterface`.

The token is managed by `Spiral\Auth\TokenStorageInterface` and always includes the payload (for
example `["userID" => $id]`, LDAP creds, etc.). The token payload is used to find current application user
via `Spiral\Auth\ActorProviderInterface`.

The token storage can either store a token in the external source (such as database, Redis, or file) or decode it on a
fly. The framework includes multiple token implementations out of the box for a more comfortable use.

> **Note**
> You can use multiple token and actor providers inside one application.

## Installation and Configuration

To install authorization extension for Web bundle:

```terminal
composer require spiral/auth spiral/auth-http
```

The package `spiral/auth` provides standard interfaces without the relation to any specific dispatching method, while
`spiral/auth-http` includes HTTP Middleware, Token transport (Cookie, Header), and Firewall components.

To activate the component add the bootloader `Spiral\Bootloader\Auth\HttpAuthBootloader`:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    // ...
]
```

## Token Storage

The `Spiral\Auth\TokenStorageInterface` is an interface that defines a set of methods for storing, retrieving and
deleting authentication tokens.

You can set the default storage for your application by using the `AUTH_TOKEN_STORAGE` environment variable.

Or define by using `defaultStorage` key in the `app/config/auth.php`:

```php app/config/auth.php
return [
    'defaultStorage' => env('AUTH_TOKEN_STORAGE', 'session'),
    // ... storages
]
```

This allows you to specify which implementation of the `TokenStorageInterface` should be retrieved from the container
by default.

```php
$storage = $container->get(\Spiral\Auth\TokenStorageInterface::class); 
// Will return a default Session token storage
```

### Session Token Storage

To store tokens in PHP session make sure that `spiral/session` extension is installed

```dotenv .env
AUTH_TOKEN_STORAGE=session
```

### Database Token Storage

The framework can store the token in the database via Cycle ORM. If you want to use this type of token you need to
install `spiral/cycle-bridge` package.

```terminal
composer require spiral/cycle-bridge
```

Activate `Spiral\Cycle\Bootloader\BridgeBootloader` for this purpose:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
    // ...
]
```

```dotenv .env
AUTH_TOKEN_STORAGE=database
```

> **See more**
> Read more about installation and configuration `spiral/cycle-bridge`
> package in the [The Basics — Database and ORM](../basics/orm.md) section.

You must generate and run database migration or run `cycle:sync` in order to create the needed table:

```terminal
php app.php migrate:init
php app.php cycle:migrate -v -r
```

### Custom Token Storage

Keep in mind that you can also create your own custom implementations of the `TokenStorageInterface` if the provided
implementations do not meet your needs. This allows you to use any storage mechanism that you choose for storing
authentication tokens in your application.

To create a custom storage in Spiral, you will need to create a class that implements
the `Spiral\Auth\TokenStorageInterface` interface.

Here is a simple example of a custom token storage implementation:

```php
use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Firebase\JWT\ExpiredException;
use Spiral\Auth\TokenInterface;
use Spiral\Auth\TokenStorageInterface;

final class JwtTokenStorage implements TokenStorageInterface
{
    /** @var callable */
    private $time;

    public function __construct(
        private readonly TokenEncoder $tokenEncoder,
        private readonly string $secret,
        private string $algorithm = 'HS256',
        private readonly string $expiresAt = '+30 days',
        callable $time = null
    ) {
        $this->tokenEncoder = $tokenEncoder;
        $this->expiresAt = $expiresAt;
        $this->time = $time ?? static function (string $offset): \DateTimeImmutable {
            return new \DateTimeImmutable($offset);
        };
    }

    public function load(string $id): ?TokenInterface
    {
        // ...
    }

    public function create(array $payload, \DateTimeInterface $expiresAt = null): TokenInterface
    {
        // ...
    }

    public function delete(TokenInterface $token): void
    {
        // ...
    }
}
```

> **Note**
> Full example of custom token storage can be found [here](../cookbook/user-authentication.md).

Spiral provides several ways to register a token storage.

:::: tabs

::: tab Bootloader
You will need to obtain an instance of `HttpAuthBootloader` and use its `addTokenStorage` method.
This method takes two arguments: a name for the storage and a class that implements
the `Spiral\Auth\TokenStorageInterface`.

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader;
use Spiral\Bootloader\Auth\HttpAuthBootloader;

final class AppBootloader extends Bootloader
{
    public function init(HttpAuthBootloader $httpAuth, JwtTokenStorage $storage): void 
    {
        $httpAuth->addTokenStorage('jwt', $storage);
    }
}
```
:::

::: tab Configuration
You can also register a token storage through the configuration file `app/config/auth.php`.

```php app/config/auth.php
use Spiral\Core\Container\Autowire;

return [
    // ...
    'storages' => [
         'session' => \Spiral\Auth\Session\TokenStorage::class,
         'jwt' => new Autowire(\App\JwtTokenStorage::class, [
             'secret' => 'secret', 
             'algorithm' => 'HS256',
             'expiresAt' => '+30 days',
         ]),
         // ...
    ]
]
```
:::

::::

## Token Storage provider

Token storage provider is a convenient way to retrieve a token storage by given name.

```php
$container->get(\Spiral\Auth\TokenStorageProviderInterface::class)
    ->getStorage('jwt');
```

## Usage with HTTP layer

### Middleware

There are three middleware classes that can be used to obtain the authentication token from the request and authenticate
the user based on the token.

- `Spiral\Auth\Middleware\AuthMiddleware` - obtains the token from the request with using default token storage and all
  available transports such as `cookies`, `headers`, `query parameters`, etc.
- `Spiral\Auth\Middleware\AuthTransportMiddleware` - obtains the token from the request with using default token storage
  and a specific transport.
- `Spiral\Auth\Middleware\AuthTransportWithStorageMiddleware` - obtains the token from the request with using a specific
  token storage and a specific transport.

The [application bundle](https://github.com/spiral/app) provides `App\Application\Bootloader\RoutesBootloader`, where
you can easily define middleware.

```php app/src/Application/Bootloader/RoutesBootloader.php
namespace App\Application\Bootloader;

use Spiral\Auth\Middleware\AuthMiddleware;
use Spiral\Auth\Middleware\AuthTransportMiddleware;
use Spiral\Auth\Middleware\AuthTransportWithStorageMiddleware;
use Spiral\Core\Container\Autowire;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                AuthMiddleware::class,
            ],
            'api' => [
                new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header']),
            ],
            'api_jwt' => [
                new Autowire(AuthTransportWithStorageMiddleware::class, ['transportName' => 'header', 'storage' => 'jwt']),
            ],
        ];
    }
    // ...
}
```

> **See more**
> Read more about middleware in the [HTTP — Middleware](../http/middleware.md) section.

### Token transport

In Spiral, the `Spiral\Auth\HttpTransportInterface` is used to read and write authentication tokens in
HTTP requests and responses using the PSR-7 interfaces.

This interface defines three methods:

- The `fetchToken` method is used to retrieve an authentication token from an incoming request. This could be from a
  cookie, a header, or any other location where the token might be stored.

- The `commitToken` method is used to write an authentication token to an outgoing response. This might involve setting
  a cookie, adding a header, or any other method of storing the token.

- The `removeToken` method is used to remove an authentication token from an outgoing response. This might involve
  unsetting a cookie or removing a header.

There are two transports available for use with the `HttpTransportInterface`:

- `cookie` stores the authentication token in a cookie. When the client makes a request to the server, the cookie is
  included in the request and can be used to identify the client.

- `header` stores the authentication token in an HTTP header `X-Auth-Token` by default.

You can set the default transport for your application by using the `AUTH_TOKEN_TRANSPORT` environment variable.

Or define by using `defaultTransport` key in the `app/config/auth.php`:

```php app/config/auth.php
return [
    'defaultTransport' => env('AUTH_TOKEN_TRANSPORT', 'cookie'),
    // ... storages
]
```

Spiral provides several ways to register a token transport.

:::: tabs

::: tab Bootloader
You will need to obtain an instance of `HttpAuthBootloader` and use its `addTransport` method.
This method takes two arguments: a name for the transport and a class that implements
the `Spiral\Auth\HttpTransportInterface`.

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader;
use Spiral\Bootloader\Auth\HttpAuthBootloader;
use Spiral\Auth\Transport\CookieTransport;
use Spiral\Auth\Transport\HeaderTransport;

final class AppBootloader extends Bootloader
{
    public function boot(HttpAuthBootloader $httpAuth): void 
    {
        $httpAuth->addTransport(
          'cookie', 
          new CookieTransport(cookie: 'token', basePath: '/')
        );

        $httpAuth->addTransport(
          'header', 
          new HeaderTransport(header: 'X-Auth-Token')
        );
    }
}
```
:::

::: tab Configuration
You can also register a token transport through the configuration file `app/config/auth.php`.

```php app/config/auth.php
use Spiral\Auth\Transport\CookieTransport;
use Spiral\Auth\Transport\HeaderTransport;

return [
    // ...
    'transports' => [
         'header' => new HeaderTransport(header: 'X-Auth-Token'),
         'cookie' => new CookieTransport(cookie: 'token', basePath: '/')
         // ...
    ]
]
```
:::

::::


## Actor Provider and Token Payload

The next step to configure a way to fetch actors/users is based on token payloads, we must implement and register
interface `Spiral\Auth\ActorProviderInterface` for these purposes.

```php
interface ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object;
}
```

For this article, we are going to use Cycle Entity and Repository:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: "string")]
    public string $name;

    #[Cycle\Column(type: "string")]
    public string $username;

    #[Cycle\Column(type: "string")]
    public string $password;
}
```

We can implement the interface in UserRepository:

```php
namespace App\Database\Repository;

use Cycle\ORM\Select\Repository;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\TokenInterface;

class UserRepository extends Repository implements ActorProviderInterface
{
    public function getActor(TokenInterface $token): ?object
    {
        if (!isset($token->getPayload()['userID'])) {
            return null;
        }

        return $this->findByPK($token->getPayload()['userID']);
    }
}
```

Once the migration is complete, we can create our first user:

```php
use Cycle\ORM\EntityManagerInterface;

public function index(EntityManagerInterface $entityManager)
{
    $user = new User();
    
    $user->name = 'Antony';
    $user->username = 'username';
    $user->password = \password_hash('password', PASSWORD_DEFAULT);

    $entityManager->persist($u)->run();
}
```

Register the actor provider to enable it, create and activate the Bootloader in your application:

```php app/src/Application/Bootloader/UserBootloader.php
namespace App\Application\Bootloader;

use App\Database\Repository\UserRepository;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Auth\AuthBootloader;

class UserBootloader extends Bootloader
{
    public function boot(AuthBootloader $auth): void
    {
        $auth->addActorProvider(UserRepository::class);
    }
}
```

### Logout

To log user out call the method `close` of auth context or AuthScope:

```php
public function logout(): void
{
    $this->auth->close();
}
```

## RBAC security

You can use an authenticated user as an actor for the RBAC security component, make sure to
implement `Spiral\Security\ActorInterface` in your `App\Database\User`:

```php
namespace App\Database;

use Spiral\Security\ActorInterface;
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User implements ActorInterface
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: "string")]
    public string $name;

    #[Cycle\Column(type: "string")]
    public string $username;

    #[Cycle\Column(type: "string")]
    public string $password;

    public function getRoles(): array
    {
        return ['user'];
    }
}
```

And activate the bootloader `Spiral\Bootloader\Auth\SecurityActorBootloader` to link two components together:

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Bootloader\Auth\SecurityActorBootloader::class,
    // ...
]
```

## Firewall Middleware

You can protect some of your route targets by attaching the firewall middleware to prevent unauthorized access.

Spiral provides the following firewall middlewares:

#### Firewall which will overwrite the target url

```php
use Spiral\Auth\Middleware\Firewall\OverwriteFirewall;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new OverwriteFirewall(new Uri('/account/login')));
```

#### Firewall which will throw an exceptions

```php
use Spiral\Auth\Middleware\Firewall\ExceptionFirewall;
use Spiral\Http\Exception\ClientException\ForbiddenException;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new ExceptionFirewall(new ForbiddenException()));
```

```php
use Spiral\Http\Exception\ClientException\RedirectFirewall;
use Psr\Http\Message\ResponseFactoryInterface;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new RedirectFirewall(
            uri: new Uri('/account/login'),
            status: 302,
            responseFactory: $container->get(ResponseFactoryInterface::class)
        ));
```

#### Firewall which will redirect to the target url

#### Custom firewall

To implement your firewall, extend `Spiral\Auth\Middleware\Firewall\AbstractFirewall`:

```php
final class CustomFirewall extends AbstractFirewall
{
    public function __construct(
        // args...
    ) {
    }

    protected function denyAccess(Request $request, RequestHandlerInterface $handler): ResponseInterface
    {
        // return response
    }
}
```

## Events

| Event                           | Description                                                     |
|---------------------------------|-----------------------------------------------------------------|
| Spiral\Auth\Event\Authenticated | The Event will be fired `after` the user authenticated success. |
| Spiral\Auth\Event\Logout        | The Event will be fired `after` the user logout success.        |

> **Note**
> To learn more about dispatching events, see the [Events](../advanced/events.md) section in our documentation.
