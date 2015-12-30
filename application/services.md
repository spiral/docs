# Service Classes and Models
Spiral framework trying to help you to separate database and business logic, since architecture of your application is only limited by your imagination there is no recommended approach of how to organize your models but only few help classes and implentations.

Framework provides simple parent for your classes `Spiral\Core\Service`, the only valuable thing this implementation can do for your - give you shorted access to [container and shared bindings](/framework/container.md).

```php
class Service extends Component
{
    use SharedTrait, SaturateTrait;

    /**
     * @var InteropContainer
     */
    protected $container = null;

    /**
     * Due usage of saturate trait you can skip call for parent __construct method, but in this 
     * case contrainer will be resolved via global/static scope which is not recommended. 
     *
     * @param InteropContainer $container Sugared. Used to drive shared/virtual bindings, if any.
     */
    public function __construct(InteropContainer $container = null)
    {
        $this->container = $this->saturate($container, InteropContainer::class);
    }
}
```

> Every Controller is Service.

## Example Model
The simpliest implementation of our service with shared bindings usage might look like:

```php
class UsersService extends Service
{
    public function getCount()
    {
        //This is better to be done using DI, see below
        return $this->db->table('users')->count();
    }
}
```

> Services DO NOT support method injection, you must provide arguments manually or use contructor injection.

Now you can use this sample service in your controller this way:

```php
protected function testAction(UsersService $users)
{
    dump($users->getCount());
}
```

## Contructor Injections
As any other class you can change your Service contructor to request additional dependecies, for example we can create model which manages our blog posts:

```php
class BlogService extends Service
{
    private $source = null;

    public function __construct(PostsSource $source)
    {
        $this->source = $source;
    }
    
    public function getPosts($date)
    {
        return $this->source->findByDate($date);
    }
}
```

> In a previous example we can simply drop `extends Service` as no shared bindings functionality were used.

Let's try to create more complex model:

```php
class BlogService extends Service
{
    private $source = null;

    public function __construct(PostsSource $source)
    {
        $this->source = $source;
    }
    
    public function countPosts()
    {
        if($this->cache->has('posts')) {
            return $this->cache->get('posts');
        }
        
        $count = $this->source->count();
        $this->cache->set('posts', $count, 3600);
    
        return $count;
    }
}
```

> In a given example we are going to cache posts count for 1 hour, following methodic can help us to keep our views and controller ligther.

This example has one issue - code will work perfectly fine, however `$this->cache` will be resolved using shared container (global for your application) which *might* create minor issues on testing stage, to solve it let's improve our contructor:

```php
class BlogService extends Service
{
    private $source = null;
    
    public function __construct(ContainerInterface $container, PostsSource $source)
    {
        parent::__constuct($container);
        $this->source = $source;
    }
    
    ...
```

Alternatively, at any moment, we can refactor our class to decouple from Service:

```php
class BlogService
{
    private $source = null;
    private $cache = null;

    public function __construct(PostsSource $source, StoreInterface $cache)
    {
        $this->source = $source;
        $this->cache = $cache;
    }
    
    ...
```

> You only need to pass container to contructor if your service using shared bindings and
virtual properties. You are not forced to use Services, use plain classes as much as you
can (concider Service class as RAD sugar).

## Testing Models
To cover services which use shared bindings with tests you only need to mock needed bindings
in a container passed into your service, let's try to create simple test for our model:

```php
class BlogServiceTest extends TestCase
{
    public function testCount()
    {
        $source = $this->getMock(PostSource::class);
        $source->expects($this->once())->method('count')->willReturn(100);

        $cache = $this->getMock(StoreInterface::class);
        $cache->expects($this->once())->method('has')->with('posts')->willReturn(false);
        $cache->expects($this->once())->method('set')->with('posts', 100, 3600);

        $container = new Container();
        $container->bind('cache', $cache);

        $service = new BlogService($source, $container);
        $this->assertEquals(100, $service->countPosts());
    }
}
```
