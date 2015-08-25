# Service Classes
Spiral provides idea to put your application business and database logic into set of specific classes called "Services", such classes can be responsible for almost
every operation in your application. You are not required to use services, hovewer they may significantly simplfify definition of your controller and data entities
and provide better base for unit testing.

> Every Controller is Service.

## Scaffolding
You can generate empty service using CLI command "create:service name", this command will allow you to define set of methods, dependecies with other services
or even pre-generate code required to manage some data entity like ORM Record or ODM Document. Let's looks on some example (`create:service some -m doSomething`):

```php
class SomeService extends Service implements SingletonInterface
{
    //Declaring to IoC that class must be constructed only once
    const SINGLETON = self::class;

    public function doSomething()
    {
    }
}
```

As you can notice, by default new service statet as singleton, meaning any requested dependency in controllers or other services will be resolved with one instance.
If you wish to make mortal service, simply run command with flag `--mortal` or `-s`. You now can use your service inside any of your contrroller by requesting it
as class or method dependency.

```php
public function index(SomeService $some)
{
    $some->doSomething();
    //...
}
```

## Services and Container
One of important pieces around service implementation, that every service requesting `ContainerInterface` in it's constructor, this provides you ability to 
request dependcies in your code using `$container` property.

```php
class SomeService extends Service implements SingletonInterface
{
    //Declaring to IoC that class must be constructed only once
    const SINGLETON = self::class;

    public function doSomething()
    {
        dump($this->container->get(SomeClass::class));
    }
}
```

If you with to declate dependcy on class level, you can simply create `init` method with list of needed instances (`init` method this is only required due Service aready have constructor with `ContainerInterface` dependency). You can also specify dependendencies with other services while scaffolding using `-d` flag: `create:service other -d some -m someMethod`

```php
class OtherService extends Service implements SingletonInterface
{
    //Declaring to IoC that class must be constructed only once
    const SINGLETON = self::class;

    /**
     * @var SomeService
     */
    protected $some = null;

    /**
     * @param SomeService $some
     */
    public function init(SomeService $some)
    {
        $this->some = $some;
    }

    public function someMethod()
    {
        dump($this->some->doSomething());
    }
}
```

## Accessing core components
In many cases your services might want to talk to framework components, for example databases, translator and etc. You can still request such components as init method
dependencies, hovewer spiral provides set of short bindings which can be resolved using short name (for example we can get instance of `HttpDispatcher` using `http` binding), such resolution can be performed via magic `__get` method.

```php
   public function someMethod()
   {
       dump($this->http->perform($this->request->withUri(...)));
   }
```

> You can use same technique in Controllers. If you will decide to use specific component in your service, you can simply create variable named as it's binding which will replace magic `__get`.

Check this list of default component bindings:

| Binding   | Compoment                               | 
| ---       | ---                                     |
| core      | Spiral\Core\Core or Application         |
| loader    | Spiral\Core\Components\Loader           |
| modules   |  Spiral\Modules\ModuleManager           |
| debugger  | Spiral\Debug\Debugger                   |
| console   | Spiral\Console\ConsoleDispatcher        |
| http      |   Spiral\Http\HttpDispatcher            |
| cache     | Spiral\Cache\CacheProvider              |
| dbal      | Spiral\Database\DatabaseProvider        |
| encrypter | 'Spiral\Encrypter\Encrypter             |
| files     | Spiral\Files\FileManager                |
| odm       | Spiral\ODM\ODM                          |
| orm       | Spiral\ORM\ORM                          |
| session   | Spiral\Session\SessionStore             |
| storage   | Spiral\Storage\StorageManager           |
| tokenizer | Spiral\Tokenizer\Tokenizer              |
| i18n      | Spiral\Translator\Translator            |
| views     | Spiral\Views\ViewManager                |

Following bindings will be available only when application executing inside `HttpDispatcher->perform()` method (actually inside `MiddlewarePipeline`):
| Binding   | Compoment                               | 
| ---       | ---                                     |
| cookies   | Spiral\Http\Cookies\CookieManager       | 
| router    | Spiral\Http\Routing\Router              | 
| request   | Psr\Http\Message\ServerRequestInterface |
| input     | Spiral\Http\InputManager                |

> There is no static Facades in Spiral.

## Communication with DataEntities (ORM and ODM)
In addition to generic service functionality spiral provides ability to create set of methods specific to some data entity, this can be done by providing `-e` flag
to `create:service` command. Let's say that we have database entity `User`, we can generate service for such user using `create:service user -e user`:

```php
class UserService extends Service implements SingletonInterface
{
    //Declaring to IoC that class must be constructed only once
    const SINGLETON = self::class;

    /**
     * Create new blank User. You must save entity using save method.
     *
     * @param array|\Traversable $fields Initial set of fields.
     * @return User
     */
    public function create($fields = [])
    {
        return User::create($fields);
    }
        
    /**
     * Save User instance.
     *
     * @param User  $user
     * @param bool  $validate
     * @param array $errors Will be populated if save fails.
     * @return bool
     */
    public function save(User $user, $validate = true, &$errors = null)
    {
        if ($user->save($validate)) {
            return true;
        }

        $errors = $user->getErrors();

        return false;
    }

    /**
     * Delete User.
     *
     * @param User $user
     * @return bool
     */
    public function delete(User $user)
    {
        return $user->delete();
    }

    /**
     * Find User it's primary key.
     *
     * @param mixed $primaryKey
     * @return User|null
     */
    public function findByPK($primaryKey)
    {
        return User::findByPK($primaryKey);
    }

    /**
     * Find User using set of where conditions.
     *
     * @param array $where
     * @return User[]|Selector
     */
    public function find(array $where = [])
    {
        return User::find($where);
    }
}
```

Now you can use this service in your controller to find, fetch save and delete entities of `User`, for example:

```php
public function index(UserService $users)
{
    dump($users->find()->count());
    dump($users->find()->findOne());
}
```