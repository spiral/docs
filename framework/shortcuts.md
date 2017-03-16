# SchemaLocator shortcuts
Some components can be accessed from container skipping usage of `get` method. Use `SharedTrait` trait in order to enable `__get` proxy.

```php
public function indexAction()
{
    //"db" points to `Database` via shortcut
    dump($this->db);
}
```

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

> Shortcuts will work in a static IoC scope when no local container is available.
