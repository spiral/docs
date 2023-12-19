# The Basics — Prototyping

Prototyping is a development phase where you sketch out your application structure, quickly iterating over the basic 
design of your classes and their interactions. The Spiral offers a powerful extension that enhances this prototyping 
process, that speeds up the development of application services, controllers, middleware, and other classes via AST 
modification (a.k.a. it writes code for you). The extension includes IDE friendly tooltips for most common framework 
components and Cycle Repositories.

## Installation

Make sure to add `Spiral\Prototype\Bootloader\PrototypeBootloader` to your App class:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
        // ...
    ];
}
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
    // ...
];
```

Read more about bootloaders in the [Framework — Bootloaders](../framework/bootloaders.md) section.
:::

::::

Now you can run `php app.php configure` to generate IDE autocomplete helpers for your application.

## Usage of Prototype Properties

### Prototyping

To start using the prototyping powers, you add `Spiral\Prototype\Traits\PrototypeTrait` trait to any class you want to 
prototype. This trait is like a magical helper that helps your IDE to find and use other parts of your app without 
needing to set up a lot of details.

![IDE Tooltips](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

**Here's a more detailed look at how to add it to a controller class:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

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

When you include this trait in your class, your IDE will start suggesting which services or components you can use. For
instance, if your application has a user management system, typing `$this->users` might prompt your IDE to suggest 
methods from your user service or repository.

This is great for quickly testing out ideas because you don’t have to set up all the formal connections (like dependency
injection) between parts of your app.

### Prototyping to Real Code

Magic properties are convenient, but they are not the best for performance and clarity in the long term. So, once you're
satisfied with how things work, you can tell Spiral to replace the magic with actual code.

Just run the following command:

```terminal
php app.php prototype:inject -r
```

This tells Spiral: *"Go through my classes, look for these magic properties, and turn them into real, solid code."* It \
will automatically add the necessary code for dependency injection, making your app ready for real-world use.

> **Note**
> Use `-r` flag to remove `PrototypeTrait` from the class.

**Here's how the code will look after the command is run:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views, 
        private readonly UserRepository $users
    ) {
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

### Discovering classes with prototyping

In some cases, you might want to find all the classes that use prototyped properties. To view all the classes just run
the following command:

```terminal
php app.php prototype:usage
```

It will output a list of classes like in the example below:

```terminal
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| Class:                                                             | Property:            | Target:                                                      |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| App\Endpoint\Web\Controller\User\SetupPasswordAction               | userService          | App\Service\UserServiceInterface                             |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
| App\Endpoint\Web\Controller\User\SetupPasswordFormAction           | users                | App\Repository\UserRepositoryInterface                       |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
| App\Endpoint\Web\Controller\Auth\LoginFormAction                   | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
```

> **Note**
> that you can remove the `spiral/prototype` extension after all the injects are complete.

### Reviewing Changes

After the previous step, it’s a good idea to review the changes to ensure everything got set up the way you expected. 
The prototyping tool is smart, but it's always good to double-check. You might need to make adjustments or 
optimizations to the code.

Once you're happy with the setup, you can keep building your app with the solid foundation that the prototyping tool 
helped you create.

## Custom Properties

You can register any number of prototyped properties using `Spiral\Prototype\Bootloader\PrototypeBootloader` in your
bootloader:

```php
use Spiral\Prototype\Bootloader\PrototypeBootloader;

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

```php app/src/Domain/User/Service/UserService.php
namespace App\Domain\User\Service;

use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'userService')]
final class UserService
{
    // ...
}
```

Make sure to run `php app.php update` or `php app.php prototype:dump` to auto-locate your service.

> **Warning**
> To use attributes with interfaces, you need to enable search of interfaces. To do so, read the
> [Configuring listeners](../advanced/tokenizer.md#configuring-listeners) section.

## Available Shortcuts

To view all the registered shortcuts in your application, run:

```terminal
php app.php prototype:list
```

### Available Shortcuts

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
| orm          | Cycle\ORM\ORMInterface (`spiral/cycle-bridge` package should be installed)                   |
| paginators   | Spiral\Pagination\PaginationProviderInterface                                                |
| queue        | Spiral\Queue\QueueInterface                                                                  |
| queueManager | Spiral\Queue\QueueConnectionProviderInterface                                                |
| request      | Spiral\Http\Request\InputManager                                                             |
| response     | Spiral\Http\ResponseWrapper                                                                  |
| router       | Spiral\Router\RouterInterface                                                                |
| server       | Spiral\Goridge\RPC (`spiral/roadrunner-bridge` package should be installed)                  |
| snapshots    | Spiral\Snapshots\SnapshotterInterface                                                        |
| storage      | Spiral\Storage\StorageInterface                                                              |
| validator    | Spiral\Validation\ValidationInterface                                                        |
| views        | Spiral\Views\ViewsInterface                                                                  |
| auth         | Spiral\Auth\AuthScope                                                                        |
| authTokens   | Spiral\Auth\TokenStorageInterface                                                            |
| cache        | Psr\SimpleCache\CacheInterface                                                               |
| cacheManager | Spiral\Cache\CacheStorageProviderInterface                                                   |
