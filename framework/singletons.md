# Framework - Singletons
A lot of internal application services reside  in a memory in the form of singleton objects. Such objects does not
implement static `getIntance` but rather configured to remain in container **between requests**.

Declaring your service or controller as singleton is the shortest path to get small performance improvement, however,
some rules must be applied in order to avoid memory and state leaks.

## Define the Singleton
Framework provides multiple ways to declare class object as singleton. At first you can create bootloader in which
you can bind class under it's own name:

```php
class ServiceBootloader extends Bootloader
{
    protected const SIGNLETONS = [
        Service::class => Service::class
    ];
}
```

Now, you will always receive the same instance from IoC container by doing the injection.

Alternative approach does not require Bootloader and can be defined on class itself, implement interface `Spiral\Core\Container\SingletonInterface`
to declare to the container that class must be constructed only once:

```php
use piral\Core\Container\SingletonInterface;

class Service implements SingletonInterface
{
    // ...
}
``` 

> You can implement singleton interface in services, controllers, middleware and etc. Do not implement it in Repositories
and mappers as this classes state are managed by ORM.

## Limitations
By keeping your services in memory between requests you can avoid doing some complex initializations and computations
over and over. However you must remember that your services must be designed in **stateless** fashion and do not contain
any user data.

You can not:
- store user information in your singleton
- store reference to PSR-7 request (use `InputManager` instead)
- store reference to session (use `SessionScope` instead)
- store RBAC actor (use `GuardScope` instead) 

## Pre-Heating
You are allowed to store data in services which can not change between user requests. For example, if you application
relies on heavy XML as configuration source:


```php
class Service implements SingletonService 
{
    private $configCache;


    public function getConfig(): array
    {
        if ($this->configCache !== null) {
            return $this->configCache;
        }
    
        $this->configCache = $this->readConfig(); // heavy operation
    
        return $this->configCache;
    }
}   
```

Using such approach you can perform complex computations only once and rely on RAM cache on later user requests.
