# HTTP - 路由

Spiral 默认的 [Web 项目框架](https://github.com/spiral/app) 已经包含了配置好的路由组件。但如果你需要在自己构建的项目或其它类型的项目中安装路由组件，可以使用以下命令：

```bash
$ composer require spiral/router
```

然后在激活它的引导程序：

```php
[
    //...
    Spiral\Bootloader\Http\RouterBootloader::class,
]
```

## 默认配置

Web 项目框架的默认配置允许你使用 `/<controller>/<action>` 的模式访问 `App\Controller` 命名空间下的控制器方法。如果你不喜欢这种模式，可以根据下面的文档来修改这一行为。

> 控制器的类名必须有 `Controller` 后缀。

## 路由配置

默认情况下路由组件不需要任何额外配置就能正常使用。但你当然可以在引导程序中通过 `Spiral\Router\RouterInterface` 来创建新的路由。作为示例，我们来创建一个最基础的处理 `/` 的路由：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'home', // 路由名称
            new Route(
                '/', // 匹配模式
                function () { return 'hello world'; } // 路由处理器
            )
        );
    }
}
```

> Route 类接受 `Psr\Http\Server\RequestHandlerInterface` 类型的参数，不管是闭包函数、可调用的类或者 `Spiral\Router\TargetInterface` 的实现都可以。如果你希望路由处理器按需创建，也可以传入类或者类名字符串。

## 闭包处理器

可以传入一个闭包函数作为路由处理器，这种情况下，该函数会收到两个参数：`Psr\Http\Message\ServerRequestInterface` 和 `Psr\Http\Message\ResponseInterface`.

```php
router->setRoute('home', new Route(
    '/<name>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        $response->getBody()->write("hello world");

        return $response;
    }
));
```

## 匹配模式和参数

通过路由匹配模式，可以指定任意数量的必须和可选参数。这些参数会通过 `ServerRequestInterface` 的 `matches` 属性传递给路由处理器。

> 在请求过滤器（Request Filter）中可以通过 `attribute:matches.<key>` 来取回路由参数的值。

在定义路由参数时，可以采用 `<parameter_name:pattern>` 的格式来限制参数格式，这里的 `pattern` 是正则表达式。如果不想对参数格式进行限制，可以省略 `pattern`，只使用 `<parameter_name>`，在这种情况（省略格式限制）下，参数会匹配 `[^\/]+`.

比如简单地添加一个名为 `name` 的参数：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('home', new Route(
            '/<name>',
            function (ServerRequestInterface $request, ResponseInterface $response) {
                return $request->getAttribute('route')->getMatches(); // 返回 JSON ['name' => '']
            }
        ));
    }
}
```

如果希望路由中的参数是可选的，只需用 `[]` 把路由匹配模式对应的部分（包括参数）包裹起来，例如：

