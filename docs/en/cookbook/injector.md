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

We can combine a bootloader and an injector into one instance:

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

Now, any model request will include a unique id, for example in the controller method:

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

## Enum Injectors

The `Spiral Boot` component provides `Spiral\Boot\Injector\InjectableEnumInterface` that allows you to create `Enums` 
that can then be automatically injected via `Container` with specific values.

Let's create an `Enum`, with which we can easily determine the environment of our application.

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum AppEnvironment: string implements InjectableEnumInterface
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }

    public function isTesting(): bool
    {
        return $this === self::Testing;
    }

    public function isLocal(): bool
    {
        return $this === self::Local;
    }

    public function isStage(): bool
    {
        return $this === self::Stage;
    }

    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

The attribute `ProvideFrom` allows you to specify a `method` by which you can determine the current value of the `Enum`.
The method will be called when the Container requests the Enum. The required dependencies from the Container will be 
injected into it.

### Usage

The `Container` will automatically inject an Enum with the correct value.

```php
class MigrateCommand extends Command 
{
     const NAME = '...';

     public function perform(AppEnvironment $appEnv): int
     {
           if ($appEnv->isProduction()) {
                 // Deny
           }
           // ...
     }
}
```

Or get `Enum` from `Container`:

```php
$appEnv = $container->get(AppEnvironment::class);

dump($appEnv->isProduction());
```

> **Note**
> The `AppEnvironment` is already available and has been provided as an example only.
