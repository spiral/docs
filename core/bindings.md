# Core Bindings and Components
Most of Spirals' core components (DBAL, Cache, Session, ...) have a custom string alias like "dbal." 
This makes possible to receive component instance via the magic getter in controller or application class.
```php
class MyControllers extends Controller
{
    public function index()
    {
        dump($this->dbal);
        return $this->view->render('home');
    }
}
```

As any other class or singleton you can request component instance using method or constructor injection:
```php
use Spiral\Components\Encrypter\Encrypter;

class MyControllers extends Controller
{
    public function index(Encrypter $encrypter)
    {
        return $this->view->render('home');
    }
}
```

List of all core component aliases:

Alias     | Component
---       | ---
core      | Will be attached to the application or core instance.
http      | `Spiral\Components\Http\HttpDispatcher`
console   | `Spiral\Components\Console\ConsoleDispatcher`
loader    | `Spiral\Core\Loader`
modules   | `Spiral\Components\Modules\ModuleManager`
file      | `Spiral\Components\Files\FileManager`
debug     | `Spiral\Components\Debug\Debugger`
tokenizer | `Spiral\Components\Tokenizer\Tokenizer`
cache     | `Spiral\Components\Cache\CacheManager`
i18n      | `Spiral\Components\Localization\Translator`
view      | `Spiral\Components\View\ViewManager`
redis     | `Spiral\Components\Redis\RedisManager`
encrypter | `Spiral\Components\Encrypter\Encrypter`
image     | `Spiral\Components\Image\ImageManager`
storage   | `Spiral\Components\Storage\StorageManager`
dbal      | `Spiral\Components\DBAL\DatabaseManager`
orm       | `Spiral\Components\ORM\ORM`
odm       | `Spiral\Components\ODM\ODM`
cookies   | `Spiral\Components\Http\Cookies\CookieStore`
session   | `Spiral\Components\Session\SessionStore`

Some aliases are only available inside `HttpDispatcher` scope:

Alias     | Component
---       | ---
request   | `Spiral\Components\Http\Request`
router    | `Spiral\Components\Http\Router\Router` Only inside router scope.

> All aliases are listed in the Controller and Core doc comments, so your IDE will highlight their methods.

![Code highlighting](../images/bindings.png)

## Extending Core Components
If you want to use your custom implementation of core component, then use container to bind your class.
```php
class MyHttp extends HttpDispatcher
{

}
```
```php
public function bootstrap()
{
    $this->bind('Spiral\Components\Http\HttpDispatcher', 'MyHttp');
}
```
