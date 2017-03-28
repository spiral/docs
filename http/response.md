# Response and Responders
As in case with active request scope, you are able to get access to current instance of response in your controllers in PSR7 fashion.

```php
public function indexAction(ResponseInterface $response)
{
    $response->getBody()->write('Hello World');

    return $response->withHeader('Test', 'My Header');
}
```

> There is no shortcut for active request as it stated as immutable and can not overwritten, only returned.

## Writing into response
To simplify development spiral MiddlewarePipeline might not only accept `ResponseInterface` from it's endpoint (your controllers and action) but also string and arrays, meaning that we can return following structures:

```php
public function indexAction()
{
  return 'Hello world';
}
```

Pipeline can also understand array and `JsonSerializable` responses which will be written as json structure:

```php
public function indexAction()
{
  return [
      'status' => 200,
      'hello'  => 'world'
  ];
}
```

> In this case 200 will be used as response code.

To better understand how this code work let's try to look to the following simplification (this is not actual code):

```php
class MiddlewarePipeline
{
    public function execute($request, $response)
    {
        //Where your endpoint is your controller
        $result = $this->executeEndpoint($request, $response);

        if ($result instanceof ResponseInterface) {
            return $request;
        }

        if (is_array($result) || $result instanceof \JsonSerializable) {
            return $this->writeJson($response, $result);
        }

        //As string
        $response->getBody()->write($result);

        return $response;
    }
}
```

## Shortcuts
You can simplify response manipulation by using `Spiral\Http\Response\ResponseWrapper`:

```php
public function indexAction(ResponseInterface $response)
{
    $responder = new ResponseWrapper($response); 
    
    return $responder->redirect('http://google.com');
}
```

Response wrapper can be requested as injection:

```php
public function indexAction(ResponseWrapper $response)
{
    return $response->redirect('http://google.com');
}
```

Or via shortcut `response`:

```php
public function indexAction()
{
    return $this->response->redirect('http://google.com');
}
```

## Responder Methods
Let's view other methods than redirect available in Responder.

### Write HTML

```php
public function indexAction()
{
    return $this->response->html('hello world');
}
```

### Write JSON
```php
public function indexAction()
{
    return $this->response->json(
        ['something' => 123],
        200
    );
}
```

### Send file (attachment)
```php
public function indexAction()
{
    return $this->response->attachment(__FILE__, 'name.php');
}
```

> You can also use Stream or Storage object as first argument and specify your own mimetype as third option.

## Modifying response after responder
Responder methods are only returns modified instance of ResponseInterface, this means that you can apply additional modifications to it:

```php
public function indexAction()
{
    return $this->response->attachment(
        __FILE__, 
        'name.php'
    )->withHeader('Header', 'Value');
}
```