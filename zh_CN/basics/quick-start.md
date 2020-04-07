# 入门基础 - 快速开始

Spiral 框架包含了大量的组件，这些组件各自承担不同的职责，彼此之间相互协同，紧密合作，从而构建出各种复杂的应用。
在本文中，我会通过一个博客应用示例，向大家演示 Rest API、ORM（对象关系映射）、migrations（数据迁移）、request validation（请求校验）、custom annotations（可选的）以及 domain interceptors（域拦截器）的使用。

> 实例中对各个组件和实现方法的介绍都只停留在基础层面，如果需要对任何一个部分进行更深入的理解，请自行阅读文档相应章节。

## 安装

可以通过 composer 命令和官方提供的 `spiral/app` 包进行安装（或者说创建项目），这个包已经为大家集成了 WEB 应用开发中可能用到的大量组件：

```bash
$ composer create-project spiral/app spiral-demo
$ cd spiral-demo
```

> 如果不需要创建完整的 WEB 应用，也可以考虑通过另外一个 `spiral/app-cli` 包来创建初始项目，这个包只继承了最少的依赖项。

当项目创建完成，且所有依赖项都成功安装之后，你可以通过下面的命令立刻启动应用服务器：

```bash
$ ./spiral serve -v -d
```

这个命令启动了 Spiral 的[应用服务器](/zh_CN/framework/application-server.md)。该服务器同时可以用于开发和生产环境，因此开发者的开发环境与最终部署环境会非常接近。Spiral 的应用程序集成了 HTTP/2、GRPC、Queue（队列）、WebSockets 等开发应用程序所需的工具，开箱即用，无需外部代理或其它配置。

默认情况下，通过上面的命令启动服务器之后，可以通过 `http://localhost:8080` 访问 web 应用。初始化的项目包含了几个预设的页面，你可以直接使用它们，或者用它们作为参考。

> 访问 `http://localhost:8080/exception.html` 可以看到默认的报错页面。在页面右侧可以看到默认项目已经集成的所有拦截器和中间件。根据你的实际情况，可以把不需要的关闭，让应用运行时更小更省资源。

