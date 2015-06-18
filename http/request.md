# Http Request
Spiral Framework implements PSR-7 to handle incoming requests and generate output responses.

You can access input request at any moment using dynamic binding `request` in application container or requesting dependency injection with `ServerRequestInteface`.

## Get incoming request
Let's review few examplex of receiving input request instance inside controller.

> Attention, request can be requested or injected only inside http request scope, following code will fail in console commands or controllers executed outside HttpDispatcher flow.

Using dynamic binding.

```php
public function index()
{
	dump($this->request->getMethod());
	dump($this->request->getUri());
}
```

Additionally you can receive active request using method injection.

```php
public function action(ServerRequestInterface $request)
{
	dump($request);
}
```

To get more information about request structure visit PSR-7 meta document.

## Nested requests
In some cases (testing, request emulation) you may want to send altered requested to be executed by `HttpDispatcher`, due PSR-7 request is imutable you can execute such operation fairly easy.

```php
public function index()
{
	//Execute request with altered uri
	return $this->http->perform(
		$this->request->withUri(new Uri('...'))
		);
}
```

You can perform any request by `HttpDispatcher` it only has to implement PSR-7 `ServerRequestInterface`. All request injections will be resolved with new provided instance inside `perform` method scope and original request will stay untouched.

> Consider using `Core` functionality `callAction` for HMVC, calling external controllers using HttpDispatcher are pretty expensive.

```php
public functio index()
{
	return $this->core->callAction(SomeController::class, 'method');
}
```

## Accessing input data (query, post, cookies)
Client request data can be accessed using ServerRequestInstance and it's getters or via InputManager component which provides simplied data access methods.

You can read more about ServerRequest getters in PSR-7 meta document, but let's review InputManger functionality as it generally much easier to use to everyday development.

InputManager can be requested by it's class name using method or constructor dependency, or via dynamic binding `input`.

```php
public function index()
{
	dump($this->input->query->all());
}
```

InputManager contains multiple magic properties linked to different part of request plus set of methods to perform common checks.

### Accessing POST, GET and COOKIES content
Most of request content can be accesseds by set of input "bags" and simplified methods, every input bag will inplement following methods (using `data` bag as example):

```php
public function index()
{
	//Receive all data as array
	dump($this->input->data->all());
	
	//Fetch single value from data by it's name, default value
	//is `null` by default
	dump($this->input->data->get('name', 'default'));
	
	//Check if data has following key
	dump($this->input->data->has('name'));
	
	//Create array with only specified keys fetched from data, you can
	//use 2nd and 3rd arguments to fill such array with default values
	dump($this->input->data->fetch(['name', 'nameB']));
}
```

> In addition to that any bag can be used as array or be interated over using foreach.

#### Request Headers
We can utilize `headers` bag and "header" method of InputManager to get access to request headers.

```php
public function index()
{
	//All input headers
	dump($this->input->headers->all());
	
	//Simplified method to get one header, 2nd argument specifies 
	//default value
	dump($this->input->header('header'))
}
```

HeaderBag has few differences comapared to other bags:

* All requested header names will be normalized, so you can ask for header in many forms (Content-Type, content-type and etc.)
* `get` method will automatically implode all header values into one string using ",", you can disable this behaviour by setting 3rd arugment as false or null.

#### POST data
We can utilize `data` bag and "data" method of InputManager to get access to request POST data.

> Data bag may be referring not only to POST data but to any value set using setParsedBody of request. For example it can be parsed Json data sent by client.

```php
public function index()
{
	dump($this->input->data->all());
	dump($this->input->data('header'))
}
```

#### Query
```php
public function index()
{
	dump($this->input->query->all());
	dump($this->input->query('name'))
}
```

#### Request Cookies
```php
public function index()
{
	dump($this->input->cookies->all());
	dump($this->input->cookie('name'))
}
```

#### Server data
This bag will usually map to _SERVER array.

```php
public function index()
{
	dump($this->input->server->all());
	dump($this->input->server('name'))
}
```

ServerBag will automatically normalize all requested server values. This makes possible to get value without uppercasing names: 

```php
$this->input->server('SERVER_PORT');
$this->input->server('server-port');
```

#### Uploaded Files
To get list of uploaded files or individual file use `files` bag and `file` method. Every uploaded file instance represent using `UploadedFileInterface` which is part of PSR-7.

```php
public function index()
{
	dump($this->input->files->all());
	dump($this->input->files('upload'))
}
```

To get more information has to handle file uploads and additional methods of FileBag read "[Working with Uploaded Files] (files.md)".

> Per PSR all files will be organized to valid ieharhy, which differs from default way php handle uploaded files.

### Simplified methods
Let's review various InputManager methods by example:

```php
public function index()
{
	//Request Uri path, will always include leading /
	dump($this->input->getPath());
	
	//Active request Uri instance
	dump($this->input->getUri());
	
	//GET, POST, PUT...
	dump($this->input->getMethod());
	
	//Check if connection made over https
	dump($this->input->isSecure());
	
	//Check request headers to verify that request made over ajax
	dump($this->request->isAjax());
	
	//Receive client ip address (this method uses _SERVER 
	//value and may not be correct in some cases).
	dump($this->request->getRemoteAddress());
}
```

### Input Facade
If you would like to work with Input using static methods you can use `Input` facade which has all methods above mapped statically.

```php
echo Input::query('name', 'default');
dump(Input::getUri());
dump(Input::getBag('data')->all());
``` 
