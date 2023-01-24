# HTTP — Request and Response

Your controllers or endpoints will need a way to access active PSR-7 request and an ability to generate the response. In
this section, we will cover the use of requests/responses in the MVC setup.

> **Note**
> The middleware and native PSR-15 handlers can receive PSR-7 objects directly.

## The Request Scope

The fastest way to get access to the user request is to create `Psr\Http\Message\ServerRequestInterface` method
injection.

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

use Psr\Http\Message\ServerRequestInterface;
use Spiral\Core\Container\SingletonInterface;

class HomeController implements SingletonInterface
{
    public function index(ServerRequestInterface $request): void
    {
        dump($request->getHeaders());
    }
}
```

> **Note**
> You are **not allowed** to use `Psr\Http\Message\ServerRequestInterface` as a constructor injection in 
> singletons.

Once the request is obtained, you can use it to all read methods available
per [PSR-7 Standard](https://www.php-fig.org/psr/psr-7/).

## InputManager

Alternatively, you can use context-manager `Spiral\Http\Request\InputManager`, which can be stored inside singleton
services/controllers, and always point to the current user request. This object provides several user-friendly methods
to read the incoming data.

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

use Spiral\Core\Container\SingletonInterface;
use Spiral\Http\Request\InputManager;

class HomeController implements SingletonInterface
{
    private InputManager $input;

    public function __construct(InputManager $input)
    {
        $this->input = $input;
    }

    public function index(): void
    {
        dump($this->input->query->all());
    }
}
```

You can also access `Spiral\Http\Request\InputManager` via `Spiral\Prototype\Traits\PrototypeTrait`.

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        // $this->request is alias to $this->input
        dump($this->request->data->all());
    }
}
```

> **Note**
> It is recommended to avoid direct access to `Psr\Http\Message\ServerRequestInterface` 
> and `Spiral\Http\Request\InputManager` unless necessary, use **Request Filters** instead.

You can use `Spiral\Http\Request\InputManager` to access the full array of input data or any specific field by its name 
(dot notation is allowed for nested structures). Every input structure is represented 
using `Spiral\Http\Request\InputBag` class with a set of common methods. Let's review query accessing as an example:

```php
/** @var \Spiral\Http\Request\InputManager $input */

// Get instance of QueryBag associated with query data
dump($input->query);
 
// Get all query params as array
dump($input->query->all());
 
// Count of query params
dump($input->query->count());
 
// Check if parameter "name" is presented in query
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

> **Note**
> In the situation when Input bag has a key with `null` value 
> `new \Spiral\Http\Request\InputBag(['name' => null]);`, the method `$input->query->has('name')` will return `true` but `isset($input->query['name])` will return `false`.
 
### Input headers

We can use the '**headers**' input bag and `header` method in `Spiral\Http\Request\InputManager` to access input 
headers. `Spiral\Http\Request\HeadersBag` has a few additions we have to mention:

* `Spiral\Http\Request\HeadersBag` will automatically normalize a requested header name
* "**get**" method will implode header values using ',' by default

```php
/** @var \Spiral\Http\Request\InputManager $sinput */

// Get all headers as array
dump($input->headers->all());

// Will be normalized into "Accept"
dump($input->headers->get('accept')); 

// Return Accept header as array of values
dump($input->headers->get('accept', false));

dump($input->header('accept'));
```

### Cookies

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->cookies->all());

dump($input->cookie('name'));
```

### Server variables

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server->all());

dump($input->server('name'));
```

> **Note**
> `Spiral\Http\Request\ServerBag` will automatically normalize all requested server values. This makes it possible to 
> get value without using all uppercase letters for names:

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->server('SERVER_PORT'));

dump($input->server('server-port'));
```

### Post/Data parameters

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->data->all());

dump($input->data('name'));

// An alias
dump($input->post('name'));
```

### Post/Data with fallback to Query parameters

If you want to read the value from POST data and then from Query, simply use the method `input`.

```php
dump($input->input('name'));
```

### PSR7 request attributes

```php
dump($input->attributes->all());

dump($input->attribute('name'));
```

#### Uploaded Files

To get a list of the uploaded files or individual files, use the `files` bag and `file` method. Every uploaded file
instance is represented using `Psr\Http\Message\UploadedFileInterface`, which is a part of PSR7.

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($this->input->files->all());

dump($this->input->file('upload'));
```

> **Note**
> Per PSR, all files are organized to a logical hierarchy, which differs from default way php handle uploaded files. You can
> use dot notation to access nested file instances.

### Simplified methods

In addition to data methods and InputBags, `Spiral\Http\Request\InputManager` provides a set of methods to read various 
properties of active requests.

```php
/** @var \Spiral\Http\Request\InputManager $input */

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

