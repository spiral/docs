# Cookies
By default PSR 7 does not separate cookie headers from any other Response headers, this may create some problems while trying to work with cookies in an efficient way. To solve this problem, spiral provides 2 different compatible ways to manage cookie headers inside your controlers and middleware.

## Set-Cookie String Header
The first option is to create the cookie header within the response manually. To do that let's use the default `Reponse->withAddedHeader()` 
constuction or directly specify a set of headers for the new Response.

```php
public function action()
{
	return new Reponse('', 200, array(
		'Set-Cookie'=>[
			'name=value'
		]
	));
}
```

```php
public function filter(Response $response)
{
	return $reponse->withAddedHeader('Set-Cookie', 'name=value');
}
```

As you may see this way does not provide a convenient way to create cookies, however you can create the header manually at any time when you want to control cookie content or other options manually.
> The CookieManager middleware (see next) will not encrypt cookies specified as simple string headers. This is not recommended due to potentical problem while decrypting cookies.

## Set-Cookie object header
Another way to create cookies requires direct access to the `Response` object but provides a simplified interface to generate cookies.
In this case, the header line will be represent not as a string but as a `Cookie` object where the values are stored as class properties.


```php
use Spiral\Components\Http\Cookies\Cookie;

...

public function action()
{
	return new Reponse('', 200, array(
		'Set-Cookie'=>[
			new Cookie('name', 'value', 3600)
		]
	));
}
```

```php
use Spiral\Components\Http\Cookies\Cookie;

...

public function filter(Response $response)
{
	return $reponse->withAddedHeader(
		'Set-Cookie', 
		new Cookie('name', 'value', 3600)
	);
}
```

The `Cookie` instance can generate a header string by itself via calling the `compile()` method or by converting the object to string. This ability
makes Cookie class compatible with PSR 7.

> Cookies created via class will be automatically encrypted via the CookieManager.

You can pass values to the `Cookie` instance constructor in the same order as in the `setcookie()` method, however the expires parameter is replaced with the lifetime which is the relative expiration interval in seconds. 

## Cookie Manager and Middleware
The third and the last option to manage cookies involves a specially designed interface which will automatically add cookie headers to the response at dispatch time.

Middleware implements the spiral singleton pattern and can be requested with DI, shortcut binding (`cookies`) in controllers or via static facade. All examples are provided for a controller action:

```php
use Spiral\Facades\Cookies;
use Spiral\Components\Http\Cookies\CookieManager;

...

public function action(CookieManager $cookies)
{
	//All examples are equal
	$cookies->set('name', 'value', 3600);
	
	$this->cookies->set('name', 'value', 3600);
	
	Cookies::set('name', 'value', 3600);
}

```
The CookieManager set method accepts parameters in the same order as the `setcookie()` method, however the expires parameter is replaced with the lifetime which is the relative expiration interval in seconds.

> The CookieManager will ALWAYS encrypt and deccrypt cookies except for whitelisted cookies (csrf token and session id). If you really need to keep your cookie in non-encrypted form you have to explicitly tell the `CookieManager` to skip cookie encrypting/decrypting by its name.
> 
> Make sure that `CookieManager` middleware in not disabled in the http config, this will make all provided examples invalid.
