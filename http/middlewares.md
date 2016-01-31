# Http Middlewares
Http Middlewares is very powerful tool allowing you manipulate application flow using PSR7 request and response. Middlewares executed using so called pipeline, 
which will push request and response thought chain on defined middlewares. Make sure you read [Http Flow] (flow.md).

## Scaffoling
You can generate empty middleware using `create:middleware name` command, scaffolded middlewares automatically extend Service class so you can use any of service features (like short bindings, singleton constants and init method) in your class. In addition to that you can define service dependencies using `-d` option.

Let's try to generate example middleware `create:middleware headers`:

```php
class HeadersMiddleware extends Service implements MiddlewareInterface
{
    /**
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     * @param \Closure               $next Next middleware/target. Always returns ResponseInterface.
     * @return ResponseInterface
     */
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response,
        \Closure $next
    ) {
        return $next($request, $response);
    }
}
```

First of all, to make this middleware work we have to assign it to primary http pipeline (using http configuration) which is applied to every request or to specific route.
We are going to add this middleware to primary middleware chain in http config:

```php
 'middlewares'  => [
        Middlewares\CsrfFilter::class,
        Http\Cookies\CookieManager::class,
        Middlewares\JsonParser::class,
        \Spiral\Session\Http\SessionStarter::class,
    ],
```

Now we can either modify incoming request and response before sending it to next pipeline element or endpoing or modify generated response after it been processed. Let's do both of this operations:

```php
class HeadersMiddleware extends Service implements MiddlewareInterface
{
     /**
     * @param ServerRequestInterface $request
     * @param ResponseInterface      $response
     * @param \Closure               $next Next middleware/target. Always returns ResponseInterface.
     * @return ResponseInterface
     */
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response,
        \Closure $next
    ) {
        return $next($request->withAttribute('name', 'value'), $response)->withHeader('Header', 'value');
    }
}
```

This will make every request passed thought middleware get atrribute "name" and every response gain header "Header". In real world application middlewares can perform
much more complex operations, for example middleware can halt execution if some condition not met:

```php
/**
 * @param ServerRequestInterface $request
 * @param ResponseInterface      $response
 * @param \Closure               $next Next middleware/target. Always returns ResponseInterface.
 * @return ResponseInterface
 */
public function __invoke(
    ServerRequestInterface $request,
    ResponseInterface $response, 
    \Closure $next
) {
    if (empty($request->getCookieParams()['access-cookie'])) {
        return new HtmlResponse("You are not allowed to view this page.", 412);
    }

    return $next($request, $response);
}
```

To understand other abilities of middlewares and PSR7 classes let's look into code of default `JsonParser` middleware:

```php
class JsonParser implements MiddlewareInterface
{
    /**
     * {@inheritdoc}
     */
    public function __invoke(
        ServerRequestInterface $request,
        ResponseInterface $response,
        \Closure $next
    ) {
        if ($request->getHeaderLine('Content-Type') == 'application/json') {
            $request = $request->withParsedBody(json_decode(
                $request->getBody()->__toString(),
                true
            ));
        }

        return $next($request);
    }
}
```

Such middleware replaces request parsed body (data) with JSON structure, but only in case if incoming request has valid Content-Type.

## Default spiral Middlewares
There is set of pre-created middlewares you can use in your application:

| Middleware                                | Description 
| ---                                       | ---         
| Spiral\Http\Cookies\CookieManager         | Encrypts and decrypts incoming/outcoming cookies using encrypter or HMAC.                                         
| Spiral\Http\Middlewares\CsrfFilter        | Halts execution if request made via non GET, HEAD or OPTIONS method and no verification code provided in request. 
| Spiral\Http\Middlewares\JsonParser        | Converts JSON payload from request body into parsed request body if valid "Content-Type" header set (to be replaced by `Psr7Middlewares\Middleware\Payload`).              
| Spiral\Session\Http\SessionStarter        | Initiates session using ID stored in cookie "session" or creates such cookies in response if needed.              

In addition to that **Profiler module** also stated as middleware and automatically mounted by it's bootloader at top of middlewares chain when debug mode is enabled (see skeleton application).

## Mounting middlewares in Bootloader
In some cases you might want to mount middleware in your bootload class rather than editing config, to do that we have to make our class bootable and request HttpDispatcher dependency:

```php
class MyBootloader extends Bootloader implements SingletonInterface
{
    const BOOT = true;

    public function boot(HttpDispatcher $http)
    {
        //To the top of chain
        $http->riseMiddleware(new MyMiddleware());
    
        //To the end of chain
        $http->pushMiddleware(new MyMiddleware());
    }
}
```

> Attention, you can simply provide class name or container binding as middleware, it will be automatically resolved via IoC container (see example below).

## Other middlewares
You can find a lot of pre build middlewares outside of spiral, for example [https://github.com/oscarotero/psr7-middlewares](https://github.com/oscarotero/psr7-middlewares).

```php
public function boot(HttpDispatcher $http)
{
    $http->riseMiddleware(Middleware::responseTime());
    $http->pushMiddleware(
        Middleware::BasicAuthentication(['username' => 'password'])->realm('You shall not pass!')
    );
}
```

If you want to use short aliases for such middlewares concider creating container binding:

```php
class MyBootloader extends Bootloader implements SingletonInterface
{
    /**
     * @return array
     */
    protected $bindings = [
        //You can also use middleware class name as alias
        'middlewares.auth' => [self::class, 'authMiddleware'],
    ];

    public function authMiddleware(AppConfig $config)
    {
        return Middleware::BasicAuthentication(
            $config->getUsernames()
        )->realm('You shall not pass!');
    }
}
```

Now you can use such middleware in your http config or other places via short alias:

```php
'middlewares'  => [
    'middlewares.auth',

    Middlewares\CsrfFilter::class,
    Middlewares\ExceptionWrapper::class,

    \Middlewares\LocaleDetector::class,
    Session\Http\SessionStarter::class,
    Http\Cookies\CookieManager::class,

    /*{{middlewares}}*/
],
```

> Concider contributing into [https://github.com/oscarotero/psr7-middlewares](https://github.com/oscarotero/psr7-middlewares) directly if you wish to create more middlewares which are not spiral specific.