## 配置
Spiral 应用程序通过 `app/config` 目录下的配置文件对项目进行配置。在配置文件中，你可以硬编码配置值，当然也可以而且推荐通过 `env` 和 `directory` 函数来获得所需的敏感信息。`spiral/app` 项目使用 [DotEnv 扩展](https://github.com/vlucas/phpdotenv)从项目根目录下的 `.env` 文件中读取环境变量。

> 在 `.rr.yaml` 文件中可以对应用服务器及其插件的参数进行调整。

项目的依赖项定义在 `composer.json` 文件中，并在 `app/src/App.php` 文件中作为引导程序启用。项目默认包含了大量预配置的组件。

### 开发模式

在开发阶段，为了简化开发调试，可以以开发模式启动应用服务器。在开发模式下，应用服务器只是用一个工作进程，并在处理完每个请求之后重新加载代码。

```bash
$ ./spiral serve -v -d -o "http.workers.pool.maxJobs=1" -o "http.workers.pool.numWorkers=1"
```

当然你也可以把参数放到一个单独的 `.rr.dev.yaml` 之类的文件中，并在是用 `spiral` 命令时通过 `-c` 参数指定该配置文件。

> 请参阅 [这篇文档](/zh_CN/start/workers.md) 以了解更多有关工作进程和应用生命周期的信息。

### 清理项目

在我们的演示应用中，不会用到 translation（多语言翻译）、session、cookies、CSRF 和 encryption（加密）组件。所以接下来先从引导程序中移除他们。

打开 `app/src/App.php` 文件，找到并删除下面列出来的代码（注释是为了方便你定位代码，不必删除相关的注释行）：

``` php
// Core Services
Framework\I18nBootloader::class,

// Security and validation
Framework\Security\EncrypterBootloader::class,

// HTTP extensions
Framework\Http\CookiesBootloader::class,
Framework\Http\SessionBootloader::class,
Framework\Http\CsrfBootloader::class,
Framework\Http\PaginationBootloader::class,

// Views and view translation
Framework\Views\TranslatedCacheBootloader::class,

// Application specific services and extensions
Bootloader\LocaleSelectorBootloader::class,
```

删除了相应的引导程序之后，如果你愿意的话，也可以从 `composer.json` 中删除相关的依赖项：

```bash
"spiral/cookies": "^1.0",
"spiral/csrf": "^1.0",
"spiral/session": "^1.1",
"spiral/translator": "^1.2",
"spiral/encrypter": "^1.1",
```

最后，还可以删除掉默认项目自带的以下文件或目录：

- `app/locale`
- `app/src/Bootloader/LocaleSelectorBootloader.php`
- `app/src/Middleware`.

> 提示， 现在应用程序不能工作了，因为我们刚刚删除了渲染 `app/views/home/dark.php` 所需的依赖项（国际化相关的依赖）。

### 数据库连接

博客系统作为常见的数据库驱动的应用，当然需要一个可操作的数据库。数据库的配置文件默认在 `app/config/database.php` 这个位置。新创建的项目已经配置好了一个 SQLite 数据库，存放在 `runtime/runtime.db` 这个路径下。

```php
// app/config/database.php 

use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'runtime'],
    ],
    'drivers'   => [
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'options'    => [
                'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            ]
        ],
    ]
];
```

接下来我们配置一个 MySQL 的连接（如果你没有 MySQL 数据库，可以跳过这部分），连接 MySQL 的信息，最好不要直接硬编码到配置文件中，可以存放到项目根目录下的 `.env` 文件里（这个文件不要上传到你的代码仓库）。比如我们在 `.env` 文件中写入以下环境变量：

``` ini
DB_HOST=localhost
DB_NAME=name
DB_USER=username
DB_PASSWORD=password
```

> 请根据自己的情况修改对应的值

然后在 `app/config/database.php` 文件中，配置一个名为 `mysql` 的数据库驱动（drivers），然后把 `databases` 项下面的 `drivers` 指向新增的 MySQL 驱动：

```php
return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'mysql'],
    ],
    'drivers'   => [
         'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'options'    => [
                'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            ]
        ],
        'mysql' => [
            'driver'     => Driver\MySQL\MySQLDriver::class,
            'connection' => sprintf('mysql:host=%s;dbname=%s', env('DB_HOST'), env('DB_NAME')),
            'username'   => env('DB_USER'),
            'password'   => env('DB_PASSWORD'),
        ],
    ]
];
```

> 请注意当前 `default` 数据库指向了 `default` 连接，`default` 连接指向了 `mysql` 配置。在 Spiral 中，你可以同时配置多个数据库驱动、同时启用多个数据库连接。具体请参阅数据库相关章节的文档。

然后通过以下命令，可以检查数据库连接是否配置正确：

```bash
$ php app.php db:list
```

如果连接配置正确，你会看到类似这样的输出（注意 `Status` 应该是 "connected"）：

```bash
+------------+-----------+---------+---------+-----------+-----------+----------------+
| Name (ID): | Database: | Driver: | Prefix: | Status:   | Tables:   | Count Records: |
+------------+-----------+---------+---------+-----------+-----------+----------------+
| default    | spiral    | MySQL   | ---     | connected | no tables | no records     |
+------------+-----------+---------+---------+-----------+-----------+----------------+
```

> 有关数据库连接的更多信息，请查看[数据库配置文档](/zh_CN/database/configuration.md).

### 假数据

在开发阶段，通常我们需要一个假数据。`fzaninotto/faker` 这个库提供了强大的假数据生成功能。

首先在项目中安装这个库作为依赖项：

```bash
$ composer require fzaninotto/faker
``` 

为了生成数据，需要创建一个 `Faker\Generator` 实例，在 Spiral 中我们不必每次用到它的时候都去生成一次新的实例，可以在 `app/src/Bootloader` 目录下创建一个引导程序，该引导程序在每次我们需要这个类的时候就提供一个单例对象。这里使用的是设计模式中的工厂模式。

```php
namespace App\Bootloader;

use Faker\Factory;
use Faker\Generator;
use Spiral\Boot\Bootloader\Bootloader;

class FakerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        Generator::class => [self::class, 'fakerGenerator']
    ];

    private function fakerGenerator(): Generator
    {
        return Factory::create(Factory::DEFAULT_LOCALE);
    }
}
```

创建好引导程序之后，要把它加到 `app/src/App.php` 的 `LOAD` 或者 `APP` 常量数组中才能启用该引导程序：

```diff
--- a/app/src/App.php
+++ b/app/src/App.php
@@ -85,5 +85,6 @@ class App extends Kernel

         // fast code prototyping
         Prototype\PrototypeBootloader::class,
+        Bootloader\FakerBootloader::class,
     ];
 }
```

> 在 `fakerGenerator` 这个方法上，你可以添加依赖项作为参数，Spiral 容器会自动注入依赖项。 

然后我们修改一下 `app/src/Controllers/HomeController.php` 中的代码，通过 `http://localhost:8080/` 来看一下假数据生成工具是否正常工作了：


```php
namespace App\Controller;

use Faker\Generator;

class HomeController
{
    public function index(Generator $generator)
    {
        return $generator->sentence(128);
    }
}
```

一切正常的话，你会看到一大段随机生成的文字。

> 要深入了解引导程序，请阅读[这篇文档](/zh_CN/framework/bootloaders.md)。

### 路由

默认情况下，路由规则的定义在 `app/src/Bootloader/RoutesBootloader.php` 文件中。对于配置路由而言，你有很多选择。可以把路由指向控制器、控制器方法、控制器组；可以指定默认的匹配参数……

作为实例，我们先创建一个简单的路由，把所有 URL 都指向 `App\Controller\HomeController`:

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>[/<id>]]', new Controller(HomeController::class));
        $route = $route->withDefaults(['action' => 'index']);

        $router->setRoute('home', $route);
    }
}
```

按照上面给出的配置，`action` 和 `id` 都是 URL 中的可选部分（使用了 `[]`），默认的 action 值是 `index`。所以如果访问 `http://localhost:8080/` 或者 `http://localhost:8080/index`，都会执行 `HomeController::index` 方法。

在控制器方法中可以采用方法注入的方式，通过路由参数的名称来访问它们，比如在 `HomeController` 中增加下面的方法：

