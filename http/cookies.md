# Cookies
Spiral CookieManager is one of middlewares which defines IoC container scope in order to share cookie queue between application services. CookieManager flow can be represented by following code (simplified):

```
middleware invoke (request, response, next)
  queue = new queue
  
  request = request with decoded cookies
  container share queue
      response = next (request, response)
      response = response with cookies from queue with encoding
  container unshare queue
   
return response 
```

## Example of Usage
By default `CookieManager` middleware already mounted in http config as one of primare (run on every request) middlewares, this give you ability to access cookie queue inside your controllers using either dependency or short bining "cookies":

```php
public function indexAction(CookieQueue $cookies)
{
   $cookies->set('hello', 'wold');
   $this->cookies->set('abc', 'value');
}
```

Attention, `CookieManager` only provides cookie writing and protection functionality, use `InputManager` to read cookie values from incoming request:

```php
dump($this->input->cookies);
```

## Configuring cookie manager
CookieManager follow configuaration settings defined in http config, let's view them:

```php
'cookies'      => [
    //You force specific domain or use pattern like ".{host}" to share cookies with sub domains
    'domain'   => null,
    
    //Cookie protection method
    'method'   => Http\Configs\HttpConfig::COOKIE_HMAC,
    
    'excluded' => [
        'csrf-token',
        env('SESSION_COOKIE')
        /*{{cookies.excluded}}*/
    ]
],
```

### Default **cookie domain**
By default cookies will be set without specified domain, you can either hardcode your value (using env for example) or use pattern

### Protection **method**
CookieManager support following methods to protect your cookies from tampering:
* HttpConfig::COOKIE_UNPROTECTED - avoid using such method without specific reason.
* HttpConfig::COOKIE_ENCRYPT - the strongest and slowest protection methods which utilize EncrypterInterface, in this method user will be able to see cookie content. 
* HttpConfig::COOKIE_HMAC - sings cookie values using `hash_hmac` (sha256), user will be able to view cookie content, but unable to edit it's value.

> When cookie manager meets value which can not be decoded it will replace it with null.

### List of excluded cookies
If your application creates set of cookies which should not be protected simply list name of them in this section.
