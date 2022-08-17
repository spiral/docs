# HTTP - Session

The default application skeleton enables session integration by default.

If you need to enable session in an alternative bundle, require composer package `spiral/session` and add
bootloader `Spiral\Bootloader\Http\SessionBootloader` into your app.

## SessionInterface

User session can be accessed using context specific object `Spiral\Session\SessionInterface`:

```php
use Spiral\Session\SessionInterface;

// ...

public function index(SessionInterface $session): void
{
    $session->resume();
    dump($session->getID());
}
```

> **Note**
> You are not allowed to store session reference in singleton objects. See the workaround below.

## Session Section

By default, you are not allowed to work with session directly, but rather allocate the isolated and named section
which provides classing `set`, `get`, `delete` and etc functionality. Use `getSection` of session object for these
purposes:

```php
public function index(SessionInterface $session): void
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Session Scope

To simplify the usage of the session in singleton services and controllers, use `Spiral\Session\SessionScope`. This
component is also available via prototype property `session`. The component can be used within singleton services and
always point to active session context:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        dump($this->session->getSection('cart')->getAll());
    }
}
```

## Session Lifecycle

The session will be automatically started on first data access and committed when the request will
leave `SessionMiddleware`. To control session manually use methods of `Spiral\Session\SessionInterface` object.

> **Note**
> SessionScope fully implements SessionInterface.

### Resume session

To manually resume/create session:

```php
$this->session->resume();
```

### Commit

To manually commit and close the session:

```php
$this->session->commit();
```

### Abort

To discard all the changes and close the session:

```php
$this->session->abort();
```

### Get Session ID

To get session ID (only when session resumed):

```php
dump($this->session->getID());
```

To check if session has been started:

```php
dump($this->session->isStarted());
```

### Destroy

To destroy the session and all the content:

```php
$this->session->destroy();
``` 

### Regenerate ID

To issue new session ID without affecting the session content:

```php
$this->session->regenerateID();
```

## Custom Configuration

To alter session configuration create file `app/config/session.php` to change needed values:

```php
<?php

declare(strict_types=1);

use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    'lifetime' => 86400,
    'cookie'   => 'sid',
    'secure'   => false,
    'handler'  => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime'  => 86400
        ]
    )
];
```

### Custom Session Handler

The session component based on native PHP session implementation. By default, session content stored in the file system
in the `runtime/session` directory.

To change the session storage driver, use
any `SessionHandlerInterface` [compatible handler](https://www.php.net/manual/en/class.sessionhandlerinterface.php).

```php
<?php
return [
    'handler' => new Autowire(
        MemoryHandler::class,
        [
            'driver' => 'redis',
            'database' => 1,
            'lifetime'  => 86400
        ]
    )
];
```

> **Note**
> You can use Autowire instead of the class name to configure additional parameters.

### Customize Session initialization

The session is initialized using a special factory `Spiral\Session\SessionFactoryInterface`.

```php

namespace Spiral\Session;

interface SessionFactoryInterface
{
    /**
     * @param string $clientSignature User specific token, does not provide full security but
     *                                     hardens session transfer.
     * @param string|null $id When null - expect php to create session automatically.
     */
    public function initSession(string $clientSignature, string $id = null): SessionInterface;
}
```

You can replace the default implementation of `Spiral\Session\SessionFactoryInterface` in the container with your own.

```php
$container->bindSingleton(\Spiral\Session\SessionFactoryInterface::class, CustomSessionFactory::class);
```