```php
public function open(string $id)
{
    dump($id);
}
```

然后可以通过 `http://localhost:8080/open/123` 来调用这个方法，`id` 参数会被自动注入。

> 有关路由配置的更多信息，请参阅[路由文档](/zh_CN/http/routing.md)。

### 注解式路由

Spiral 框架默认没有提供开箱即用的注解式路由配置。但是可以通过已有的组件进行简单地配置来[实现它](/zh_CN/cookbook/annotated-routes.md)。

### 注解

我们首先要创建一个简单的注解，稍后可以把它应用到公共控制器方法上：

```php
namespace App\Annotation;

use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation()
 * @Annotation\Target({"METHOD"})
 * @Annotation\Attributes({
 *      @Annotation\Attribute("action", type="string", required=true),
 *      @Annotation\Attribute("verbs", type="array"),
 * })
 */
class Route
{
    /** @var string */
    public $action;

    /** @var string[]|null */
    public $verbs;
}
```

> WEB 应用框架 `apiral/app` 默认已经引入并启用了注解组件 `spiral/annotations`（作为 `spiral/prototype` 的依赖项）。

#### 引导程序

修改 `RoutesBootloader`，让它可以把注解转换为路由。可以使用 `Spiral\Annotations\AnnotationLocator` 这个类来查找代码中可用的路由注解。

```php
namespace App\Bootloader;

use Spiral\Annotations\AnnotationLocator;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(AnnotationLocator $annotationLocator, RouterInterface $router): void
    {
        $methods = $annotationLocator->findMethods(\App\Annotation\Route::class);

        foreach ($methods as $method) {
            $name = sprintf(
                "%s.%s",
                $method->getClass()->getShortName(),
                $method->getMethod()->getShortName()
            );

            $route = new Route(
                $method->getAnnotation()->action,
                new Action(
                    $method->getClass()->getName(),
                    $method->getMethod()->getName()
                )
            );

            $route = $route->withVerbs(...(array)$method->getAnnotation()->verbs);

            $router->setRoute($name, $route);
        }
    }
}
```

#### 控制器

创建好了注解，并在引导程序中实现了注解转换为路由的逻辑之后，就可以在控制器中用注解来定义路由规则了：

```php
namespace App\Controller;

use App\Annotation\Route;

class HomeController
{
    /**
     * @Route(action="/", verbs={"GET"})
     */
    public function index()
    {
        return 'hello world';
    }
    
    /**
     * @Route(action="/open/<id>", verbs={"GET"})
     */
    public function open(string $id)
    {
        dump($id);
    }
}
```

可以在命令行下执行以下命令列出所有已经登记的路由：

```bash
$ php app.php route:list
```

上面示例中的控制器注解，执行这个命令后的输出类似这样：

```bash
+--------+------------+----------------------------------+
| Verbs: | Pattern:   | Target:                          |
+--------+------------+----------------------------------+
| GET    | /          | Controller\HomeController->index |
| GET    | /open/<id> | Controller\HomeController->open  |
+--------+------------+----------------------------------+
```

> 还可以使用更多的路由参数来配置中间件、通用前缀等。

在接下来的示例中，为了简单起见，我们就一直使用注解路由来演示了。

> 如果你在调试过程中觉得日志不够详细，可以在 `App` 中禁用 `ErrorHandleRootLoader` 来查看完整的错误日志。

### 领域内核

连接自定义的控制器拦截器（领域内核）可以用附加功能来丰富应用的领域层。比如改变应用的默认行为、把路由参数自动解析为 Cycle 实体，进行请求参数的过滤和验证，或者实现 @Guard 注解等。

首先创建一个引导程序 `AppBootloader` 来注入拦截器：

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        Domain\FilterInterceptor::class, 
        Domain\CycleInterceptor::class,
        Domain\GuardInterceptor::class,
    ];
}
```

上面的代码通过 `AppBootloader` 引导程序在应用中启用了一些拦截器。记得在 `App.php` 的 `LOAD` 或者 `APP`（推荐）中添加该引导程序。

```php
    protected const APP = [
        Bootloader\RoutesBootloader::class,

        // fast code prototyping
        Prototype\PrototypeBootloader::class,
        Bootloader\FakerBootloader::class,
        Bootloader\AppBootloader::class,
    ];
```

> 要深入了解领域内核，可以查询领域内核的[详细文档](/zh_CN/cookbook/domain-core.md)。

## 数据库脚手架

Spiral 支持通过数据库迁移文件来配置数据库的结构。执行以下命令可以初始化数据库迁移记录表：

```bash
$ php app.php migrate:init
```

之后可以用以下命令来观察数据库迁移记录表的的结构：

```bash
$ php app.php db:list
$ php app.php db:table migrations
```

你可以手工创建数据库迁移文件，或者让 Cycle ORM 帮你生成。

> 请参阅 [数据库迁移文档](/zh_CN/database/migrations.md)，使用[脚手架](/zh_CN/basics/scaffolding.md)组件来辅助人工创建迁移文件。

### 定义 ORM 实体

我们的示例项目以及所有基于 `spiral/app` 创建的项目都自带了 [Cycle ORM](https://cycle-orm.dev) 组件。默认配置下你直接就可以使用注解来配置关系对象实体。

我们先创建 `Post`, `User` 和 `Comment` 三个实体以及它们之间的关系。这里使用脚手架命令来创建：

```bash
$ php app.php create:entity post -f id:primary -f title:string -f content:text -e
$ php app.php create:entity user -f id:primary -f name:string -e
$ php app.php create:entity comment -f id:primary -f message:string
``` 

> Observe the classes generated in `app/src/Database` and `app/src/Repository`.
> 执行命令后请观察项目下 `app/src/Database` 和 `app/src/Repository` 目录，相关的类文件已经自动创建。

Post:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository = "App\Repository\PostRepository")
 */
class Post
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $title;

    /**
     * @Cycle\Column(type = "text")
     */
    public $content;
}

```

User:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository = "App\Repository\UserRepository")
 */
class User
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $name;
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class Comment
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $message;
}
```

如果你不喜欢默认配置下的目录映射关系以及自动生成的文件的头部注释等，可以通过 [脚手架配置](/zh_CN/basics/scaffolding.md)来调整。

> 关于 Cycle 的更多信息，请参考 [Cycle 文档](/zh_CN/cycle/configuration.md)。使用[自定义映射](https://cycle-orm.dev/docs/advanced-timestamp)可以配置自动时间戳。

### 生成数据库迁移文件

通过 cycle 的脚手架命令，可以自动把实体的最新修改自动生成为数据库迁移文件：

```bash
$ php app.php cycle:migrate -v
```

生成的迁移文件存放在 `app/migrations/` 目录下，可以通过命令来执行数据库迁移文件，更新已连接的数据库结构：

```bash
$ php app.php migrate -vv
```

然后通过 `db:list` 命令就可以看到新生成的数据表。

### 创建实体关系
 
通过 [关系注解](https://cycle-orm.dev/docs/annotated-relations) 来定义实体之间的关系。配置 Post 和 Comment 属于 User、Post 拥有多个 Comment。

Post:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;
use Doctrine\Common\Collections\ArrayCollection;

/**
 * @Cycle\Entity(repository = "App\Repository\PostRepository")
 */
class Post
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $title;

    /**
     * @Cycle\Column(type = "text")
     */
    public $content;

    /**
     * @Cycle\Relation\BelongsTo(target = "User", nullable = false)
     * @var User
     */
    public $author;

    /**
     * @Cycle\Relation\HasMany(target = "Comment")
     * @var ArrayCollection|Comment[]
     */
    public $comments;

    public function __construct()
    {
        $this->comments = new ArrayCollection();
    }
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class Comment
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $message;

    /**
     * @Cycle\Relation\BelongsTo(target = "User", nullable = false)
     * @var User
     */
    public $author;

    /**
     * @Cycle\Relation\BelongsTo(target = "Post", nullable = false)
     * @var Post
     */
    public $post;
}
``` 

实体发生变更之后，再次执行 `cycle:migrate` 生成迁移文件，`migrate` 执行迁移：

```bash
$ php app.php cycle:migrate -v
$ php app.php migrate -vv
```

> 提示：你可以通过一条命令生成迁移文件并执行文件：`php app.php cycle:migrate -r`.

我们可以通过 `db:table` 命令来看一下自动生成的外键：

```bash
$ php app.php db:table comments
```

> 切记：修改了实体之后，一定要记得执行 `php app.php cycle:migrate`。

## 服务

我们把业务逻辑单独封装到服务层，与数据层隔离开。先在 `app/src/Service` 目录下创建 `PostService` 服务类。在服务类中，需要用到实现 `Cycle\ORM\TransactionInterface` 接口的实例来进行 post 的数据持久化。

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;

class PostService
{
    private $tr;

    public function __construct(TransactionInterface $tr)
    {
        $this->tr = $tr;
    }

    public function createPost(User $user, string $title, string $content): Post
    {
        $post = new Post();
        $post->author = $user;
        $post->title = $title;
        $post->content = $content;

        $this->tr->persist($post);
        $this->tr->run();

        return $post;
    }
}
```

> 你可以在 `run` 方法后重用事务。

### 原型开发辅助

Spiral 框架强大功能的其中一项就是它的 [原型开发辅助](/zh_CN/basics/prototype.md)。给 `PostService` 类加一个原型注解，指定 `postService` 指向 `PostService`:

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="postService")
 */
class PostService
{
    // ...
}
```

同样的给 `app/src/Repository/UserRepository.php` 中的 `UserRepository` 也加一个原型注解，指定 `users` 指向 `UserRepository`:

```php

```

更新了原型开发辅助相关的配置后，要执行配置命令来收集所有可用的原型类：

```bash
$ php app.php configure
```

> 要获得 IDE 智能提示，需要使用支持的 IDE，比如 Jetbrains PHPStorm.

经过上述配置和操作，现在通过引入 `PrototypeTrait`，就可以在类中直接访问到 `PostService` 对象了（见下面的示例）。

## 控制台命令

经过上面的步骤，我们已经创建了数据实体、实体关系、数据访问仓库、服务等，但是还没有可用的假数据呢。所以，接下来我们先创建三个命令，用来生成假数据：

