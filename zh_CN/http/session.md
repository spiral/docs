# HTTP - Session

默认的 WEB 应用框架已经默认内置和启用了 Session 组件。

如果在自行构建的项目中需要启用 Session 组件，首先通过 composer 安装 `spiral/session` 包，然后把引导程序 `Spiral\Bootloader\Http\SessionBootloader` 添加到应用中。

## SessionInterface

用户的 Session 可以通过特定的上下文对象 `Spiral\Session\SessionInterface` 访问：

```php
use Spiral\Session\SessionInterface;

// ...

public function index(SessionInterface $session)
{
    $session->resume();
    echo $session->getID();
}
```

> 注意，不要在单例对象中存储对 Session 的引用。参见下面的解决办法。

## Session 分区

默认情况下，禁止直接操作 Session，而是分配给隔离的命名分区，通过命名分区的 `set`、`get`、`delete` 等方法去操作 Session. 通过 Session 对象的 `getSection` 方法可以获取指定的分区：

```php
public function index(SessionInterface $session)
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Session 作用域

为了简化在单例服务或者控制器中的 Session 操作，可以使用 `Spiral\Session\SessionScope`. 该对象也可以通过原型开发辅助的 `session` 属性访问。与 `SessionInterface` 不同，这个对象可以在单例服务中使用，它始终指向当前请求关联的 Session 上下文：

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->session->getSection('cart')->getAll());
    }
}
```

## Session 生命周期

Session 会在第一次数据请求时自动启动，并在用户请求离开 `SessionMiddleware` 的时候自动提交。如果需要手动控制 Session, 可以使用 `SessionInterface` 对象中的对应方法。

> 说明：SessionScope 是 SessionInterface 的完全实现。

### Session 恢复

如果需要手工恢复或创建 Session，可以用 `resume` 方法：

```php
$this->session->resume();
```

### Session 提交

如果需要手工提交修改或者关闭 Session，可以用 `commit` 方法：

```php
$this->session->commit();
```

### 退出（放弃修改）

如果不打算提交对 Session 的修改并关闭它，可以用 `abort` 方法：

```php
$this->session->abort();
```

### 获取 Session ID

读取 Session ID 的方法（仅在 Session 已恢复后可用）：

```php
dump($this->session->getID());
```

检查 Session 是否已经启动：

```php
dump($this->session->isStarted());
```

### Session 删除

如果要删除 Session，使用 `destroy` 方法：

```php
$this->session->destroy();
```

### 重新生成 ID

如果想保留 Session 的内容，但重新分配一个 ID：

```php
$this->session->regenerateID();
```

## 个性化配置

如果要调整 Session 组件的设置，可以创建配置文件 `app/config/session.php` 来对需要的配置项进行修改：

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

### 自定义 Session 处理器

Session 组件是基于 PHP 原生的 Session 实现开发的。默认情况下，Session 存放于项目下的 `runtime/session` 目录下。

如果希望采用文件系统之外的方法存储 Session, 可以使用任何[实现了 `SessionHandlerInterface` 接口的处理器](https://www.php.net/manual/en/class.sessionhandlerinterface.php)。

```php
<?php
return [
    'handler' => MyHandlerClass::class
];
```

> 你可以用 Autowire 代替类名来配置附加参数。
