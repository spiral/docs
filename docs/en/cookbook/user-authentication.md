# Advanced - User authentication based on JWT token storage

The Spiral Framework is designed to be flexible and allow developers to use a variety of authentication storage
mechanisms. In addition to traditional session-based authentication and database-based authentication, the framework
also allows to use authentication methods like [JSON Web Tokens (JWT)](https://jwt.io/).

Using JWT for authentication allows developers to store user authentication information in a compact, self-contained
token that can be easily transmitted between the client and the server. This can be particularly useful in situations
where the client and server are distributed across different environments or networks, as it allows for secure and
efficient communication without the need for additional infrastructure or dependencies.

The Spiral Framework provides a number of interfaces and classes for implementing user authentication in an application.
For example, the `Spiral\Auth\TokenStorageInterface` can be used to generate and store tokens for authenticated users,
while the `Spiral\Auth\HttpTransportInterface` can be used to send or receive tokens to clients.

> **Note:**
> Read more about the interfaces in the [Authentication](../security/authentication.md) section.

Before you can implement user authentication in an application, you will need to add
the `Spiral\Auth\Middleware\AuthMiddleware` to the middleware list. This middleware is responsible for checking the
incoming request for a valid authentication token and loading the user's authentication information from the token
storage if one is found.

> **Note:**
> Read more about the middleware купшыекфешщт in the [Route](../http/routing.md#middleware) section.

When the `AuthMiddleware` loads an authentication token from the incoming request, it creates a new scope and binds an
instance of the `Spiral\Auth\AuthContext` to the `Spiral\Auth\AuthContextInterface` in the scoped container. This
allows you to access the `AuthContext` and its associated authentication information (such as the user's ID)
throughout the current request by injecting the `AuthContextInterface` into your controllers or other components.

## Implementing a JWT token storage

In the case of using a JWT for authentication, you will need to implement the `JWTTokenStorage` and defile it as the
default token storage.

Here is an example of how you can implement the `JWTTokenStorage` class:

```php
namespace App\Auth\Storage;

use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Firebase\JWT\ExpiredException;
use Spiral\Auth\TokenInterface;
use Spiral\Auth\TokenStorageInterface;

final class JWTTokenStorage implements TokenStorageInterface
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
        try {
            $token = (array) JWT::decode($id, new Key($this->secret, $this->algorithm));
        } catch (ExpiredException $exception) {
            throw $exception;
        } catch (\Throwable $exception) {
            return null;
        }

        if (
            false === isset($token['data'])
            || false === isset($token['iat'])
            || false === isset($token['exp'])
        ) {
            return null;
        }

        return new Token(
            $id,
            $token,
            (array) $token['data'],
            (new \DateTimeImmutable())->setTimestamp($token['iat']),
            (new \DateTimeImmutable())->setTimestamp($token['exp'])
        );
    }

    public function create(array $payload, \DateTimeInterface $expiresAt = null): TokenInterface
    {
        $issuedAt = ($this->time)('now');
        $expiresAt = $expiresAt ?? ($this->time)($this->expiresAt);
        $token = [
            'iat' => $issuedAt->getTimestamp(),
            'exp' => $expiresAt->getTimestamp(),
            'data' => $payload,
        ];

        return new Token(
            JWT::encode($token,$this->secret,$this->algorithm),
            $token,
            $payload,
            $issuedAt,
            $expiresAt
        );
    }

    public function delete(TokenInterface $token): void 
    {
        // We don't need to do anything here since JWT tokens are self-contained.
    }
}
```

And here is an example of how you can implement the `Token` class:

```php
namespace App\Auth\Storage;

use DateTimeInterface;
use Spiral\Auth\TokenInterface;

final class Token implements TokenInterface
{
    public function __construct(
        private readonly string $id,
        private readonly array $token,
        private readonly array $payload,
        private readonly \DateTimeImmutable $issuedAt,
        private readonly \DateTimeImmutable $expiresAt
    ) {
    }

    public function getID(): string
    {
        return $this->id;
    }

    public function getToken(): array
    {
        return $this->token;
    }

    public function getPayload(): array
    {
        return $this->payload;
    }

    public function getIssuedAt(): \DateTimeImmutable
    {
        return $this->issuedAt;
    }

    public function getExpiresAt(): DateTimeInterface
    {
        return $this->expiresAt;
    }
}
```

Register it through the configuration file `app/config/auth.php`.

```php
use Spiral\Core\Container\Autowire;

return [
    // ...
    'storages' => [
         'jwt' => new Autowire(\App\Auth\Storage\JWTTokenStorage::class, [
             'secret' => 'secret', 
             'algorithm' => 'HS256',
             'expiresAt' => '+30 days',
         ]),
         // ...
    ]
]
```

After that, you can set the `JWTTokenStorage` as the default token storage:

```dotenv
AUTH_TOKEN_STORAGE=jwt
```

## Steps for implementing user authentication in a Spiral Framework application

### 1. Create a login form

The first step in implementing user authentication is to create a login form that allows users to enter their
credentials (e.g. username and password).

### 2. Implement a request filter

Next, you will need to implement a request filter that can handle login requests from the login form. The request filter
should validate the user's credentials and authenticate the user if they are valid.

> **Note:**
> Read more about request filters [here](../filters/configuration.md).

Here is an example of a request filter that can be used to authenticate users:

```php
namespace App\Filters;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class LoginRequest extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;

    #[Post]
    public string $password;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            'username' => ['required', 'string', ['string::longer', 3], ['string::shorter', 32]],
            'password' => ['required'],
        ]);
    }
}
```

### 3. Create a login method in the controller

To authenticate the user for subsequent requests, you will need to create a login method in the controller dedicated to
authentication. Once the login method has been implemented, you can authenticate the user by calling it and passing in
the user's
credentials.

Here is an example of a login method that can be used to authenticate users:

```php
declare(strict_types=1);

namespace App\Controller;

use App\Filters\LoginRequest;
use App\Repository\UserRepository;
use Spiral\Auth\AuthScope;
use Spiral\Auth\TokenStorageInterface;
use Spiral\Router\Annotation\Route;

class LoginController
{
    public function __construct(
        private readonly UserRepository $users,
        private readonly AuthScope $auth,
        private readonly TokenStorageInterface $tokens
    ) {
    }

    #[Route(route: '/login', name: 'login.post', methods: ['POST'])]
    public function login(LoginRequest $request)
    {
        // application specific login logic
        $user = $this->users->findByUsername($request->username);

        if (
            $user === null
            ||
            !$user->verifyPassword($request->password)
        ) {
            // Invalid credentials ...
        }
        
        // If credentials are valid, we can log the user in
    }
}
```

### 4. Authenticate the user

If the credentials are valid, the method should create a token that can be used to authenticate the user for subsequent
requests. This method should use the `AuthContextInterface` to generate a token with a payload (e.g. `userID => id`).

```php
public function login(LoginRequest $request)
{
    // ... see above
    
    // If credentials are valid, we can log the user in
    
    $this->auth->start(
        $this->authTokens->create(['userID' => $user->id])
    );
    
    // Send success response ...
}
```

### 5. Send the token to the client

After generating an authentication token and storing it in the token storage, the final step is to send the token to the
client. The `AuthMiddleware` middleware will automatically send the token to the client using the transport defined in
the `AuthContext` or default transport if none is defined.

## Working with authenticated users

The `AuthMiddleware` is responsible for checking incoming requests for an authentication token and, if one is found,
loading the user's authentication information from the token storage.

This means that if an authenticated user makes a request to your Spiral Framework application, the `AuthMiddleware` will
automatically load the user's token and use it to authenticate the user. The user's authentication information will then
be available to your controllers and other components via the `AuthContextInterface`.

### Accessing the authenticated user

When you retrieve a user object from the `AuthContextInterface` using the `getActor()` method, a default
`Spiral\Auth\ActorProviderInterface` will be used to load the object from the appropriate data source.

The `ActorProviderInterface` is a convenience interface that provides a standard way to load user objects from various
data sources, such as a database, a microservice, or other external system. By implementing this interface, you can
create a custom provider that can retrieve user objects from your desired data source.

To see if the user authenticated, simply check if the auth context has a non-empty actor:

```php
use Spiral\Http\Exception\ClientException\ForbiddenException;

public function index()
{
    if ($this->auth->getActor() === null) {
        throw new ForbiddenException();
    }
    
    dump($this->auth->getActor());
}
```

### Logging out the user

To log user out call the method close of auth context or `AuthContextInterface`:

```php
public function logout(): void
{
    $this->auth->close();
}
```


