# The Basics — Session

If you need to enable a session in an alternative bundle, require composer package `spiral/session` and add
bootloader `Spiral\Bootloader\Http\SessionBootloader` into your app.

## SessionInterface

A user session can be accessed using context specific object `Spiral\Session\SessionInterface`:

```php app/src/Interface/Controller/HomeController.php
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
which provides classing `set`, `get`, `delete` or any different functionality. Use `getSection` of session object for
these
purposes:

```php app/src/Interface/Controller/HomeController.php
public function index(SessionInterface $session): void
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Session Scope

To simplify the usage of a session in singleton services and controllers, use `Spiral\Session\SessionScope`. This
component is also available via prototype property `session`. The component can be used within singleton services and
always point to an active session context:

```php app/src/Interface/Controller/HomeController.php
namespace App\Interface\Controller;

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

The session will be automatically started on first data access and committed when the request
leaves `SessionMiddleware`. To control the session manually, use methods of `Spiral\Session\SessionInterface` object.

> **Note**
> SessionScope fully implements SessionInterface.

### Resume session

To manually resume/create a session:

```php
$this->session->resume();
```

### Commit

To manually commit and close a session:

```php
$this->session->commit();
```

### Abort

To discard all the changes and close a session:

```php
$this->session->abort();
```

### Get Session ID

To get a session ID (only when the session is resumed):

```php
dump($this->session->getID());
```

To check if the session has started:

```php
dump($this->session->isStarted());
```

### Destroy

To destroy a session and all the content:

```php
$this->session->destroy();
``` 

### Regenerate ID

To issue a new session ID without affecting the session content:

```php
$this->session->regenerateID();
```

## Custom Configuration

To alter session configuration, create file `app/config/session.php` to change needed values.

The session component is based on native PHP session implementation. By default, the session content is stored in the
file system in the `runtime/session` directory. If your application will be load balanced across multiple web servers,
you should choose a centralized store that all servers can access, such as Redis.

The session `handler` configuration option defines where session data will be stored for each request.
Spiral ships with several drivers out of the box:

### **FileHandler** configuration

Sessions are stored in `runtime/session` folder.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    'lifetime' => 86400,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime'  => 86400
        ]
    )
];
```

### **CacheHandler** configuration

Sessions are stored in one of cache based storages configured in Cache component.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\CacheHandler;

$ttl = 86400;

return [
    'lifetime' => $ttl,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        CacheHandler::class,
        [
            'storage' => 'my-storage', // (Optional)  Cache storage name. Default - current cache storage
            'ttl' => $ttl,
            'prefix' => 'foo:' // (Optional) By default, session:
        ]
    )
];
```

### Custom Session Handler

If none of the existing session drivers fit your application's needs, Spiral makes it possible to write your own session
handler. Your custom session driver should implement PHP's
built-in [`SessionHandlerInterface`](https://www.php.net/manual/en/class.sessionhandlerinterface.php).

```php app/config/session.php
return [
    'handler' => new Autowire(
        MemoryHandler::class,
        [
            'driver' => 'redis',
            'database' => 1,
            'lifetime' => 86400
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
