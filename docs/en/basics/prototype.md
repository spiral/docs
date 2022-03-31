# Prototyping
Spiral Framework comes with a development extension that speeds up the development of application services, controllers,
middleware, and other classes via AST modification (a.k.a. it writes code for you). The extension includes IDE friendly tooltips for most common framework components and Cycle Repositories.

## Installation

To install the extension:

```bash
$ composer require spiral/prototype
```

> Please note that the spiral/framework >= 2.7 already includes this component.

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

> Attention, the extension will invoke `TokenizerConfig`, make sure to add it at the end of the bootload chain.

Now you can run `php app.php configure` to generate IDE tooltips.

## Usage of Prototype Properties
To use the prototyping abilities of the framework, add `Spiral\Prototype\Traits\PrototypeTrait` to any of your classes. 
Once complete your IDE will immediately suggest you available classes and Cycle Repositories:

![IDE Tooltips](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

You can use this suggestion directly, without need for any import:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

> The code will work via magic `__get` on the object.

Once your prototyping phase is complete, you can remove the trait and inject dependencies via:

```bash
$ php app.php prototype:inject -r
```

> Use `-r` flag to remove `PrototypeTrait`.

The extension will modify your class into a given form:

```php
namespace App\Controller;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    /** @var ViewsInterface */
    private $views;
    
    /** @var UserRepository */
    private $users;

    /**
     * @param ViewsInterface $views
     * @param UserRepository $users
     */
    public function __construct(ViewsInterface $views, UserRepository $users)
    {
        $this->users = $users;
        $this->views = $views;
    }

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```
> The formatting around the injected lines will be affected.

For PHP 7.4 now available two additional flags:
- `typedProperties (t)` will inject properties with a type
- `no-phpdoc` will omit PHP doc block for typed properties
> Note that these flags work only if the latest `nikic/php-parser` version with PHP 7.4 support is installed.

```bash
$ php app.php prototype:inject -r -t --no-phpdoc
```
```php
namespace App\Controller;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    private ViewsInterface $views;
    private UserRepository $users;

    /**
     * @param ViewsInterface $views
     * @param UserRepository $users
     */
    public function __construct(ViewsInterface $views, UserRepository $users)
    {
        $this->users = $users;
        $this->views = $views;
    }

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```


To view all the classes which use prototyped properties without modifying them:

```bash
$ php app.php prototype:list
```

> You can remove the `spiral/prototype` extension after all injects are complete.

## Custom Properties
You can register any number of prototyped properties using `Spiral\Prototype\Bootloader\PrototypeBootloader` in your bootloader:

```php
public function boot(PrototypeBootloader $prototype)
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> You can combine such an approach with automatic class discovery to achieve better integration of domain layer architecture into your development process.

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
db | Cycle\Database\DatabaseInterface
dbal | Cycle\Database\DatabaseProviderInterface
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
storage | Spiral\Storage\StorageInterface
validator | Spiral\Validation\ValidationInterface
views | Spiral\Views\ViewsInterface
auth | Spiral\Auth\AuthScope
authTokens | Spiral\Auth\TokenStorageInterface
