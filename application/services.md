# Service Classes and Models
Spiral framework tries to help you to separate database and business logic, since architecture of your application is only limited by your imagination there is no recommended approach of how to organize your models but only few help classes and implementations.

To speed up model development, use class `Spiral\Core\Service` which can give you shorter access to [container and shared bindings](/framework/container.md). 

> Every Controller is a Service as well. Consider implementing [SingletonInterface](/framework/container.md) in your services in order to share them across application.

## Example Model
Due to container shortcuts simpliest implementation of service class might look like:

```php
class UsersService extends Service
{
    public function getCount(): int
    {
        return $this->db->table('users')->count();
    }
}
```

> Services DO NOT support method injection, you must provide arguments manually or use constructor dependencies.

Now you can use this sample service in your controller:

```php
protected function testAction(UsersService $users)
{
    dump($users->getCount());
}
```

> Consider container shortcuts as RAD approach and replace them with proper dependencies when ready.

## Constructor Injections
As any other class you can change your Service constructor to request additional dependencies, for example we can create model which manages our blog posts source:

```php
class BlogService implements SingletonInterface
{
    private $source;

    public function __construct(PostsSource $source)
    {
        $this->source = $source;
    }
    
    public function getPosts(DateTimeInterface $date): RecordSelector
    {
        return $this->source->findByDate($date);
    }
}
```

> As you can notice, you are not required to extend `Service` in this case.