```bash
$ php app.php create:command seed/user seed:user
$ php app.php create:command seed/post seed:post
$ php app.php create:command seed/comment seed:comment
``` 

生成的命令类，文件都在 `app/src/Command/Seed` 目录下。

### UserCommand

在生成的 "UserCommand" 类中，可以在 `perform` 方法上使用方法注入需要的对象，比如 `Faker\Generator`，然后用它来生成假数据：

```php
// app/src/Command/Seed/UserCommand.php
namespace App\Command\Seed;

use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Faker\Generator;
use Spiral\Console\Command;

class UserCommand extends Command
{
    protected const NAME = 'seed:user';

    protected function perform(TransactionInterface $tr, Generator $faker): void
    {
        for ($i = 0; $i < 100; $i++) {
            $user = new User();
            $user->name = $faker->name;

            $tr->persist($user);
        }

        $tr->run();
    }
}
```

然后执行创建好的命令：

```bash
$ php app.php seed:user
```

### PostCommand

同样地，使用 `PostCommand` 来生成假的文章数据，不过，在这个命令里，我们会使用原型开发辅助提供的属性来代替方法注入。通过原型开发辅助，可以大大提升我们的开发效率。请注意下面的示例代码中，我们直接通过类属性来访问 `postService` 和 `users`（对应 `UserRepository`）。

> 如果你的 IDE 没有智能提示仓库类或者其它服务，请执行 `php app.php configure`.

```php
// app/src/Command/Seed/PostCommand.php
namespace App\Command\Seed;

use Faker\Generator;
use Spiral\Console\Command;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostCommand extends Command
{
    use PrototypeTrait;

    protected const NAME = 'seed:post';

    protected function perform(Generator $faker): void
    {
        $users = $this->users->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];

            $post = $this->postService->createPost(
                $user,
                $faker->sentence(12),
                $faker->text(900)
            );

            $this->sprintf("New post: <info>%s</info>\n", $post->title);
        }
    }
}
```

然后在控制台执行 `seed:post` 命令，加上 `-vv` 参数可以观察到 SQL 查询：

```bash
$ php app.php seed:post -vv
```

开发完成以后，通过脚手架命令可以自动从源代码中移除原型开发辅助：

```bash
$ php app.php prototype:inject -r
```

这条命令会自动修改使用了 `PrototypeTrait` 的类，修改后的代码如下：

```php
namespace App\Command\Seed;

use App\Repository\UserRepository;
use App\Service\PostService;
use Faker\Generator;
use Spiral\Console\Command;

class PostCommand extends Command
{
    protected const NAME = 'seed:post';
    
    /** @var UserRepository */
    private $users;

    /** @var PostService */
    private $postService;

    /**
     * @param UserRepository $users2
     * @param PostService    $postService
     * @param string|null    $name
     */
    public function __construct(UserRepository $users2, PostService $postService, ?string $name = null)
    {
        parent::__construct($name);
        $this->postService = $postService;
        $this->users = $users2;
    }

    protected function perform(Generator $faker): void
    {
        $users = $this->users->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];

            $post = $this->postService->createPost(
                $user,
                $faker->sentence(12),
                $faker->text(900)
            );

            $this->sprintf("New post: <info>%s</info>\n", $post->title);
        }
    }
}
```

> 在开发阶段，你可以在任意代码中使用原型开发辅助，然后在最终上线前通过命令批量移除开发辅助，还可以从项目中删除掉整个扩展组件。

### CommentCommand

生成 comment 假数据时，需要随机指定 user 和 post 来生成关联的评论。可以通过方法注入来获得需要的实例对象。

```php
namespace App\Command\Seed;

use App\Database\Comment;
use App\Repository\PostRepository;
use App\Repository\UserRepository;
use Cycle\ORM\TransactionInterface;
use Faker\Generator;
use Spiral\Console\Command;

class CommentCommand extends Command
{
    protected const NAME = 'seed:comment';

    protected function perform(
        Generator $faker,
        TransactionInterface $tr,
        UserRepository $userRepository,
        PostRepository $postRepository
    ): void {
        $users = $userRepository->findAll();
        $posts = $postRepository->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];
            $post = $posts[array_rand($posts)];

            $comment = new Comment();
            $comment->author = $user;
            $comment->post = $post;
            $comment->message = $faker->sentence(12);

            $tr->persist($comment);
            $tr->run();
        }
    }
}
```

同样地，执行 `seed:comment` 命令生成评论数据：

```bash
$ php app.php seed:comment -vv
```

## 控制器

作为 Restful API 应用，我们需要创建一系列 REST 端点来提供访问数据的 API。首先创建一个简单的控制器，`App\Controller\PostController`, 可以通过脚手架命令来快速创建：

```bash
$ php .\app.php create:controller post -a test -a get -p 
```

> 提示： `-a` 选项可以预创建控制器方法，`-p` 选项可以预加载原型开发辅助扩展。

