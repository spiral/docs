# Cookbook - Prototyping
Spiral Framework comes with development extension which speeds up the development of application services, controllers,
middleware and other classes via AST modification (a.k.a. it writes code for you). Extension includes IDE friendly tooltips for most common framework components and Cycle Repositories.

## Installation
To install extension:

```bash
$ composer require spiral/prototype
```

Make sure to add `Spiral\Prototype\Bootloader\PrototypeBootloader` to your App class:

```php
class App extends Kernel
{
    // ...

    protected const APP = [
        RoutesBootloader::class,
        LoggingBootloader::class,

        PrototypeBootloader::class
    ];
}
```

> Attention, the extension will invoke `TokenierConfig`, make sure it add it after the end of bootload chain.

Now you can run `php app.php configure` to generate IDE tooltips.

## Usage of Prototype Properties
To use prototyping abilities of framework add `Spiral\Prototype\Traits\PrototypeTrait` to any of your classes. 
Once complete your IDE will immediately suggest you available classes and Cycle Repositories:

![IDE Tooltips](/resources/virtual-bindings.gif)

You can use this suggestion directly, without need for any import:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        $select = $this->users->select();
    }
}
```

> The code will work via magic `__get` on the object.

Once your prototyping phase is complete you can remove the trait and inject dependencies via:

```bash
$ php app.php prototype:inject -r
```

> Use `-r` flag to remove `PrototypeTrait`.

The extension will modify your class into given form:


```php
namespace App\Controller;

use App\UserRepository;

class HomeController
{
    /** @var UserRepository */
    private $users;

    /**
     * @param UserRepository $users
     */
    public function __construct(UserRepository $users)
    {
        $this->users = $users;
    }

    public function index()
    {
        $select = $this->users->select();
    }
}
```

> The formatting around the injected lines will be affected.

To view all the classes which use prototyped properties without modifying them:

```bash
$ php app.php prototype:list
```

> You can remove `spiral/prototype` extension after all injects are complete.

## Custom Properties
You can register any number of prototyped properties using `Spiral\Prototype\Bootloader\PrototypeBootloader` in your bootloader:

```php
public function boot(PrototypeBootloader $prototype)
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> You can combine such approach with automatic class discovery to achieve better integration of domain layer architecture into your development process.

## Annotation Based
Alternatively, you can use annotations to register prototype classes and services. Use annotation `Spiral\Prototype\Annotation\Prototyped` in the class you want to inject:

```php
namespace App\Service;

use Spiral\Prototype\Annotation\Prototyped;

/** @Prototyped(property="myService") */
class MyService
{

}
```

Make sure to run `php app.php update` or `php app.php prototype:dump` to auto-locate your service.

## Available Shortcuts
There are number of component shortcuts available for your usage:

Property | Component
--- | ---
app | App\App (or class which implements `Spiral\Boot\Kernel`)
classLocator | Spiral\Tokenizer\ClassesInterface
console | Spiral\Console\Console
container | Psr\Container\ContainerInterface
db | Spiral\Database\DatabaseInterface
dbal | Spiral\Database\DatabaseProviderInterface
encrypter | Spiral\Encrypter\EncrypterInterface
env | Spiral\Boot\EnvironmentInterface
files | Spiral\Files\FilesInterface
guard | Spiral\Security\GuardInterface
http | Spiral\Http\Http
i18n | Spiral\Translator\TranslatorInterface
input | Spiral\Http\Request\InputManager
session | Spiral\Session\SessionScope
cookies | Spiral\Cookies\CookieManager
logger | Psr\Log\LoggerInterface
logs | Spiral\Logger\LogsInterface
memory | Spiral\Boot\MemoryInterface
orm | Cycle\ORM\ORMInterface
paginators | Spiral\Pagination\PaginationProviderInterface
queue | Spiral\Jobs\QueueInterface
request | Spiral\Http\Request\InputManager
response | Spiral\Http\ResponseWrapper
router | Spiral\Router\RouterInterface
server | Spiral\Goridge\RPC
snapshots | Spiral\Snapshots\SnapshotterInterface
'storage | Spiral\Storage\StorageInterface
validator | Spiral\Validation\ValidationInterface
views | Spiral\Views\ViewsInterface
