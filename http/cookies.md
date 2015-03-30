# Cookies
By default PSR 7 does not separate cookie headers from any other Response header, this may create some problems while trying to work with cookies in efficient way. To solve this problem spiral provides 2 different compatible ways to manage cookie headers inside your controlers and middleware.

## Set-Cookie String Header
One the option to create cookie header in response is to add it manually, to do that let's use default `Reponse->withAddedHeader()` 
constuction or directly specify set of headers for new Response.

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

As you may see this way does not provide too convinient way to create cookies, hovewer you can use direct header writing at any moment in cases where you want to control cookie content or other options manually.
> CookieManager middleware (see next) will not encrypt cookies specified as simple string headers. This way is not really recommended due potentical problem while decrypting cookies.

## Set-Cookie object header
Another way to create cookie still involves direct access to `Response` but provides simplified interface to generate cookie.
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

`Cookie` instance can generate header string by itself via calling `compile()` method or by converting object to string. This ability
makes Cookie class compatible with PSR 7.

> Cookies specified via class will be automatically encrypted via CookieManager.

You can provide values to `Cookie` instance constructor in a same order as in `setcookie()` method, however expires parameter replaced with lifetime which is relative expiration interval in seconds. 

## Cookie Manager and Middleware
Third and the last options to manage cookie involves specially designed interface which will automatically add cookie headears to response at moment of dispatching.

Middleware impelements spiral singleton pattern and can be requested with DI, shortcut binding (`cookies`) in controllers or via static facade. All examples provided for controller action:

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
CookiManager set method accepts parameters in a same order as `setcookie()` method, however expires parameter replaced with lifetime which is relative expiration interval in seconds.

> CookieManager will ALWAYS encrypt and descrypt cookies except whitelisted cookies (csrf token and session id). If you really need to keep your cookie in non encrypted form you have to expllicitly tell `CookieManager` to skip cookie encrypting/decrypting by it's name.
> 
> Make sure that `CookieManager` middleware in not disable in http config, this will make all provided examples invalid.