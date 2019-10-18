# HTTP - Request and Response
You controllers or endpoints will require a way to access active PSR-7 request and ability to generate
the response. In this section we will cover the use of request/responses in controllers setup. 

> The middleware and native PSR-15 handlers can receive PSR-7 objects directly.

## The Request Scope
The fastest way to get access to user request is to create `Psr\Http\Message\ServerRequestInterface` method injection.

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

> Attention, you are **not allowed** to use `ServerRequestInterface` as constructor injection in singletons.

Once request is obtained you can use all read methods available per [PSR-7 Standard](https://www.php-fig.org/psr/psr-7/).

## InputManager
Alternatively you can use context-manager `Spiral\Http\Request\InputManager` which can be stored inside singleton
services/controllers and always point to current user request. This object provides a number of user-friendly methods
to read the incoming data.

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

You can also access `InputManager` via `PrototypeTrait`.

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        // $this->request is alias to $this->input
        dump($this->request->data->all());
    }
}
```

> Note, is it recommended to avoid direct access to `ServerRequestInterface` and `InputManager` unless necessary, 
> use **Request Filters** instead.

You can use input manager to access full array of input data or any specific field by it's name (dot notation is allowed for nested structures). 
Every input structure are represented using `InputBag` class with set of common methods, let's review query accessing as example:

```php
/**
 * @var InputManager $input
 */
 
// Get instance of QueryBag associated with query data
dump($input->query);
 
// Get all query params as array
dump($input->query->all());
 
// Count of query params
dump($input->query->count());
 
// Check if parameter "name" presented in query
dump($input->query->has('name'));
 
// Get value for parameter "name"
dump($input->query->get('name'));
 
// Both get and has methods support dot notation for nested structures
dump($input->query->has('name.subName'));
dump($input->query->get('name.subName'));
 
// Fetch only given query params (no dot notation allowed), only existed records will be returned
dump($input->query->fetch(["name", "nameB"]);

// Fetch only given query params (no dot notation allowed), non existed records will be filled with `null`
dump($input->query->fetch(['name', 'nameB'], true, null);

// In addition query get method has short alias in input manager
dump($input->query('name'));
```


### Input headers
We can use 'headers' input bad and `header` method in `InputManager` to access input headers. HeadersBag has few additions we have to mention:
* HeaderBad will automatically normalize requested header name
* "get" method will implode header values using ',' by default

```php
dump($input->headers->all());

// Will be normalized into "Accept"
dump($input->headers->get('accept')); 

// Return Accept header as array of values
dump($input->headers->get('accept', false));

dump($input->header('accept'));
```

### Cookies
```php
dump($input->cookies->all());
dump($input->cookie('name'));
```

### Server variables
```php
dump($input->server->all());
dump($input->server('name'));
```

> ServerBag will automatically normalize all requested server values. This makes it possible to get value without using all uppercase letters for the names: 

```php
dump($input->server('SERVER_PORT'));
dump($input->server('server-port'));
```

### Post/Data parameters
```php
dump($input->data->all());
dump($input->data('name'));

// An alias
dump($input->post('name'));
```

### Post/Data with fallback to Query parameters
If you want to read value from POST data and then from Query simply use method `input`.

```php
dump($input->input('name'));
```

### PSR7 request attributes
```php
dump($input->attributes->all());
dump($input->attribute('name'));
```

#### Uploaded Files
To get a list of the uploaded files or individual file use the `files` bag and `file` method. Every uploaded file instance is represented
using `UploadedFileInterface` which is part of PSR7.

```php
dump($this->input->files->all());
dump($this->input->file('upload'));
```

> Per PSR all files will be organized to valid hierarchy, which differs from default way php handle uploaded files, you can use dot notation to access nested file instances.

### Simplified methods
In addition to data methods and InputBags `InputManager` provides set of methods to read various properties of active request.

```php
//Request Uri path, will always include leading /
dump($input->path());

//Active request Uri instance
dump($input->uri());

//GET, POST, PUT...
dump($input->method());

//Check if connection made over https
dump($input->isSecure());

//Check request headers to verify that request made over ajax
dump($input->isAjax());

//Check is request expects application/json as response (Accept: application/json)
dump($input->isJsonExpected());

//Receive client ip address (this method uses _SERVER value and may not be correct in some cases).
dump($input->remoteAddress());
```

To access `InputBag` without the use of `__get`:

```php
dump($input->bag('data')->all());
```

## InputInterface
The `InputManager` does not have `get` prefix for it's methods. The reason for that is located in external package
`spiral/filters` which require data source provider via `Spiral\Filters\InputInterface`:

```php
namespace Spiral\Filters;

// ...

interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

The `InputManager` used as default source provider for this interface allowing you to invoke `InputManager` via short notation.
Both approaches will produce the same set of data.

```php
public function index(InputInterface $inputSource, InputManager $inputManager)
{
    dump($inputManager->query('name'));
    dump($inputSource->getValue('query', 'name'));

    dump($inputManager->path());
    dump($inputSource->getValue('path'));
}
```

> You must activate `Spiral\Bootloader\Security\FiltersBootloader` in order to access `Spiral\Filters\InputInterface`.

## Generate Response
You can return an instance of `Psr\Http\Message\ResponseInterface` from your controller, it will be send directly to the user.

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

The PSR-15 handler enabled by default provides the ability to generate the response automatically based on resulted
string or output buffer:

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

Identical to:

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

> We recommend to use output buffer only during the development, to display debug information. Stick to strict
> return types.

## JSON responses
The default PSR-15 also support array and `JsonSerializable` responses which will be converted into JSON:

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

## Response Factory
The proper way to abstract from manual response creation is to use `Psr\Http\Message\ResponseFactoryInterface`:

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

## ResponseWrapper
To generate more complex responses use `ResponseFactoryInterface` wrapper `Spiral\Http\ResponseWrapper` which adds
a number of methods for simpler response generation:

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

You can also access the wrapper via `PrototypeTrait` and property `response`:

```php
namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    Use PrototypeTrait;

    public function index(): ResponseInterface
    {
        // temporary redirect
        return $this->response->redirect('https://google.com', 307);
    }
}
```

The create HTML response:
```php
public function index()
{
    return $this->response->html('hello world');
}
```

To create `application/json` response:

```php
public function index()
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

To send attachment: 

```php
public function index()
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> You can also use StreamInterface as first argument and specify your own mimetype as third option.
