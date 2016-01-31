# Response and Responders
As in case with active request scope you are able to get access to initial instance of response in your controllers in PSR7 fashion.

```php
public function indexAction(ResponseInterface $response)
{
    $response->getBody()->write('Hello World');

    return $response->withHeader('Test', 'My Header');
}
```

MiddlewarePipeline will automatically used this response to be passed back thought the chain of middlewares.

> There is no short binding for initial request as it stated as immutable and can not overwriten, only returned.

## Writing into response
To simplify development spiral MiddlewarePipeline might not only accept `ResponseInterface` from it's endpoint (your controllers and action) but also string and arrays, meaning that we can return followin structures:

```php
public function indexAction()
{
  return 'Hello world';
}
```

Pipeline can also understand array and JsonSerializable response which will be written as json structure:

```php
public function indexAction()
{
  return [
      'status' => 200,
      'hello'  => 'world'
  ];
}
```

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

> Attention, at this moment you have use proposed PSR7 flow (withHeader) to modify response headers in your controllers, HeaderManager is coming. 

## Shortcuts
In many cases you might want to skip talking to response in PSR7 fashion for a simple responses like redirects, html or sending file, this can be archived using instance of `Spiral\Http\Responses\Responder`:

```php
public function indexAction(ResponseInterface $response)
{
    $responder = new Responder($response, $this->files); //Files are needed for attachments

    return $responder->redirect('http://google.com');
}
```

Due `ResponseInterface` are clearly defined in container scope we can simply our code this way (responder will automatically get valid response dependency)

```php
public function indexAction(Responder $responder)
{
    return $responder->redirect('http://google.com');
}
```

We can even use short binding `responses` or `responder` for such purposes:

```php
public function indexAction()
{
    return $this->responses->redirect('http://google.com');
}
```

## Responder Methods
Let's view other methods than redirect available in Responder.

### Write HTML

```php
public function indexAction()
{
    return $this->responses->html('hello world');
}
```

### Write JSON
```php
public function indexAction()
{
    return $this->responses->json(
        ['something' => 123],
        200
    );
}
```

### Send file (attachment)
```php
public function indexAction()
{
    return $this->responses->attachment(__FILE__, 'name.php');
}
```

> You can also use Stream or Storage object as first argument and specify your own mimetype as third option.

## Modifying response after responder
Responder methods are only returns modified instance of ResponseInterface, this means that you can apply additional modifications to it:

```php
public function indexAction()
{
    return $this->responses->attachment(__FILE__, 'name.php')->withHeader('Header', 'Value');
}
```
