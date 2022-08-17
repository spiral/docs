# Cookbook - Injectors

You can control the creation process of any interface or abstract class children using an injection interface. This
article demonstrates how to create a class instance and assign a unique value to it, no matter what children implement
it.

```php
abstract class Model
{
    public int $id;
    public string $context;
}
```

We can combine bootloader and injector into one instance:

```php
namespace App\Bootloader;

use App\Model;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\Container;

class ModelBootloader extends Bootloader implements Container\InjectorInterface, Container\SingletonInterface
{
    private int $id = 0;

    public function boot(Container $container): void
    {
        $container->bindInjector(Model::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null): Model
    {
        $model = $class->newInstance();
        $model->context = $context;
        $model->id = $this->id++;

        return $model;
    }
}
```

> **Note**
> Do not forget to activate the bootloader.

Multiple children are allowed:

```php
namespace App;

class A extends Model
{

}
```

and

```php
namespace App;

class B extends Model
{

}
```

Now, any model request will include unique id, for example in the controller method:

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

> **Note**
> You can call `make` and `get` methods inside the injectors... but instead use the force. 