```php
$router->setRoute('home', new Route(
    '/[<name>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> 上面的例子可以匹配 `/`，没有传递参数时，name 参数的值是 `null`.

可以指定任意数量的参数，并使他们的中的一部分或全部作为可选。比如，要定义一个类似 `/group/user` 的路由，其中 `/user` 是可选的：

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

在定义路由的时候，可以通过 `Route` 构造函数的第三个参数为路由参数指定默认值：

```php
$router->setRoute('home', new Route(
    '/<group>[/<user>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    },
    [
        'user' => 'default' // 为 user 参数指定默认值 `default`
    ]
));
```

最后再讲一下怎么用 `<parameter:pattern>` 规定参数的格式：

```php
$router->setRoute('home', new Route(
    '/user/<id:\d+>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> 上面的例子只有在 `id` 是数字时才能匹配到这个路由规则。

除了使用正则表达式以外，也可以给参数指定多个预设值：

```php
$router->setRoute('home', new Route(
    '/do/<action:login|logout>',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

> 这个路由规则只匹配 `/do/login` 和 `/do/logout`.

### 匹配主机名

还可以指定路由只匹配特定的域名或者子域名，只要给匹配模式增加 `//` 前缀：

```php
$router->setRoute('home', new Route(
    '//<host>/',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

匹配子域名：

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

可以混合使用主机和路径的匹配规则：

```php
$router->setRoute('home', new Route(
    '//<sub>.domain.com/[<action>]',
    function (ServerRequestInterface $request, ResponseInterface $response) {
        return $request->getAttribute('route')->getMatches();
    }
));
```

### 路由的不变性

所有路由规则都被设计为不可变对象，因此在运行中一旦路由规则已被创建，就不能再改变它们的状态，但可以创建副本并赋予新值。比如在构造函数之外给路由参数指定默认值：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withDefaults([
            'action' => 'default'
        ]));
    }
}
```

### 动词

使用 `withVerbs` 方法，可以让路由只匹配特定的 HTTP 动词（也称为“谓词”）：

```php
namespace App\Bootloader;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/[<action>]', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('get.route',
            $route->withVerbs('GET')->withDefaults(['action' => 'GET'])
        );

        $router->setRoute(
            'post.route',
            $route->withVerbs('POST', 'PUT')->withDefaults(['action' => 'POST'])
        );
    }
}
```

### 中间件

要把中间件关联到特定的路由，可以使用 `withMiddleware` 方法，对应的中间件中可以通过请求对象的 `route` 属性来获取路由参数的值：

```php
namespace App\Bootloader;

use App\Middleware\ParamWatcher;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $route = new Route('/<param>', function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        });

        $router->setRoute('home', $route->withMiddleware(
            ParamWatcher::class
        ));
    }
}
```

示例中的 `ParamWatcher` 中间件代码如下：

```php
namespace App\Middleware;

use Psr\Http\Message\ResponseInterface as Response;
use Psr\Http\Message\ServerRequestInterface as Request;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Http\Exception\ClientException\UnauthorizedException;
use Spiral\Router\RouteInterface;

class ParamWatcher implements MiddlewareInterface
{
    public function process(Request $request, RequestHandlerInterface $handler): Response
    {
        /** @var RouteInterface $route */
        $route = $request->getAttribute('route');

        if ($route->getMatches()['param'] === 'forbidden') {
           throw new UnauthorizedException();
        }

        return $handler->handle($request);
    }
}
```

上面的中间件在匹配到 `/forbidden` 的时候会抛出 `UnauthorizedException` 异常。

> 根据需要，可以给路由关联任意数量的中间件。

## 多路由配置

当有多条路由规则时，系统按照路由的定义顺序进行匹配，首先匹配到的路由生效。因此请避免先定义的路由规则包含后定义的路由规则的情况。

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

// 下面的路由永远不会被触发
$router->setRoute(
    'hello',
    new Route('/hello',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);
```

## 默认路由

Spiral 路由组件支持定义默认（fallback）路由，也可称为“保底”路由。在所有其它路由都检查过且不匹配的情况下，默认路由总会被调用并检查请求是否与符合它的匹配模式。

比如，如果按照下面的方式定义默认路由之后，就不必为每个控制器和方法单独定义路由规则：

```php
$router->setRoute(
    'home',
    new Route('/<param>',
        function (ServerRequestInterface $request, ResponseInterface $response) {
            return $request->getAttribute('route')->getMatches();
        }
    )
);

$router->setDefault(new Route('/', function () {
    return 'default';
}));
```

所有没命中任何规则的路由，都会返回 "default" 字符串，而不是 404 错误。

参考下文，了解如何通过默认路由快速搭建应用程序的路径模式。

## 路由目标（控制器方法）

使用路由最高效的方法是把路由的目标指向控制器中的方法。为了演示所有功能，我们需要在 `App\Controller` 命名空间下定义多个控制器：

```php
namespace App\Controller;

class HomeController
{
    public function index(): string
    {
        return 'index';
    }

    public function other(): string
    {
        return 'other';
    }

    public function user(int $id): string
    {
        return "hello {$id}";
    }
}
```

通过脚手架命令 `php app.php create:controller demo -a test` 创建第二个控制器：

```php
namespace App\Controller;

class DemoController
{
    public function test(): string
    {
        return 'demo test';
    }
}
```

### 指向控制器方法

要把路由的目标指向特定的控制器方法，可以使用 `Spiral\Router\Target\Action` 作为路由处理器：

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'index',
            new Route('/index', new Action(HomeController::class, 'index'))
        );
    }
}
```

结合必须或可选控制器参数使用时，参数会以方法注入的方式传递给控制器方法：

```php
$router->setRoute(
    'user',
    new Route('/user/<id:\d+>', new Action(HomeController::class, 'user'))
);
```

### 方法通配符

在一条路由规则中，可以同时指定其匹配多个控制器方法，只要在路由匹配模式中增加一个 `<action>` 参数。由于要匹配的控制器方法中的某一个需要 `<id>` 参数，而另外的方法不需要，可以指定一个可选的 `<id>` 参数：

```php
$router->setRoute(
    'home',
    new Route('/<action>[/<id>]', new Action(HomeController::class, ['index', 'user']))
);
```

> 上面的路由可以同时匹配 `/index` 和 `/user/`（注意：`/index/1` 和 `/user` 也会匹配）

底层的实现中，上面的路由匹配模式会被编译为 `/^(?P<action>index|user)(?:\/(?P<id>[^\/]+))?$/iu`。这样的实现不只提升了性能，也让不同控制器方法重用相同的匹配模式。

```php
// 匹配 "/index"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'index'))
);

// 匹配 "/other"
$router->setRoute(
    'home',
    new Route('/<action>', new Action(HomeController::class, 'other'))
);

// 匹配 "/test"
$router->setRoute(
    'demo',
    new Route('/<action>', new Action(DemoController::class, 'test'))
);
```

### 指向控制器

通过前面的例子，你会发现其实可以用一条路由规则匹配同一个控制器下的所有方法，但这种情况使用 `Spiral\Router\Target\Controller` 会更合适。以 `Controller` 为目标时，`<action>` 参数不能为空（除非为其指定了默认值）。

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute(
            'home',
            new Route('/home/<action>[/<id>]', new Controller(HomeController::class))
        );
    }
}
```

> 上面的路由对 `/home/index`, `/home/other` 和 `/home/user/1` 都匹配。

上面的路由规则配合路由参数默认值可以让 URL 更短。

```php
$router->setRoute(
    'home',
    (new Route('/home[/<action>[/<id>]]', new Controller(HomeController::class)))
        ->withDefaults(['action' => 'index'])
);
```

> 增加默认值之后，对于 `/home` 也可以匹配了，等同于 `/home/index`. 但要注意，用 `[]` 定义的路由匹配模式中的可选部分必须在匹配模式的最后，不能在必须参数之前。

### 指向命名空间

有时候，你可能想让一个路由规则同时匹配相同命名空间下的多个控制器。这种情况可以使用 `Spiral\Router\Target\Namespaced` 来实现。以 `Namespaced` 作为目标的路由匹配模式必须指定 `<controller>` 和 `<action>` 参数（除非为其指定了默认值）。

定义这样的路由规则时，可以指定目标命名空间和控制器类名后缀：

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Namespaced;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('app', new Route(
            '/<controller>/<action>',
            new Namespaced('App\Controller', 'Controller')
        ));
    }
}
```

> 这条路由规则可以匹配 `/home/index`, `/home/other` 和 `/demo/test`.

如果指定了 `controller` 和 `action` 参数的默认值，那么 URL 中就可以省略它们：

```php
$router->setRoute('app',
    (new Route(
        '[/<controller>[/<action>]]',
        new Namespaced('App\Controller', 'Controller')
    ))->withDefaults([
        'controller' => 'home',
        'action'     => 'index'
    ])
);
```

> 改造后的路由可以匹配 `/`（等于 `/home/index`）、`/home`（等于 `/home/index`）、`/home/index`、`/home/other` 和 `/demo/test`. 但访问 `/demo` 会触发 404 错误，因为 `DemoController` 没有定义默认的 `index` 方法。

Spiral 的 Web 应用框架默认已经定义了上面这条路由规则并将其[作为默认路由](https://github.com/spiral/app/blob/master/app/src/Bootloader/RoutesBootloader.php#L42)。因此你不必为 `App\Controller` 命名空间下的控制器和方法创建任何路由，只要使用 `/controller/action` 这种形式的 URL 就能访问到对应的方法。如果没有指定方法名，那么 `index` 方法会被默认调用，如果没有指定控制器，那么 `HomeController` 会被默认调用。还有一点要说明的是，只有控制器中的访问级别为 `public` 的方法才会被路由调用。

> 在开发完成之后，你可以考虑把默认路由关闭。

### 指向控制器组

一条路由规则同时匹配多个控制器还有另一个替代方法，可以手动指定要匹配的控制器而不是公共命名空间。这种情况需要使用 `Spiral\Router\Target\Group` 作为路由目标，同样必须指定 `<Controller>` 和 `<action>` 参数（除非为它们提供默认值）。

```php
namespace App\Bootloader;

use App\Controller\DemoController;
use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $router->setRoute('app', new Route('/<controller>/<action>', new Group([
            'home' => HomeController::class,
            'demo' => DemoController::class
        ])));
    }
}
```

> 在需要让一级路径匹配多个模块（比如公共页面和管理面板）的时候，这种方法会很有用，因为通常它们会位于不同的命名空间下。

## RESTful

上面介绍过的所有路由目标都支持第三个参数，这个参数规定了系统选择目标方法的行为。如果将这个参数值指定为 `TargetInterface::RESTFUL`，那么系统在调用对应的方法函数时，会自动在方法名称前面加上 HTTP 动词，比如 `index` 会变成 `getIndex` 或者 `postIndex`.

举个例子，假如有如下的控制器：

```php
namespace App\Controller;

class UserController
{
    public function getUser(int $id): string
    {
        return "get {$id}";
    }

    public function postUser(int $id): string
    {
        return "post {$id}";
    }

    public function deleteUser(int $id): string
    {
        return "delete {$id}";
    }
}
```

然后为它定义了这样的路由规则：

```php
$router->setRoute('user', new Route(
    '/user/<id:\d+>',
    new Controller(UserController::class, Controller::RESTFUL),
    ['action' => 'user']
));
```

> 在用不同的 HTTP 动词访问 `/user/1` 的时候，会调用不同的控制器方法。注意：你还是要指定方法名称（例子中的 `user`）。

### 跨路由共享目标

另一个定义 RESTful 路由，或者说给多个控制器定义相同的路由规则的方法是让不同的路由共享相同的路由目标。这种方法要求你的控制器方法名要采用相同的风格。

举个例子，我们可以把不同的 HTTP 动词路由到下面这种命名风格的控制器：

```php
namespace App\Controller;

class UserController
{
    public function load(int $id): string
    {
        return "get {$id}";
    }

    public function store(int $id): string
    {
        return "post {$id}";
    }

    public function delete(int $id): string
    {
        return "delete {$id}";
    }
}
```

我们可以创建形如 `GET|POST|DELETE /v1/<controller>` 的 API, 该 API 会被指向正确的控制器方法。

基础的路由规则看起来类似这样：

```php
$resource = new Route('/v1/<controller>', new Group([
    'user' => UserController::class,
]));
```

然后我们把这个基础的路由注册到不同的 HTTP 动词和控制器方法：

```php
namespace App\Bootloader;

use App\Controller\UserController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Group;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router)
    {
        $resource = new Route('/v1/<controller>/<id>', new Group([
            'user' => UserController::class,
            // 'book' => BookController::class,
            // 'order' => OrderController::class,
        ]));

        $router->setRoute(
            'resource.get',
            $resource->withVerbs('GET')->withDefaults(['action' => 'load'])
        );

        $router->setRoute(
            'resource.store',
            $resource->withVerbs('POST')->withDefaults(['action' => 'store'])
        );

        $router->setRoute(
            'resource.delete',
            $resource->withVerbs('DELETE')->withDefaults(['action' => 'delete'])
        );
    }
}
```

这样就可以给多个资源控制器使用相同的一组路由规则。

## URL 生成

路由除了用来匹配用户访问的 URL，也可以根据给出的路由和参数生成正确的 URL.

```php
$router->setRoute(
    'home',
    new Route('/home/<action>', new Controller(HomeController::class))
);
```

对于上面的路由规则，使用 `RouterInterface` 的 `uri` 方法可以生成正确的 URL:

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', ['action' => 'index']);

    dump((string)$uri); // /home/index
}
```

传入额外的（未在路由匹配模式中定义的）参数，会自动添加到查询字符串：

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
        $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri); // /home/index?page=123
}
```

`uri` 方法返回的值是 `Psr\Http\Message\UriInterface` 的实例而不是字符串：

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'index',
        'page'   => 123
    ]);

    dump((string)$uri->withFragment('hello')); // /home/index?page=123#hello
}
```

> 注意：所有传递给 URL 匹配模式的参数都会 slug 化（空格转为 `-`，大写转为小写）：

```php
use Spiral\Router\RouterInterface;

// ...

public function index(RouterInterface $router)
{
    $uri = $router->uri('home', [
        'action' => 'hello World',
    ]);

    dump((string)$uri); // /home/hello-world
}
```

> 在 Stempler 模板引擎中，可以使用 `@route(name, params)` 指定来生成 URL.
