# Cookies
By default, PSR 7 does not separate cookie headers from any other `Response` headers, this may create some problems while trying to work with cookies in an efficient way. To solve this problem, Spiral provides 2 different compatible options to manage cookie headera froms inside your controllers and middleware.

## Set-Cookie String Header
The first option is to create the cookie header within the response manually. To do that let's use the default `Reponse->withAddedHeader()` 
constuction or directly specify a set of headers for the new Response.

```php
public function action()
{
	return new Reponse('', 200, array(
		'Set-Cookie'=>[
			'nameA=valueA',
			'nameB=valueB'
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

As you can see, this option does not provide a convenient way to create cookies, however you can directly write the header in cases where you want to control the cookie content manually.

> The CookieManager middleware (see next) will not encrypt cookies specified as simple string headers. This is not recommended due to potential problems while decrypting cookies.

## Set-Cookie object header
Another way to create cookies requires direct access to the `Response` object but provides a simplified interface to generate cookies.
In this case, the header line will not be a string but instead will be  a `Cookie` object where the values are stored as class properties.

In this case header line will be represent not as string but as `Cookie` object where values stored as class properties.

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
makes the Cookie class compatible with PSR 7.

> Cookies specified using object header will be automatically encrypted via the CookieManager.

You can pass values to the `Cookie` instance constructor in the same order as in the `setcookie()` method, however the expires parameter is replaced with the lifetime which is the relative expiration interval in seconds. 

## Cookie Manager and Middleware
The third and the last option to manage cookies involves a specially designed interface, which will automatically add cookie headers to the response at dispatch time.

Middleware implements the spiral singleton pattern and can be requested with DI, shortcut binding (`cookies`) in controllers or via static facade. All examples are assume the use of a controller action:

```php
use Spiral\Facades\Cookies;
use Spiral\Components\Http\Cookies\CookieManager;

...

public function action(CookieManager $cookies)
{
	//Every example provides same functionality
	$cookies->set('name', 'value', 3600);
	
	$this->cookies->set('name', 'value', 3600);
	
	Cookies::set('name', 'value', 3600);
}

```
The CookieManager `set` method accepts parameters in the same order as the `setcookie()` method, however the expires parameter is replaced with the lifetime which is the relative expiration interval in seconds.

> The CookieManager will ALWAYS encrypt and deccrypt cookies except for whitelisted cookies (csrf token and session id). If you really need to keep your cookie in non-encrypted form you have to explicitly tell the `CookieManager` to skip cookie encrypting/decrypting by its name.
> 
> Make sure that the `CookieManager` middleware is not disabled in the http config, this will make all provided examples invalid.