生成的代码如下：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    public function test()
    {
    }

    public function get()
    {
    }
}
```

### 测试方法

在 Spiral 的控制器方法中，你可以返回不同类型的数据。以下这些类型的数据都是有效的：

- string（字符串）
- PSR-7 Response（PSR-7 规范定义的 Response 对象）
- array（数组会自动作为 JSON 响应给用户）
- JsonSerializable（可以序列化为 JSON 字符串的对象）

> 除了上述默认支持的类型外，还可以通过自定义的[领域核心](/zh_CN/cookbook/domain-core.md)执行自己定义的响应格式转换。在使用了原型开发辅助之后，也可以借由 `$this->response` 辅助属性把数据写入到标准的 PSR-7 响应对象。

为了演示返回数组的实现，我们在返回数据中加了一个 `status` 键，代表响应状态。

```php
/**
 * @Route(action="/api/test/<id>", verbs={"GET"})
 * @param string $id
 * @return array
 */
public function test(string $id)
{
    return [
        'status' => 200,
        'data'   => [
            'id' => $id
        ]
    ];
}
``` 
在浏览器中打开 `http://localhost:8080/api/test/123` 可以看到输出的 JSON 数据。

上面这种方式，我们无法控制 HTTP 响应的状态码，响应的数据里的 `status` 只是 JSON 数据里的响应状态，而 HTTP 响应状态码始终是 200. 因此我们还有另外的方法，比如使用 `ResponseWrapper` 辅助类：

```php
use Psr\Http\Message\ResponseInterface;

// ...

/**
 * @Route(action="/api/test/<id>", verbs={"GET"})
 * @param string $id
 * @return ResponseInterface
 */
public function test(string $id): ResponseInterface
{
    return $this->response->json(
        [
            'data' => [
                'id' => $id
            ]
        ],
        200
    );
}
```

这里向 `json` 函数传入的第二个参数值 `200` 就指定了本次响应的状态码。

> 这个 test 方法仅仅作为这里的演示，后续不再需要。

### 获取文章数据

要从数据库里查询 post 数据，需要 `PostRepository`，可以在控制器的构造函数、`get` 方法中通过方法注入来获得它的实例，也可以通过原型开发辅助提供的 `posts` 缩写（前文有相关介绍）。而需要用户提供的文章 `id`，可以通过路由参数访问到：

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<id:\d+>", verbs={"GET"})
     * @param string $id
     * @return array
     */
    public function get(string $id)
    {
        /** @var Post $post */
        $post = $this->posts->findByPK($id);
        if ($post === null) {
            throw new NotFoundException("post not found");
        }

        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

由于前面我们有配置领域核心，启用了 `CycleInterceptor` 拦截器，因此上面的方法也可以进一步简化，直接在方法注入中依赖 `Post`，`CycleInterceptor` 会用 `id` 进行查询并将对应的 post 注入到我们的方法中：

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<post:\d+>", verbs={"GET"})
     * @param Post $post
     * @return array
     */
    public function get(Post $post)
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

> 实际开发中可以考虑使用视图对象把响应的数据映射到 `JsonSerializable` 形式。

### Post 视图映射

要把数据对象映射到 JSON 格式，可以使用已有的解决方案（例如 `jms/serializer`），或者编写自己的映射实现。下面我们演示一下如何创建一个视图对象把 post 数据转换为 JSON 格式，别忘了之前的知识：通过注释中的 Prototyped 注解可以简化我们的开发。

```php
namespace App\View;

use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Core\Container\SingletonInterface;
use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;

/**
 * @Prototyped(property="postView")
 */
class PostView implements SingletonInterface
{
    use PrototypeTrait;

    public function map(Post $post): array
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }

    public function json(Post $post): ResponseInterface
    {
        return $this->response->json($this->map($post), 200);
    }
}
```

> 执行 `php app.php configure` 命令注册原型开发辅助类，并生成 IDE 代码提示。

然后修改控制器的代码使用刚才创建的 `PostView`:

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<post:\d+>", verbs={"GET"})
     * @param Post $post
     * @return ResponseInterface
     */
    public function get(Post $post): ResponseInterface
    {
        return $this->postView->json($post);
    }
}
```

> 你可以通过浏览器来验证，这里对代码的重构并没有改变 API 的行为。

### 获取文章列表

我们可以直接通过实体仓库（repository）来加载多个文章。在加载多个文章时，可以考虑同时加载可用的文章（post）和它们的作者（post）以减少查询数量。

先在 `PostRepository` 中创建一个 `findAllWithAuthors` 方法：

```php
namespace App\Repository;

use Cycle\ORM\Select;
use Cycle\ORM\Select\Repository;

class PostRepository extends Repository
{
    public function findAllWithAuthor(): Select
    {
        return $this->select()->load('author');
    }
}
``` 

然后在 `PostController` 中创建一个 `list` 方法来调用它：

```php
/**
     * @Route(action="/api/post", verbs={"GET"})
     * @return array
     */
    public function list(): array
    {
        $posts = $this->posts->findAllWithAuthor();

        return [
            'posts' => array_map([$this->postView, 'map'], $posts->fetchAll())
        ];
    }
```

> 你可以访问 `http://localhost:8080/api/post` 查看包含所有文章的 JSON 数据。

### 数据网格（Data Grid）

上面的实现方案有个很大的问题，因为你必须手动对结果进行分页、筛选和排序操作。

不过不用担心，Spiral 提供了[数据网格组件（Data Grid Component）](/zh_CN/component/data-grid.md)来提供数据格式化的操作：

