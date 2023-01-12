# The Basics — Prototyping

Spiral Framework comes with a development extension that speeds up the development of application services, controllers,
middleware, and other classes via AST modification (a.k.a. it writes code for you). The extension includes IDE friendly
tooltips for most common framework components and Cycle Repositories.

## Installation

To install the extension:

```bash
composer require spiral/prototype
```

> **Note**
> that the spiral/framework >= 2.7 already includes this component.

Make sure to add `Spiral\Prototype\Bootloader\PrototypeBootloader` to your App class:

```php  app/src/Application/Kernel.php
class App extends Kernel
{
    // ...

    protected const APP = [
        //...

        \Spiral\Prototype\Bootloader\PrototypeBootloader::class
    ];
}
```

> **Note**
> that the extension will invoke `TokenizerConfig`, make sure to add it at the end of the bootload chain.

Now you can run `php app.php configure` to generate IDE tooltips.

## Usage of Prototype Properties

To use the prototyping abilities of the framework, add `Spiral\Prototype\Traits\PrototypeTrait` to any of your classes.
Once it's added, your IDE will immediately suggest available classes and Cycle Repositories to you:

![IDE Tooltips](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

You can use this suggestion directly, without a need for any import:

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

> **Note**
> that the code will work via magic `__get` on the object.

Once the prototyping phase is complete, you can remove the trait and inject dependencies via:

```bash
php app.php prototype:inject -r
```

> **Note**
> Use `-r` flag to remove `PrototypeTrait`.

The extension will modify your class into the given form:

```php
namespace App\Controller;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    private ViewsInterface $views;
    private UserRepository $users;

    public function __construct(ViewsInterface $views, UserRepository $users)
    {
        $this->users = $users;
        $this->views = $views;
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

> **Note**
> that the formatting around the injected lines will be affected.

For PHP 7.4 there are two additional flags available now:

- `typedProperties (t)` will inject properties with a type
- `no-phpdoc` will omit PHP doc block for typed properties

> **Note**
> that these flags work only if the latest `nikic/php-parser` version with PHP 7.4 support is installed.

```bash
php app.php prototype:inject -r -t --no-phpdoc
```

```php
namespace App\Controller;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    private ViewsInterface $views;
    private UserRepository $users;

    public function __construct(ViewsInterface $views, UserRepository $users)
    {
        $this->users = $users;
        $this->views = $views;
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

To view all the classes which use prototyped properties without modifying them:

```bash
php app.php prototype:list
```

> **Note**
> that you can remove the `spiral/prototype` extension after all the injects are complete.

## Custom Properties

You can register any number of prototyped properties using `Spiral\Prototype\Bootloader\PrototypeBootloader` in your
bootloader:

```php
public function boot(PrototypeBootloader $prototype): void
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> **Note**
> that you can combine such an approach with automatic class discovery to achieve better integration of domain layer
> architecture into your development process.

## Attribute Based

Alternatively, you can use attributes to register prototype classes and services. Use
attribute `Spiral\Prototype\Annotation\Prototyped` in the class you want to inject:

```php
namespace App\Service;

use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'myService')]
class MyService
{

}
```

Make sure to run `php app.php update` or `php app.php prototype:dump` to auto-locate your service.

## Available Shortcuts

There are a number of component shortcuts available for you to use:

| Property     | Component                                                                                    |
|--------------|----------------------------------------------------------------------------------------------|
| app          | App\App (or class which implements `Spiral\Boot\Kernel`)                                     |
| classLocator | Spiral\Tokenizer\ClassesInterface                                                            |
| console      | Spiral\Console\Console                                                                       |
| container    | Psr\Container\ContainerInterface                                                             |
| db           | Cycle\Database\DatabaseInterface (`spiral/cycle-bridge` package should be installed)         |
| dbal         | Cycle\Database\DatabaseProviderInterface (`spiral/cycle-bridge` package should be installed) |
| encrypter    | Spiral\Encrypter\EncrypterInterface                                                          |
| env          | Spiral\Boot\EnvironmentInterface                                                             |
| files        | Spiral\Files\FilesInterface                                                                  |
| guard        | Spiral\Security\GuardInterface                                                               |
| http         | Spiral\Http\Http                                                                             |
| i18n         | Spiral\Translator\TranslatorInterface                                                        |
| input        | Spiral\Http\Request\InputManager                                                             |
| session      | Spiral\Session\SessionScope                                                                  |
| cookies      | Spiral\Cookies\CookieManager                                                                 |
| logger       | Psr\Log\LoggerInterface                                                                      |
| logs         | Spiral\Logger\LogsInterface                                                                  |
| memory       | Spiral\Boot\MemoryInterface                                                                  |
| orm          | Cycle\ORM\ORMInterface (If `spiral/cycle-bridge` package should be installed)                |
| paginators   | Spiral\Pagination\PaginationProviderInterface                                                |
| queue        | Spiral\Queue\QueueInterface                                                                  |
| queueManager | Spiral\Queue\QueueConnectionProviderInterface                                                |
| request      | Spiral\Http\Request\InputManager                                                             |
| response     | Spiral\Http\ResponseWrapper                                                                  |
| router       | Spiral\Router\RouterInterface                                                                |
| server       | Spiral\Goridge\RPC (If `spiral/roadrunner-bridge` package should be installed)               |
| snapshots    | Spiral\Snapshots\SnapshotterInterface                                                        |
| storage      | Spiral\Storage\StorageInterface                                                              |
| validator    | Spiral\Validation\ValidationInterface                                                        |
| views        | Spiral\Views\ViewsInterface                                                                  |
| auth         | Spiral\Auth\AuthScope                                                                        |
| authTokens   | Spiral\Auth\TokenStorageInterface                                                            |
| cache        | Psr\SimpleCache\CacheInterface                                                               |
| cacheManager | Spiral\Cache\CacheStorageProviderInterface                                                   |
