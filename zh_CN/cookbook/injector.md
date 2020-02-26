# 速查手册 - 注入器

通过注入器接口，可以控制任何接口的具体实现，或者抽象类的实现子类的创建过程。下面的示例演示了如何在创建每个类示例时为它分配一个唯一 id，尽管在分配 id 的时候还不知道要创建的是具体是哪一个子类的实例：

```php
// 要控制的抽象类
abstract class Model
{
    public $id;
    public $context;
}
```

可以把注入器和引导程序合并到一起：

```php
namespace App\Bootloader;

use App\Model;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;

class ModelBootloader extends Bootloader implements Container\InjectorInterface, Container\SingletonInterface
{
    // 全局唯一 id
    private $id = 0;

    public function boot(Container $container)
    {
        $container->bindInjector(Model::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        // 创建对象实例
        $model = $class->newInstance();
        $model->context = $context;
        // 全局唯一 id 自增，并把值分配给创建的对象实例
        $model->id = $this->id++;

        return $model;
    }
}
```

> 别忘了激活你的引导程序。

多子类也是可以的：

```php
namespace App;

class A extends Model
{

}
```

以及

```php
namespace App;

class B extends Model
{

}
```

通过以上的代码，任何针对 `App\Model` 的依赖都会获得一个具有唯一 id 的实例，比如在控制器方法中依赖它：

```php
namespace App\Controller;

use App\A;
use App\B;

class HomeController
{
    public function index(A $first, B $second)
    {
        assert($first->id > $second->id);

        assert($first->context === 'first');
        assert($first->context === 'second');
    }
}
```

> 在注入器中可以调用 `make` 和 `get` 方法，但最好不要这样做。