```bash
$ composer require spiral/data-grid-bridge
```

在应用中激活 `Spiral\DataGrid\Bootloader\GridBootloader` 引导程序（参见前文介绍）。

要使用数据网格，首先要指定数据格式，创建 `App\View\PostGrid` 类：

```php
namespace App\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\IntValue;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="postGrid")
 */
class PostGrid extends GridSchema
{
    public function __construct()
    {
        $this->addFilter('author', new Equals('author.id', new IntValue()));

        $this->addSorter('id', new Sorter('id'));
        $this->addSorter('author', new Sorter('author.id'));

        // default limit, available limits
        $this->setPaginator(new PagePaginator(10, [10, 20, 50]));
    }
}
```

> 在上面的示例中，我们为数据网格定义了一个筛选字段、两个排序字段，也指定了分页的实现和分页限制。

然后通过 `Spiral\DataGrid\GridFactory` 把引导程序与方法连接起来：


```php
/**
 * @Route(action="/api/post", verbs={"GET"})
 * @param GridFactory $grids
 * @return array
 */
public function list(GridFactory $grids): array
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return [
        'posts' => array_map(
            [$this->postView, 'map'],
            iterator_to_array($grid->getIterator())
        )
    ];
}
```

> 如果刚才还没有做的话，别忘了每次添加 prototyped 类之后都要执行 `php app.php configure`

数据网格是一个扩展性极强的组件，有大量可以个定制化的选项。默认配置下，数据网格组件从用户请求的查询字符串和请求数据中读取需要的参数值。
 
网址 | 说明
--- | ---
`http://localhost:8080/api/post?paginate[page]=2` | 返回第二页数据，每页显示 10 条（上面指定的默认值）
`http://localhost:8080/api/post?paginate[page]=2&paginate[limit]=20` | 返回第二页数据，每页显示 20 条
`http://localhost:8080/api/post?sort[id]=desc` | 按照 post->id 倒序排列
`http://localhost:8080/api/post?sort[author]=asc` | 按照 post->author->id 正序排列
`http://localhost:8080/api/post?filter[author]=1` | 只返回指定 author->id 相关的 post

在同一个请求 URL 中可以同时使用排序、筛选和分页，也可以一次应用多个筛选条件。

## 校验请求

在交互式程序开发中，永远不要忘记 [校验/验证](/zh_CN/security/validation.md) 用户输入的数据。在 Spiral 中你可以使用更底层的 ValidationInterface 或者 [请求过滤（Request Filters）](/zh_CN/filters/configuration.md) 来对用户输入进行过滤、验证和映射。

我们的示例应用允许用户提交评论，通过[脚手架命令](/zh_CN/basics/scaffolding.md)创建一个 `CommentFilter` 类：

```bash
$ php app.php create:filter comment
```

按照下面的示例来配置过滤类：

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class CommentFilter extends Filter
{
    protected const SCHEMA = [
        'message' => 'data:message'
    ];

    protected const VALIDATES = [
        'message' => ['notEmpty']
    ];

    protected const SETTERS = [
        'message' => 'strval'
    ];

    public function getMessage(): string
    {
        return $this->message;
    }
}
```

### 评论服务类

创建 `App\Service\CommentService`:

```php
namespace App\Service;

use App\Database\Comment;
use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="commentService")
 */
class CommentService
{
    private $tr;

    public function __construct(TransactionInterface $tr)
    {
        $this->tr = $tr;
    }

    public function comment(Post $post, User $user, string $message): Comment
    {
        $comment = new Comment();
        $comment->post = $post;
        $comment->author = $user;
        $comment->message = $message;

        $this->tr->persist($comment);
        $this->tr->run();

        return $comment;
    }
}
```

### 控制器方法

在 `PostController` 控制器中定义发表评论的 `comment` 方法，通过方法注入获得请求过滤类的实例。由于之前我们在领域核心中配置了 `FilterInterceptor` 拦截器，Spiral 框架会保证过滤器的有效性。

参考下面的代码来实现 `comment` 端点，可以对指定的文章发表评论：

```php
/**
 * @Route(action="/api/post/<post:\d+>/comment", verbs={"POST"})
 * @param Post          $post
 * @param CommentFilter $commentFilter
 * @return array
 */
