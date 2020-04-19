# HTTP - 请求和响应

在 MVC 架构的 WEB 应用中，控制器或路由端点需要根据当前活动的 PSR-7 请求来执行相应的业务逻辑代码。本章节就向大家介绍在 Spiral Web 应用中的请求和响应操作。

> 中间件和原生 PSR-15 处理器会直接接收到代表当前请求的 PSR-7 对象。

## 请求作用域

读取用户请求的最快捷的方式是通过方法注入直接获得实现 `Psr\Http\Message\ServerRequestInterface` 接口的实例：

```php
namespace App\Controller;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Container\SingletonInterface;

class HomeController implements SingletonInterface
{
    public function index(ServerRequestInterface $request)
    {
        dump($request->getHeaders());
    }
}
```

> 注意：不允许在单例对象的构造函数中注入 `ServerRequestInterface`，存在用户信息泄露的危险。

获得请求对象后，你就可以调用 [PSR-7 标准](https://www.php-fig.org/psr/psr-7/) 定义的所有读取方法了。

## 请求上下文管理器（InputManager）

除了 PSR-7 请求对象外，你也可以使用 Spiral 提供的请求上下文管理器 `Spiral\Http\Request\InputManager`，这是可以存储在单例服务、控制器中的对象，而且通过它访问到的始终是当前活动的用户请求。除此之外，它比 PSR-7 请求对象还提供了更多方便快捷的方法来读取传入数据。

```php
namespace App\Controller;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Request\InputManager;

class HomeController implements SingletonInterface
{
    private $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index()
    {
        dump($this->input->query->all());
    }
}
```

如果使用了 `原型开发辅助（PrototypeTrait）`，不需要方法注入就可以快捷地调用 `InputManager`，通过 `$this->request`：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        // $this->request 是 $this->input 的别名
        dump($this->request->data->all());
    }
}
```

> 请注意，除非有必要，否则不建议直接访问 `ServerRequestInterface` 或者 `InputManager`，最好使用 `请求过滤（Request Filter）` 来过滤和获取用户传入数据。

通过 `InputManager` 辅助对象可以访问输入数据中的所有数组（Query, Cookie, Header, Post Data 等），或者其它特定的字段（对于对象中的嵌套数组和对象，可以用点表示法（dot notation）访问深层次的字段）。每个输入结构都用带有一组常用方法的 `InputBag` 类表示，以查询参数为例：

```php
/**
 * @var InputManager $input
 */

// 获得 QueryBag 实例，对应 URL 查询参数传递的数据
dump($input->query);

// 获得查询参数数组
dump($input->query->all());

// 获得查询参数个数
dump($input->query->count());

// 检查名为 "name" 的查询参数是否存在
dump($input->query->has('name'));

// 获得名为 "name" 的查询参数
dump($input->query->get('name'));

// get 和 has 方法都支持用点表示法获得嵌套结构中的数据
dump($input->query->has('name.subName'));
dump($input->query->get('name.subName'));

