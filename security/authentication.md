# Security - User Authentication
The framework includes the set of components to authorize users via temporary or permanent tokens from different
sources and safely manage user context.

> The component does not enforce any specific User entity interface and does not limit the application to HTTP scope only 
> (GRPC auth is possible as well).

## Principle of Work
The authentication extension will create an IoC scope for `Spiral\Auth\AuthContextInterface` which points to the currently authorized actor 
(User, API Client). The actor is fetched from `Spiral\Auth\ActorProviderInterface` using `Spiral\Auth\TokenInterface`.

The token is managed by `Spiral\Auth\TokenStorageInterface` and always includes the payload (for example `["userID" => $id]`, LDAP creds, etc.). 
The token payload used to find current application user via `Spiral\Auth\ActorProviderInterface`.

The token storage can either store token in the external source (such as database, Redis, or file) or decode it on a fly. The framework
includes multiple token implementations out of the box for more comfortable use. 

> You can use multiple token and actor providers inside one application.

## Installation and Configuration
To install authorization extension for Web bundle:

```bash
$ composer require spiral/auth spiral/auth-http
```

> Please note that the spiral/framework >= 2.6 already includes this component.

The package `spiral/auth` provides standard interfaces without the relation to any specific dispatching method, while
`spiral/auth-http` includes HTTP Middleware, Token transport (Cookie, Header), and Firewall components.

To activate the component add the bootloader `Spiral\Bootloader\Auth\HttpAuthBootloader`:

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    // ...
]
```

Once installed, you must decide how to store the user authentication token.

### Session Token Storage
To store tokens in PHP session make sure that `spiral/session` extension is installed, to enable session storage use 
bootloader `Spiral\Bootloader\Auth\TokenStorage\SessionTokensBootloader`:

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    Framework\Auth\TokenStorage\SessionTokensBootloader::class,
    // ...
]
```

### Database Token Storage
The framework can store the token in the database via Cycle ORM. Activate `Spiral\Bootloader\Auth\TokenStorage\CycleTokensBootloader` for this purpose:

```php
[
    // ...
    Framework\Auth\HttpAuthBootloader::class,
    Framework\Auth\TokenStorage\CycleTokensBootloader::class,
    // ...
]
```

You must generate and run database migration or run `cycle:sync` in order to create needed table:

```php
$ php app.php migrate:init
$ php app.php cycle:migrate -v -r
```

## Actor Provider and Token Payload
The next step to configure a way to fetch actors/users based on token payloads, we must implement and register interface
`Spiral\Auth\ActorProviderInterface` for this purposes.

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

/**
 * @Cycle\Entity(repository="Repository/UserRepository")
 * @Cycle\Table(indexes={
 *     @Cycle\Table\Index(columns={"username"}, unique=true)
 * })
 */
class User
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /** @Cycle\Column(type="string") */
    public $name;

    /** @Cycle\Column(type="string") */
    public $username;

    /** @Cycle\Column(type="string") */
    public $password;
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
public function index(Transaction $t)
{
    $u = new User();
    $u->name = 'Antony';
    $u->username = 'username';
    $u->password = password_hash('password', PASSWORD_DEFAULT);

    $t->persist($u)->run();
}
```

Register actor provider to enable it, create and activate the Bootloader in your application:

```php
namespace App\Bootloader;

use App\Database\Repository\UserRepository;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Auth\AuthBootloader;

class UserBootloader extends Bootloader
{
    public function boot(AuthBootloader $auth)
    {
        $auth->addActorProvider(UserRepository::class);
    }
}
```

## Authenticate User
The user authentication process happens via `Spiral\Auth\AuthContextInterface`. You can receive the instance of the auth context
object via method injection. 

```php
public function index(AuthContextInterface $auth)
{
    // work with auth context
}
```

> You are not allowed to store `AuthContextInterface` inside singleton services, see above how to bypass it.

Alternatively, you can use `Spiral\Auth\AuthScope` which can be stored in singleton services and prototyped via property
`auth`.

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->auth);
    }
}
```

### Login
The user login will require us to create a login form and proper request filter. 

```php
namespace App\Request;

use Spiral\Filters\Filter;

class LoginRequest extends Filter
{
    public const SCHEMA = [
        'username' => 'data:username',
        'password' => 'data:password'
    ];

    public const VALIDATES = [
        'username' => ['notEmpty'],
        'password' => ['notEmpty']
    ];
}
```

Create login method in the controller dedicated to authentication:

```php
public function login(LoginRequest $login)
{
    if (!$login->isValid()) {
        return [
            'status' => 400,
            'errors' => $login->getErrors()
        ];
    }

    // application specific login logic
    $user = $this->users->findOne(['username' => $login->getField('username')]);
    if (
        $user === null
        || !password_verify($login->getField('password'), $user->password)
    ) {
        return [
            'status' => 400,
            'error'  => 'No such user'
        ];
    }

    // create token
}
```

To authenticate the user for the following requests, you must create token with the payload compatible with your
`ActorProviderInterface` (userID => id). 

We will need an instance of `AuthContextInterface` and `TokenStorageInterface`
to do that. We can access both instances via prototype properties `auth` and `authTokens`:

```php
public function login(LoginRequest $login)
{
    // ... see above

    // create token
    $this->auth->start(
        $this->authTokens->create(['userID' => $user->id])
    );

    return [
        'status'  => 200,
        'message' => 'Authenticated!'
    ];
}
```

The user authenticated. 

### Check Authenticated
To see if the user authenticated simply check if auth context has non-empty actor:

```php
public function index()
{
    if ($this->auth->getActor() === null) {
        throw new ForbiddenException();
    }
    
    dump($this->auth->getActor());
}
```

> You can use RBAC Security to authenticate and authorize users at the same time. 

### Logout
To log user out call method `close` of auth context or AuthScope:

```php
public function logout()
{
    $this->auth->close();
}
```

## RBAC security
You can use authenticated user as an actor for the RBAC security component, make sure to implement `Spiral\Security\ActorInterface` in your `App\Database\User`:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;
use Spiral\Security\ActorInterface;

/**
 * @Cycle\Entity(repository="Repository/UserRepository")
 * @Cycle\Table(indexes={
 *     @Cycle\Table\Index(columns={"username"}, unique=true)
 * })
 */
class User implements ActorInterface
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /** @Cycle\Column(type="string") */
    public $name;

    /** @Cycle\Column(type="string") */
    public $username;

    /** @Cycle\Column(type="string") */
    public $password;

    public function getRoles(): array
    {
        return ['user'];
    }
}
```

And activate the bootloader `Spiral\Bootloader\Auth\SecurityActorBootloader` to link two components together:

```php
[
    // ...
    Framework\Auth\SecurityActorBootloader::class,
    // ...
]
```

## Firewall Middleware
You can protect some of your route targets by attaching firewall middleware to prevent unauthorized access.

By default, spiral provides only one firewall which will overwrite the target url: 

```php
use Spiral\Auth\Middleware\Firewall\OverwriteFirewall;

// ...

(new Route('/account/<controller>/<action>', $accountTarget))
        ->withMiddleware(new OverwriteFirewall(new Uri('/account/login')));
```

### Custom Firewall
To implement your firewall, extend `Spiral\Auth\Middleware\Firewall\AbstractFirewall`:

```php
final class OverwriteFirewall extends AbstractFirewall
{
    protected function denyAccess(Request $request, RequestHandlerInterface $handler): Response
    {
        // user is not authenticated
        return $handler->handle($request);
    }
}
```