public function comment(Post $post, CommentFilter $commentFilter)
{
    $this->commentService->comment(
        $post,
        $this->users->findOne(), // todo: use current user
        $commentFilter->getMessage()
    );

    return ['status' => 201];
}
```

> 如果需要实现 HTML 格式的 AJAX 原生表单，可以使用 `spiral/toolkit` 扩展。

### 执行和验证

我们直接通过 curl 命令来检查我们的 API 功能是否正确：

```bash
$ curl -X POST -H 'content-type: application/json' --data '{}' http://localhost:8080/api/post/1/comment
``` 

响应内容：

```json
{"status":400,"errors":{"message":"This value is required."}}
```

或者当指定 ID 的文章不存在时响应 404:

```bash
$ curl -X POST -H 'content-type: application/json' --data '{"message":"some comment"}' http://localhost:8080/api/post/9999/comment
``` 

> 要获取 JSON 格式的错误响应，别忘了在请求头信息加上 `accept: application/json`.

发表一个有效的评论：

```bash
$ curl -X POST -H 'content-type: application/json' --data '{"message": "first comment"}' http://localhost:8080/api/post/1/comment
```

> 关于请求过滤的更多信息，请阅读[相关文档](/zh_CN/filters/filter.md). 也可以通过改变[脚手架](/zh_CN/basics/scaffolding.md)的配置来修改生成的请求处理逻辑或者默认命名空间。

## 渲染模板

虽然我们的演示程序是设计 Restful API 应用，不过作为教程，还是向大家介绍一下视图模板渲染的方法。

在 Spiral 中，可以使用 [视图](/zh_CN/views/configuration.md) 和 [Stempler](/zh_CN/stempler/configuration.md) 组件来把数据渲染成 HTML 页面。在渲染列表页时可以直接把数据网格对象传递给模板。

先创建一个控制器方法来处理 `http://localhost:8080/posts` 的请求： 

```php
/**
 * @Route(action="/posts", verbs={"GET"})
 * @param GridFactory $grids
 * @return string
 */
public function all(GridFactory $grids): string
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return $this->views->render('posts', ['posts' => $grid]);
}
```

### 创建布局

创建或者编辑位于 `app/views/layout/base.dark.php` 的布局文件：

```html
<!DOCTYPE html>
<html>
<head>
    <title>${title}</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@4.4.1/dist/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<div class="container">
<block:body/>
</div>
</body>
</html>
```  

### 文章列表页面

然后在 `app/views/posts.dark.php` 创建一个视图模板，继承刚才的布局。

```html
<extends:layout.base title="Posts"/>

<define:body>
    @foreach($posts as $post)
        <div class="post">
            <div class="title">{{$post->title}}</div>
            <div class="author">{{$post->author->name}}</div>
        </div>
    @endforeach
</define:body>
```

> 现在打开 `http://localhost:8080/posts` 就能看到文章列表，可以参考之前讲数据网格时介绍的URL查询参数来控制数据的筛选、排序、分页（比如 `http://localhost:8080/posts?paginate[page]=2`）

### 文章详情页

要实现查看某篇文章和它的评论，在 `PostController` 控制器中创建一个新的控制器方法，通过数据仓库类手动加载文章并预加载它的作者和评论信息。

```php
use Spiral\Http\Exception\ClientException\NotFoundException;
// ...

/**
 * @Route(action="/post/<id:\d+>", verbs={"GET"})
 * @param string $id
 * @return string
 */
public function view(string $id): string
{
    $post = $this->posts->findOneWithComments($id);
    if ($post === null) {
        throw new NotFoundException();
    }

    return $this->views->render('post', ['post' => $post]);
}
```

调用的仓库方法 `findOneWithComments($id)` 代码如下：

```php
public function findOneWithComments(string $id): ?Post
{
    return $this
        ->select()
        ->wherePK($id)
        ->load('author')
        ->load(
            'comments.author',
            [
                'load' => function (Select\QueryBuilder $q) {
                    // last comments at top
                    $q->orderBy('id', 'DESC');
                }
            ]
        )
        ->fetchOne();
}
```

类似地，创建 `app/views/post.dark.php` 模板：


```html
<extends:layout.base title="Posts"/>

<define:body>
    <div class="post">
        <div class="title">{{$post->title}}</div>
        <div class="author">{{$post->author->name}}</div>
    </div>
    <div class="comments">
        @foreach($post->comments as $comment)
            <div class="comment">
                <div class="message">{{$comment->message}}</div>
                <div class="author">{{$comment->author->name}}</div>
            </div>
        @endforeach
    </div>
</define:body>
```

Open the post page using `http://localhost:8080/post/1`.

> 网页的样式以及评论的时间显示就留给读者自己完成了。

### 路由
在之前有关注解式路由的部分，我们在路由引导程序 `RoutesBootloader` 中为每个注解式路由都按照 `ControllerName.methodName` 的格式做了命名。所以在模板中要生成指向 `PostController` 的 `view` 方法的链接时，对应的路由名称为 `PostController.view`:

```html
<extends:layout.base title="Posts"/>
       
<define:body>
   @foreach($posts as $post)
       <div class="post">
           <div class="title">
               <a href="@route('PostController.view', ['id' => $post->id])">{{$post->title}}</a>
           </div>
           <div class="author">{{$post->author->name}}</div>
       </div>
   @endforeach
</define:body>
```

> 有关 Stempler 模板引擎的指令，请阅读 [Stempler 的文档](/zh_CN/stempler/directives.md)。

## 后续步骤

Spiral 框架为你提供了大量预先构建好的强大功能。请参考以下文档以了解更多：

- [用户身份验证](/zh_CN/security/authentication.md)
- [基于角色的访问鉴权](/zh_CN/security/rbac.md)
- [后台任务](/zh_CN/queue/configuration.md)

## 教程源码

本教程的示例项目的最终源码：[https://github.com/spiral/demo](https://github.com/spiral/demo)

代码下载到本地之后，别忘记先执行以下命令：

```bash
$ composer install
$ vendor/bin/spiral get
$ php app.php configure
```
