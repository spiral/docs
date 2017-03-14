# Cookies
CookieManager is http middleware middlewares which defines IoC and Request attribute for CookieQueue class which is used as bucket for all cookies created in a such scope:

```
middleware invoke (request, response, next)
  queue = new queue
  
  request = request with decoded cookies
  container share queue
      response = next (request, response)
      response = response with cookies from queue with encrypting
  container release queue
   
return response 
```

## Example of Usage
By default `CookieManager` middleware already mounted in http config  (run on every request), this gives you ability to access cookie queue inside your controllers using either dependency or shortcut "cookies":

```php
public function indexAction(CookieQueue $cookies)
{
   $cookies->set('hello', 'wold');
   
   $this->cookies->set('abc', 'value');
}
```

Attention, CookieQueue is not the same as request cookies:

```php
dump($this->input->cookies);
```

## Configuring cookie manager
`CookieManager` can be configured using `HttpConfig` section:

```php
'cookies'      => [
    //Default cookie domain (null - no header value)
    'domain'   => null,
    
    //Cookie protection method
    'method'   => Http\Configs\HttpConfig::COOKIE_HMAC,
    
    //Cookies excluded from encryption
    'excluded' => [
        /*{{cookies.excluded}}*/
    ]
],
```

### Cookie protection methods
* HttpConfig::COOKIE_UNPROTECTED - all cookie values sent in raw mode
* HttpConfig::COOKIE_ENCRYPT - value passed though EncrypterInterface
* HttpConfig::COOKIE_HMAC - sings cookie values using `hash_hmac` (sha256) (EncrypterInterface key [~under consideration])

> When cookie manager meets value which can not be decoded it will replace it with null.

### List of excluded cookies
If your application creates set of cookies which should not be protected/encrypter (for example to exchange data with SPA) simply list name of such cookie in section 'excluded'.