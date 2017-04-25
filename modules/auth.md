# User Authentication
Install `spiral/auth` module in order to enable token based authorization of your users.

## Installation
`$ composer require spiral/auth` 

`$ spiral register spiral/auth`

## User Entity and Source
First of all we have to define what is actually represent user in your application, in order to do that create Record `User` and repository `UserRepository`:

```php
class User extends Record
{
    use TimestampsTrait;

    const SCHEMA = [
        'id'         => 'primary',
        'email'      => 'string',
        'password'   => 'string'
    ];

    const INDEXES = [
        [self::UNIQUE, 'email']
    ];

    const SECURED = ['password', 'roles'];
}
```

```php
class UserRepository extends RecordSource
{
    const RECORD = User::class;
}
```

To link your repository and user to auth component we have to implement 2 interfaces.

### PasswordAwareInterface
This interface indicates that user entity contain hashed password:

```php
class User extends Record implements PasswordAwareInterface
{
    /**
     * @inheritdoc
     */
    public function getPasswordHash(): string
    {
        return $this->password;
    }
}
```

### UsernameSource
In order to enable credentials based authentication in your application your repository must implement `UsernameSourceInterface`:
 
```php
class UserRepository extends RecordSource implements UsernameSourceInterface
{
    const RECORD = User::class;

    /**
     * @inheritdoc
     */
    public function findByUsername(string $username)
    {
        return $this->findOne(['email' => $username]);
    }
}
```

## Register Bindings
To allow auth module locate your users via repository create container binding:

```php
const BINDINGS = [
    UsernameSourceInterface::class => UserRepository::class,
];
```

## Passwords
To properly create user use `Spiral\Auth\Hashing\PasswordHasher` dependency to handle password hashing:

```php
public function createUser(PasswordHasher $hasher)
{
    $user = new User();
    $user->email = 'email@domain.com';
    $user->password = $hasher->hash('password');
    $user->save();
}
```

## Enable Authorization context
To enable access to authorization scope in your application add `AuthMiddleware` into your http config or specific route. Once completed you can access current user context using shortcut `auth` or via dependency `Spiral\Auth\ContextInterface`.

```php
public function indexAction()
{
    dump($this->auth->isAuthenticated());
}
```

## Authorize User
To authorize users use `CredentialsAuthenticator` which will handle password comparision logic for us:

```php
public function loginAction(LoginRequest $request, CredentialsAuthenticator $authenticator)
{
    if (!$request->isValid()) {
        return [
            'status' => 400,
            'errors' => $request->getErrors()
        ];
    }

    try {
        $user = $authenticator->getUser($request->username, $request->password);
    } catch (CredentialsException $e)
    {
        thow new ForbiddenException('unable to authorize');
    }

    //Create auth token
    $this->auth->init($this->tokens->createToken($user));

    //Redirect and etc
}
```

Where `LoginRequest`:

```php
class LoginRequest extends RequestFilter
{
    const SCHEMA = [
        'username' => 'data:username',
        'password' => 'data:password'
    ];
    
    const SETTERS = [
        'username' => 'trim',
        'password' => 'trim'
    ];

    const VALIDATES = [
        'username' => [
            ['notEmpty', 'error' => '[[Please enter your username]]']
        ],
        'password' => [
            ['notEmpty', 'error' => '[[Please enter your password]]']
        ],
    ];
}
```

> You can write your own authenticators, see how to manually create auth tokens below.

## Tokens
Note that auth module uses token based authentication for all users, you are able to write your own token operator (modules/auth config):

```php
return [
    //Default token provider
    'defaultOperator' => 'cookie',

    /*
      * Set of auth providers/operators responsible for user session support
      */
    'operators'       => [
        /*
         * Uses active session storage to store user information
         */
        'session'     => [
            'class'   => Operators\SessionOperator::class,
            'options' => [
                'section' => 'auth'
            ]
        ],

        /*
         * Utilized default HTTP basic auth protocol to authenticate user
         */
        'basic'       => [
            'class'   => Operators\HttpOperator::class,
            'options' => []
        ],

        /*
         * Reads token hash from a specified header
         */
        'header'      => [
            'class'   => Operators\PersistentOperator::class,
            'options' => [
                //Token lifetime
                'lifetime' => 86400 * 14,

                //Persistent token storage
                'source'   => bind(\Spiral\Auth\Database\Sources\AuthTokenSource::class),

                //How to read and write tokens in request
                'bridge'   => bind(Operators\Bridges\HeaderBridge::class, [
                    'header' => 'X-Auth-Token',
                ])
            ]
        ],

        /*
         * Stores authentication token into cookie
         */
        'cookie'      => [
            'class'   => Operators\PersistentOperator::class,
            'options' => [
                //Cookie and token lifetime
                'lifetime' => 86400 * 7,

                //Persistent token storage
                'source'   => bind(\Spiral\Auth\Database\Sources\AuthTokenSource::class),

                //How to read and write tokens in request
                'bridge'   => bind(Operators\Bridges\CookieBridge::class, [
                    'cookie' => 'auth-token',
                ])
            ]
        ],

        /*
         * Stores authentication token into cookie as a remember-me cookie
         */
        'long' => [
            'class'   => Operators\PersistentOperator::class,
            'options' => [
                //Cookie and token lifetime
                'lifetime' => 86400 * 30,

                //Persistent token storage
                'source'   => bind(\Spiral\Auth\Database\Sources\AuthTokenSource::class),

                //How to read and write tokens in request
                'bridge'   => bind(Operators\Bridges\CookieBridge::class, [
                    'cookie' => 'long-token',
                ])
            ]
        ],

        /*{{operators}}*/
    ]
];
```

For example, we can create token to be stored in cookie and verified using ORM models represented by `Spiral\Auth\Database\Sources\AuthTokenSource`:

```php
$this->auth->init($this->tokens->createToken($user, 'long'));
```

> Auth middleware will detect what token and operator automatically.

## Logout
To log user out use `close` method of your context:

```php
$this->auth->close();
```

## Combine with Security component
It's recommended to use authorized user as actor for your security component, you can do that by implementing additional interface in your user entity and by configuring container binding:
 
```php
class User extends Record implements PasswordAwareInterface, ActorInterface
{
    //...
    public function getRoles(): array
    {
        return ['user'];
    }
}
```

To indicate that our user have to be used as actor:

```php
const BINDINGS = [
    ActorInterface::class          => [self::class, 'getActor'],
]
```

Where `getActor` is IoC specific proxy:

```php
public function getActor(\Spiral\Auth\ContextInterface $context)
{
    if ($context->isAuthenticated()) {
        return $context->getUser();
    }
    
    return new Guest();
}
```
