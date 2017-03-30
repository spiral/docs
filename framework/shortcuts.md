# SchemaLocator shortcuts
Some components can be accessed from container skipping usage of `get` method. Use `SharedTrait` trait in order to enable `__get` proxy.

```php
public function indexAction()
{
    //"db" points to `Database` via shortcut
    dump($this->db);
}
```

> SharedTrait already used in base Controller and Service classes.

#### Shortcuts defined in `Spiral\Core\Bootloaders\SpiralBootloader`
Common Components:

Binding/Alias     | Component                               
---               | ---      
memory            | Spiral\Core\MemoryInterface
container         | Spiral\Core\ContainerInterface *(Factory + Interop Container)*
logs              | Spiral\Debug\LogsInterface
files             | Spiral\Files\FilesInterface
tokenizer         | Spiral\Tokenizer\TokenizerInterface
locator           | Spiral\Tokenizer\ClassesInterface
invocationLocator | Spiral\Tokenizer\InvocationsInterface

Security:

Binding/Alias   | Component  
---             | ---  
permissions     | Spiral\Security\PermissionsInterface
rules           | Spiral\Security\RulesInterface
actor           | Spiral\Security\ActorInterface

Framework Components:

Binding/Alias   | Component  
---             | ---  
encrypter       | Spiral\Encrypter\EncrypterInterface
translator      | Spiral\Translator\TranslatorInterface
views           | Spiral\Views\ViewManager
http            | Spiral\Http\HttpDispatcher  
console         | Spiral\Console\ConsoleDispatcher
storage         | Spiral\Storage\StorageInterface 

Databases:

Binding/Alias   | Component  
---             | ---
dbal            | Spiral\Database\DatabaseManager 
odm             | Spiral\ODM\ODM
orm             | Spiral\ORM\ORM
db              | Spiral\Database\Entities\Database *(default database)*
mongo           | Spiral\ODM\Entities\MongoDatabase *(default database)*

HTTP Scope Only:

Binding/Alias   | Component  
---             | ---
paginators      | Spiral\Pagination\PaginatorsInterface
request         | Psr\Http\Message\ServerRequestInterface 
session         | Spiral\Session\SessionInterface 
input           | Spiral\Http\Input\InputManager 
cookies         | Spiral\Http\Cookies\CookieQueue
response        | Spiral\Http\Response\ResponseWrapper

Only inside Routes:

Binding/Alias   | Component  
---             | ---
router          | Spiral\Http\Routing\RouterInterface 
route           | Spiral\Http\Routing\RouteInterface 

> Shortcuts will work in a static IoC scope when no local container is available. Read more about scopes [here](components.md).

Use proper IDE to get maximum from shortcuts:
 
![Short Bindings](https://raw.githubusercontent.com/spiral/guide/master/resources/virtual-bindings.gif)

Consider shortcuts as temporary replacement for a proper class dependency or as a way to make some of application services available from outside:

```php
$app = MyBlogApp::init(...);
foreach($app->postsService->getTodayPosts() as $post) {
    echo $post->getTitle();
}
```

### SharedTrait
You can enable shortcuts in any of your class by using `SharedTrait` which defines `__get` method. Such trait already used by Controllers, Commands and Services.

```php
use SharedTrait;

public function indexAction(ViewsInterface $views)
{
    dump($this->views === $this->container->get('views'));
    dump($this->views === $views);
    
    echo $this->views->render(...);
}
```

> Attention, `SharedTrait` requires method `iocContainer()` to be defined. Extend `Componen` class in order to enable such method in your classes or define it manually.
