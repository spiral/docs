# Request and Input
As mentioned in [Http Flow](/http/flow.md) you can access incoming request in your controllers and services using "request" binding, or via `ServerRequestInterface` dependency. Such instance is only available inside `HttpDispatcher` perform method.

As result we are able to access user request in our controllers using 3 different approaches:

```php
protected function indexAction(ServerRequestInterface $request)
{
    dump($request->getQueryParams());
    dump($this->request->getQueryParams());
    dump($this->container->get(ServerRequestInterface::class)->getCookieParams());
}
```

However, PSR7 implementation is not very user friendly. Spiral framework provides specific service `InputManager` (`InputInterface`) used to simplify reading request properties. 
InputManager can be retrieved using it's class name or short binding "input". Let's rewrite previous example using input manager:

```php
protected function indexAction(InputManager $input)
{
    dump($input->cookies->all());
    dump($input->query->all());
    
    //POST or JSON data
    dump($this->input->data);
}
```

## Working with InputManager
You can use input manager to access full array of input data or any specific field by it's name (dot notation is allowed for nested structures). 
Every input structure are represented using `InputBag` class with set of common methods, let's review query accessing as example:

```php
/**
 * @var InputManager $input
 */
 
//Get instance of QueryBag associated with query data
dump($input->query);
 
//Get all query params as array
dump($input->query->all());
 
//Count of query params
dump($input->query->count());
 
//Check if parameter "name" presented in query
dump($input->query->has('name'));
 
//Get value for parameter "name"
dump($input->query->get('name'));
 
//Both get and has methods support dot notation for nested structures
dump($input->query->has('name.subName'));
dump($input->query->get('name.subName'));
 
//Fetch only given query params (no dot notation allowed), only existed records will be returned
dump($input->query->fetch(["name", "nameB"]);

//Fetch only given query params (no dot notation allowed), non existed records will be filled with `null`
dump($input->query->fetch(['name', 'nameB'], true, null);

//In addition query get method has short alias in input manager
dump($input->query('name'));
```

> Please note that `InputManager` always refer to active user request, you are not able to use this class outside of http scope.

### Input headers
We can use 'headers' input bad and `header` method in `InputManager` to access input headers. HeadersBag has few additions we have to mention:
* HeaderBad will automatically normalize requested header name
* "get" method will implode header values using ',' by default

```php
dump($input->headers->all());

//Will be normalized into "Accept"
dump($input->headers->get('accept')); 

//Return Accept header as array of values
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

//An alias
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
dump($this->input->files('upload'));
	
//If you want to get finename pointing to file content you can use `localFilename` method, attention, this is not local filename
//but unique stream uri which can work only inside spiral
dump($input->files->localFilename('upload'));
```

> Per PSR all files will be organized to valid hierarchy, which differs from default way php handle uploaded files, you can use dot notation to access nested file instances.

### Simplified methods
In addition to data methods and InputBags `InputManager` provides set of methods to read various properties of active request.

```php
//Request Uri path, will always include leading /
dump($input->getPath());

//Active request Uri instance
dump($input->getUri());

//GET, POST, PUT...
dump($input->getMethod());

//Check if connection made over https
dump($input->isSecure());

//Check request headers to verify that request made over ajax
dump($input->isAjax());

//Check is request expects application/json as response (Accept: application/json)
dump($input->isJsonExpected());

//Receive client ip address (this method uses _SERVER value and may not be correct in some cases).
dump($input->remoteAddress());
```