// 只返回指定名称的查询参数（不允许点表示法），只有指定的且存在的参数会返回
dump($input->query->fetch(["name", "nameB"]);

// 只返回指定名称的查询参数（不允许点表示法），不存在的参数同样返回，但值被设置为 `null`
dump($input->query->fetch(['name', 'nameB'], true, null);

// 查询参数的 get 方法可以用简短的别名方式调用，更加简洁
dump($input->query('name'));
```

### 请求头信息

如果要读取请求头信息，可以用 `InputManager` 的 `header` 方法，返回的是 `HeadersBag`，它也包含了一些有用的方法：

- HeaderBag 会自动规范头信息名称，因此开发者可以不用关注大小写的问题
- "get" 方法默认会自动把包含 "," 的头信息值用 implode 方法进行拆分

```php
// 所有请求头信息
dump($input->headers->all());

// "accept" 会自动规范成 "Accept"
dump($input->headers->get('accept'));

// 返回的 Accept 头信息会转为数组（浏览器大部分时候会附带多个 accept 值）
dump($input->headers->get('accept', false));

// HeaderBag 的 "get" 方法也有别名
dump($input->header('accept'));
```

### Cookies

```php
// 包含所有 cookie 信息的数组
dump($input->cookies->all());
dump($input->cookie('name'));
```

### 服务器变量

```php
dump($input->server->all());
dump($input->server('name'));
```

> 注意：ServerBag 会自动规范化请求中包含的所有服务器变量，因此在调用相关的变量值时传入的键名不必区分大小写。

```php
dump($input->server('SERVER_PORT'));
dump($input->server('server-port'));
```

### Post/Data 请求数据

```php
dump($input->data->all());
dump($input->data('name'));

// 别名
dump($input->post('name'));
```

### Post/Data 数据兼容查询字符串

有时候，我们会希望在 Post 数据中找不到需要的数据时，自动从查询字符串（Query）中查询，这种情况下，只要使用 `input` 方法就可以：

```php
dump($input->input('name'));
```

### PSR7 请求属性

```php
dump($input->attributes->all());
dump($input->attribute('name'));
```

### 上传文件

要获取上传的所有文件，或者单个文件，可以通过 `files` 包以及 `file` 方法。每一个成功上传的文件都表示为实现 `UploadFileInterface` 接口的实例，它其实是 PSR7 标准的一部分：

```php
dump($this->input->files->all());
dump($this->input->file('upload'));
```

> 根据 PSR 规范，所有文件都按照逻辑层次结构进行组织，这与 PHP 处理上传文件的默认方式有所不同。你可以使用点表示法来访问嵌套的文件实例。

### 简化方法

在 data 方法和 InputBags 之外，`InputManager` 提供了大量用于读取当前请求的各种属性的简化方法：

```php
// 请求的 Uri 中的路径，总是包含开头的 "/"
dump($input->path());

// 当前请求的 Uri
dump($input->uri());

// 当前请求的方法（动词），GET, POST, PUT 等
dump($input->method());

// 判断当前请求是否采用 https
dump($input->isSecure());

// 通过请求头信息判断这是否 Ajax 请求
dump($input->isAjax());

// 判断当前请求是否希望获得 `application/json` 类型的响应（Accept: application/json）
dump($input->isJsonExpected());

// 获取客户 ip 地址（这个方法使用 _SERVER 里的值，有时候得到并不是正确的用户 ip）
dump($input->remoteAddress());
```

不通过 `__get` 魔术方法获取 `InputBag`，可以这样做：

```php
dump($input->bag('data')->all());
```

## InputInterface

`InputManager` 这个辅助类中的方法没有 `get` 前缀。原因是在外部组件 `spiral/filters` 中，需要通过 `Spiral\Filters\InputInterface` 来提供数据源。

```php
namespace Spiral\Filters;

// ...

interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

你可以通过 `InputInterface` 的短符号来调用 `InputManager` 的方法。两种方式返回的数据是相同的。

```php
public function index(InputInterface $inputSource, InputManager $inputManager)
{
    dump($inputManager->query('name'));
    dump($inputSource->getValue('query', 'name'));

    dump($inputManager->path());
    dump($inputSource->getValue('path'));
}
```

这一实现主要用于把输入数据映射到请求过滤器（Request Filter）。

> 如果要访问 `Spiral\Filter\InputInterface`，必须激活 `Spiral\Bootloader\Security\FiltersBootloader` 引导程序。

## 生成响应

在控制器方法中可以返回任何实现 `Psr\Http\Message\ResponseInterface` 接口的实例，它们会被直接发送给用户。

```php
namespace App\Controller;

use Nyholm\Psr7\Response;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    public function index(): ResponseInterface
    {
        $r = new Response(200);
        $r->getBody()->write("hello world");

        return $r;
    }
}
```

Spiral 默认启用的 PSR-15 处理器提供了基于返回字符串或输出缓冲区的内容自动生成响应对象的能力：

```php
namespace App\Controller;

class HomeController
{
    public function index(): string
    {
        return "hello world";
    }
}
```

下面的示例可以达到与上面相同的结果：

```php
namespace App\Controller;

class HomeController
{
    public function index()
    {
        echo "hello world";
    }
}
```

> 为了采用严格的返回数据类型，建议只在开发阶段为了显示 debug 信息才使用输出缓冲区。

## JSON 响应

除了字符串，默认的 PSR-15 处理器还支持数组和 `JsonSerializable` 响应，并自动将它们转换为 JSON:

```php
namespace App\Controller;

class HomeController
{
    public function index(): array
    {
        return [
            'status' => 200,
            'data'   => ['some' => 'json']
        ];
    }
}
```

## 响应工厂

对手动创建响应进行抽象的正确方式是使用 `Psr\Http\Message\ResponseFactoryInterface`:

```php
namespace App\Controller;

use Psr\Http\Message\ResponseFactoryInterface;
use Psr\Http\Message\ResponseInterface;

class HomeController
{
    public function index(ResponseFactoryInterface $responseFactory): ResponseInterface
    {
        $response = $responseFactory->createResponse(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

## 响应包装类

为了生成更复杂的响应，可以使用 Spiral 提供的响应包装类 `Spiral\Http\ResponswWrapper`，它对 `ResponseFactoryInterface` 进行了实现和包装，增加了一些方法，用于简化响应的处理：

```php
namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Spiral\Http\ResponseWrapper;

class HomeController
{
    public function index(ResponseWrapper $r): ResponseInterface
    {
        return $r->attachment(
            __FILE__,
            'controller.php'
        )->withAddedHeader('Key', 'value');
    }
}
```

通过原型开发辅助 `PrototypeTrait` 的 `response` 属性，可以快速访问响应包装类：

```php
namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    Use PrototypeTrait;

    public function index(): ResponseInterface
    {
        // 临时重定向
        return $this->response->redirect('https://google.com', 307);
    }
}
```

创建 HTML 格式的响应：

```php
public function index()
{
    return $this->response->html('hello world');
}
```

创建 JSON 响应：

```php
public function index()
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

发送附件：

```php
public function index()
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> 提示：你也可以用 StreamInterface 的实现作为第一个参数，并在第三个参数指定要使用的 mime-type.