To access `Spiral\Http\Request\InputBag` without the use of `__get`:

```php
/** @var \Spiral\Http\Request\InputManager $input */

dump($input->bag('data')->all());
```

### Adding new input bag

With `Spiral\Bootloader\Http\HttpBootloader` you can add your own input bag.

For example, we need to receive uploaded files as a Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile object.
Let's create an input bag class.

```php
namespace App\Http\Request;

use Spiral\Http\Request\InputBag;
use Symfony\Bridge\PsrHttpMessage\Factory\UploadedFile;

final class FilesBag extends InputBag
{
    public function __construct(array $data, string $prefix = '')
    {
        foreach ($data as $name => $file) {
            $data[$name] = new UploadedFile($file, fn(): string => $this->getTemporaryPath());
        }

        parent::__construct($data, $prefix);
    }

    protected function getTemporaryPath(): string
    {
        return \tempnam(\sys_get_temp_dir(), \uniqid('symfony', true));
    }
}
```

After that, add the created `FilesBag` by the method `addInputBag` in the `HttpBootloader`.

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use App\Http\Request\FilesBag;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Http\HttpBootloader;

class AppBootloader extends Bootloader
{
    public function init(HttpBootloader $http): void
    {
        $http->addInputBag('symfonyFiles', [
            'class'  => FilesBag::class,
            'source' => 'getUploadedFiles',
            'alias' => 'symfony-file'
        ]);
    }
}
```

## InputInterface

The `Spiral\Http\Request\InputManager` does not have `get` prefix for its methods. The reason for that is located in an 
external package `spiral/filters` which requires a data source provider via `Spiral\Filters\InputInterface`:

```php
namespace Spiral\Filters;

// ...

interface InputInterface
{
    public function withPrefix(string $prefix, bool $add = true): InputInterface;

    public function getValue(string $source, string $name = null);
}
```

You can invoke `Spiral\Http\Request\InputManager` methods via short notation of `Spiral\Filters\InputInterface`. Both 
approaches will produce the same set of data.

```php app/Interface/Controllers/HomeController.php
use Spiral\Filters\InputInterface;
use Spiral\Http\Request\InputManager;

public function index(InputInterface $inputSource, InputManager $inputManager): void
{
    dump($inputManager->query('name'));
    dump($inputSource->getValue('query', 'name'));

    dump($inputManager->path());
    dump($inputSource->getValue('path'));
}
```

This approach is used to map the incoming data into Request Filter.

> **Note**
> You must activate `Spiral\Bootloader\Security\FiltersBootloader` in order to access `Spiral\Filters\InputInterface`.

## Generate Response

You can return the instance `Psr\Http\Message\ResponseInterface` from your controller, and it will be sent directly to
the user.

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

use Nyholm\Psr7\Response;
use Psr\Http\Message\ResponseInterface;

class HomeController 
{
    public function index(): ResponseInterface
    {
        $response = new Response(200);
        $response->getBody()->write("hello world");

        return $response;
    }
}
```

The PSR-15 handler enabled by default provides you with an ability to generate the response automatically based on the returned
string or the content of output buffer:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

class HomeController
{
    public function index(): string
    {
        return "hello world";
    }
}
```

Identical to:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

class HomeController
{
    public function index(): void
    {
        echo "hello world";
    }
}
```

> **Note**
> We recommend using output buffer only during the development to display debug information. Stick to strict return
> types.

## JSON responses

The default PSR-15 also supports array and `JsonSerializable` responses which will convert into JSON:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

class HomeController
{
    public function index(): array
    {
        return [
            'status' => 200,
            'data' => ['some' => 'json']
        ];
    }
}
```

## Response Factory

The proper way to abstract from manual response creation is to use `Psr\Http\Message\ResponseFactoryInterface`:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

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

To generate more complex responses, use `ResponseFactoryInterface` wrapper `Spiral\Http\ResponseWrapper` which adds
a number of methods for simpler response generation:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

use Psr\Http\Message\ResponseInterface;
use Spiral\Http\ResponseWrapper;

class HomeController
{
    public function index(ResponseWrapper $response): ResponseInterface
    {
        return $response->attachment(
            __FILE__,
            'controller.php'
        )->withAddedHeader('Key', 'value');
    }
}
```

You can also access the wrapper via `PrototypeTrait` and property `response`:

```php app/Interface/Controllers/HomeController.php
namespace App\Interface\Controller;

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

To create an HTML response:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->html('hello world');
}
```

To create `application/json` response:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

To send an attachment:

```php app/Interface/Controllers/HomeController.php
public function index(): ResponseInterface
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> **Note**
> You can also use `Psr\Http\Message\StreamInterface` as the first argument and specify your mime-type as the third 
> option.
