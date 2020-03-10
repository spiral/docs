# Cookbook - Injectors
You are able to control the creation process of any interface or abstract class 
children using injection interface. Following demonstrates how to create class
instance with assign it unique value, no matter what children implements it.

```php
abstract class Model
{
    public $id;
    public $context;
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
    private $id = 0;

    public function boot(Container $container)
    {
        $container->bindInjector(Model::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null)
    {
        $model = $class->newInstance();
        $model->context = $context;
        $model->id = $this->id++;

        return $model;
    }
}
```

> Do not forget to activate the bootloader.

Multiple children is allowed:

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

Now, any model request will include unique id, for example in controller method:

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

> You can call `make` and `get` methods inside the injectors... but instead use the force. 
