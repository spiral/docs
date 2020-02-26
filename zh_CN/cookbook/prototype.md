# 速查手册 - 开发辅助

Spiral 框架自带了一个开发扩展，可以通过 AST 修改（可以理解为自动编写代码）辅助开发者，提升服务、控制器、中间件和其它类的开发速度。该扩展包含 IDE 友好的智能提示，可以为大部分 Spiral 框架下的常用组件和 Cycle 系列组件提供代码提示。

## 安装
要安装该扩展，可以执行以下命令：

```bash
$ composer require spiral/prototype
```

安装完成以后，把 `Spiral\Prototype\Bootloader\PrototypeBootloader` 添加到 App 类的引导程序列表中：

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

> 注意：Prototype 扩展会调用 `TokenizerConfig`, 所以请把 `PrototypeBootloader` 添加到引导程序链的最后位置。

然后可以执行 `php app.php configure` 命令来生成 IDE 提示。

## 原型属性的使用

在开发过程中，要使用框架的原型制作功能，需要在你的类中使用 `Spiral\Prototype\Traits\PrototypeTrait`. 一旦使用了这个 trait, IDE 就能够为可用的类以及 Cycle 系列组件提供代码提示功能：

![IDE 代码提示](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

无需引入任何额外的类或者 PHPDoc 声明，即可直接使用智能提示：

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

> 代码提示对于通过 `__get` 魔术方法定义的对象也有效。

在原型开发阶段完成以后，你可以通过以下命令快速移除使用的 trait 以及相关的注入依赖：

```bash
$ php app.php prototype:inject -r
```

> 加上 `-r` 参数表示移除 `PrototypeTrait`.

执行上面的命令之后，会自动从你的类代码中删除对应的引入和 trait 使用：

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

> 注入代码周围行的代码格式也会受到影响。

也可以执行下面的命令，列出所有使用了原型属性的类，但不对它们进行修改：

```bash
$ php app.php prototype:list
```

> 在所有的注入都完全移除之后，你可以从项目中完全删除 `spiral/prototype` 扩展。

## 自定义属性

除了 Spiral 框架官方提供的属性以外，你也可以在你的引导程序中借助 `Spiral\Prototype\Bootloader\PrototypeBootloader` 来注册自己的原型属性：

```php
public function boot(PrototypeBootloader $prototype)
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> 可以把这种方法与类的自动发现相结合，以便更好地把领域层体系结构集成到开发过程中。

## 基于注解实现

除了上述的使用 trait 的方法以外，也可以用注解的方式来注册原型类和服务。在你需要注入的类上面使用 `Spiral\Prototype\Annotation\Prototyped` 注解：

```php
namespace App\Service;

use Spiral\Prototype\Annotation\Prototyped;

/** @Prototyped(property="myService") */
class MyService
{

}
```

在通过任何一种方式注册了自己的原型属性之后，需要执行 `php app.php update` 或者 `php app.php prototype:dump` 命令来自动定位你的服务类。

## 可用快捷方式

Spiral 框架有大量的组件快捷方式可以使用：

属性 | 组件
--- | ---
app | App\App (或其它实现 `Spiral\Boot\Kernel` 的类)
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
storage | Spiral\Storage\StorageInterface
validator | Spiral\Validation\ValidationInterface
views | Spiral\Views\ViewsInterface
auth | Spiral\Auth\AuthScope
authTokens | Spiral\Auth\TokenStorageInterface
