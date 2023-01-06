# Long Start

The Spiral Framework contains a lot of components built to operate seamlessly with each other. In this article, we will
show you how to create a demo blog application with REST API, ORM, Migrations, request validation, custom annotations
(optional) and domain interceptors.

> **Note**
> The components and approaches will be covered at basic levels only. Read the corresponding sections to gain more
> information. You can find the demo repository [here](https://github.com/spiral/demo).

## Installation

Use composer to install the default `spiral/app` bundle with most of the components out of the box:

```bash
composer create-project spiral/app spiral-demo

cd spiral-demo
```

If everything is installed correctly, you can open your application immediately by starting the server:

```bash
./rr serve
```

You've just started an [application server](/framework/application-server.md). The same server can be used on
production,
making your development environment similar to the final setup. Out of the box, the server includes instruments to
write portable applications with HTTP/2, [GRPC](../grpc/configuration.md), [Queue](../queue/configuration.md),
[WebSockets](../broadcast/configuration.md), etc. and does not require external brokers to operate.

By default, the application is available on `http://127.0.0.1:8080`. The build includes multiple pre-defined pages you
can play with.

> **Note**
> Check the exception page `http://localhost:8080/exception.html`, at the right part of this page you can see all the
> interceptors and middleware included in the default build. We will turn some of them off to make the application
> runtime smaller.

## Configure

Spiral applications are configured using config files located in `app/config`, you can use the hardcoded values for the
configuration, or get the values using available functions `env` and [`directory`](../start/structure.md).
The `spiral/app` bundle uses [DotEnv](../extension/dotenv.md) extension which will load ENV variables from the`.env` 
file.

> **Note**
> Tweak the application server and its plugins using `.rr.yaml` file.

The application dependencies defined in `composer.json` and activated in `app/src/Application/Kernel.php` as
Bootloaders. The default build includes quite a lot of pre-configured components.

### Developer Mode

To simplify the tweaking of the application, restart the application server in developer mode. In this mode, the server
uses only one worker and reloads it after every request.

```bash
./rr serve -o "http.pool.max_jobs=1" -o "http.pool.num_workers=1" -o "http.pool.debug=true" -o "http.address=127.0.0.1:8181"
```

You can also create and use an alternative configuration file via `-c` flag of the `rr` application.

> **Note**
> Read more about Workers and Lifecycle [here](/start/workers.md).

### Lighter Up

We won't need translation, session, GRPC, cookies, CSRF, routing configuration via RoutesBootloader in our demo
application. Remove these components and their bootloaders.

Delete the following bootloaders from `app/src/Application/Kernel.php`:

```php
Spiral\Bootloader\I18nBootloader::class,

// from RoadRunner
Spiral\RoadRunnerBridge\Bootloader\GRPCBootloader::class,

// from http
Spiral\Bootloader\Http\CookiesBootloader::class,
Spiral\Bootloader\Http\SessionBootloader::class,
Spiral\Bootloader\Http\CsrfBootloader::class,
Spiral\Bootloader\Http\PaginationBootloader::class,

// from views 
Spiral\Bootloader\Views\TranslatedCacheBootloader::class,
```

Remove `App\Middleware\LocaleSelector` from method `globalMiddleware` in the `App\Bootloader\RoutesBootloader`:

```diff
class RoutesBootloader extends BaseRoutesBootloader
    // ...
    
    protected function globalMiddleware(): array
    {
        return [
-           LocaleSelector::class,
            ErrorHandlerMiddleware::class,
            JsonPayloadMiddleware::class,
            HttpCollector::class,
        ];
    }
    
    // ...
 }
```

By default, the routing rules are located in `app/src/Application/Bootloader/RoutesBootloader.php`. You have many
options on how to configure the routing. Point route to actions, controllers, controller groups, set the default 
pattern parameters, verbs, middleware, etc.

Remove method `defineRoutes`. We will add routes using attributes:

```diff
class RoutesBootloader extends BaseRoutesBootloader
    // ...
    
-    protected function defineRoutes(RoutingConfigurator $routes): void
-    {
-        $routes->import($this->dirs->get('app') . '/routes/web.php')->group('web');
-
-        $routes->default('/[<controller>[/<action>]]')
-            ->namespaced('App\\Controller')
-            ->defaults([
-                'controller' => 'home',
-                'action' => 'index',
-            ])
-            ->middleware([
-                // SomeMiddleware::class,
-            ]);
-    }
    
    // ...
 }
```

> **Note**
> Read more about Routing [here](/http/routing.md).

Remove all middlewares in method `middlewareGroups`:

```diff
class RoutesBootloader extends BaseRoutesBootloader
    // ...
    
    protected function middlewareGroups(): array
    {
        return [
-            'web' => [
-                CookiesMiddleware::class,
-                SessionMiddleware::class,
-                CsrfMiddleware::class,
-                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'cookie'])
-            ],
-            'api' => [
-                // new Autowire(AuthTransportMiddleware::class, ['transportName' => 'header'])
-            ],
        ];
    }
    
    // ...
 }
```

Remove unused imports and `__constructor`:

```diff
- use Spiral\Boot\DirectoriesInterface;
- use Spiral\Cookies\Middleware\CookiesMiddleware;
- use Spiral\Csrf\Middleware\CsrfMiddleware;
- use Spiral\Router\Loader\Configurator\RoutingConfigurator;
- use Spiral\Session\Middleware\SessionMiddleware;

class RoutesBootloader extends BaseRoutesBootloader
-    public function __construct(
-        private readonly DirectoriesInterface $dirs
-    ) {
-    }
    
    // ...
 }
```

Delete the following files and directories as no longer required:

- `app/locale`
- `app/routes`
- `app/src/Middleware`

> **Note**
> Note that the application won't work at the moment as we removed the dependency required to
> render `app/views/home.dark.php`.

### Database Connection

Our application needs a database to operate. By default, the database configuration is located
in `app/config/database.php` file. The demo application comes with a pre-configured SQLite database located in
`runtime/runtime.db`.

```php
// database.php 
use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'default'],
    ],
    'drivers' => [
        'default' => new Config\SQLiteDriverConfig(
            connection: new Config\SQLite\MemoryConnectionConfig(),
            queryCache: true
        ),
        // ...
    ],
];
```

We can store a database name, username, password and port in `.env` file, add the following lines into it:

```dotenv
DB_HOST=localhost
DB_NAME=name
DB_USER=username
DB_PASSWORD=password
DB_PORT=3306
```

> **Note**
> Change the values to match your database parameters.

To change the default database to MySQL, change the `drivers` section of the configuration, use `env` function to access
the ENV variables.

```php
use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'default'],
    ],
    'drivers'   => [
        'default' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: env('DB_NAME'),
                host: env('DB_HOST'),
                port: (int) env('DB_PORT', 3306),
                user: env('DB_USER'),
                password: env('DB_PASSWORD')
            ),
            queryCache: true
        ),
    ]
];
```

To check that the database connection was successful, run:

```bash
php app.php db:list
```

> **Note**
> Read more about Databases [here](/database/configuration.md).

### Connect Database Seeder

We will need some sample data for the application.
Let's install [Database Seeder](https://github.com/spiral-packages/database-seeder).

```bash
composer require spiral-packages/database-seeder --dev
``` 

Add the bootloader `Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader` to `LOAD` section
in `app/src/Application/Kernel.php` to activate the package:

```diff
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel

         RoadRunnerBridge\CommandBootloader::class,
+        \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
     ];
 }
```

> **Note**
> Read more about Bootloaders [here](/framework/bootloaders.md).

### Creating Routes

In order to simplify the route definition, we can use `spiral/annotated-routes` extension. Read more about the
extension [here](/http/annotated-routes.md). Add the bootloader to `LOAD` in `app/src/Application/Kernel.php` to 
activate the component:

```diff
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel

         SapiBootloader::class,
+        AnnotatedRoutesBootloader::class,
     ];
 }
```

We can use this attribute in our controller as follows:

```php
namespace App\Controller;

use Spiral\Router\Annotation\Route;

class HomeController
{
    #[Route(route: '/', name: 'index', methods: 'GET')]
    public function index(): string
    {
        return 'hello world';
    }
    
    #[Route(route: '/open/<id>', name: 'open', methods: 'GET')] 
    public function open(string $id)
    {
        dump($id);
    }
}
```

Run CLI command to check the list of available routes:

```bash
php app.php route:list
```

> **Note**
> Use additional route parameters to configure middleware, route group, etc.

In the following examples, we will stick to the routes with attributes for simplicity.

To flush route cache (when `DEBUG` disabled):

```bash
php app.php route:reset
```

### Domain Core

The domain core represents the fundamental domain logic and business rules that drive the functionality of an
application. It constitutes the central aspect of the codebase and often encompasses the most critical and intricate
elements of the software.

It is possible to modify the default behavior of the application by utilizing a route parameter and a `Guard` attribute
to enable resolution of the Cycle Entity. To further enhance the application, the `ValidationHandlerMiddleware` can be 
incorporated to properly validate incoming HTTP requests.

Here is the example:

```php
namespace App\Bootloader;

use App\Interceptor\ValidationInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\FilterValidationHandlerMiddleware;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        GuardInterceptor::class,
    ];
    
    public function boot(HttpBootloader $http)
    {
        $http->addMiddleware(ValidationHandlerMiddleware::class);
    }
}
```

Enable the domain core in your application. We will demonstrate the use of the interceptor below.
Add the bootloader to `APP` in `app/src/Application/Kernel.php`:

```diff
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel
+        Bootloader\AppBootloader::class,
     ];
 }
```

> **Note**  
> Read more about Domain Cores [here](/cookbook/domain-core.md).

## Scaffold Database

The framework can configure the database schema using a set of migration files. To configure migrations in your
application, run:

```bash
php app.php migrate:init
```

You can now observe the migration table structure by using:

```bash
php app.php db:list
php app.php db:table migrations
```

You can write the migration manually, or let Cycle ORM generate it for you.

> **Note**  
> Read more about migrations [here](https://cycle-orm.dev/docs/database-migrations).
> Use [Scaffolder](/basics/scaffolding.md) component to create migrations manually.

### Define ORM Entities

The demo application comes with [Cycle ORM](https://cycle-orm.dev). By default, you can use attributes to configure
your entities.

Enable the Cycle Bridge `Spiral\Cycle\Bootloader\ScaffolderBootloader` bootloader to use commands that create
entities and repositories. Add the bootloader to `APP` in `app/src/Application/Kernel.php`:

```diff
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel
        Scaffolder\ScaffolderBootloader::class,
+       CycleBridge\ScaffolderBootloader::class,
     ];
 }
```

Let's create `Post`, `User` and `Comment` entities and their repositories using the Scaffolder extension:

```bash
php app.php create:entity post -f id:primary -f title:string -f content:text -e
php app.php create:entity user -f id:primary -f name:string -e
php app.php create:entity comment -f id:primary -f message:string
``` 

> **Note**
> Observe the classes generated in `app/src/Database` and `app/src/Repository`.

Post:

```php
namespace App\Database;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $title;

    #[Column(type: 'text')]
    public string $content;
}
 ```

We can move the definition of the `$title` and `$content` properties to the `__construct` method:

```php
namespace App\Database;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $title,

        #[Column(type: 'text')]
        public string $content
    ) {
    }
}
```

User:

```php
namespace App\Database;

use App\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: UserRepository::class)]
class User
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $name;
}
```

We can move the definition of the `$name` property to the `__construct` method:

```php
namespace App\Database;

use App\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: UserRepository::class)]
class User
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $name
    ) {
    }
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $message;
}
```

We can move the definition of the `$message` property to the `__construct` method:

```php
namespace App\Database;

use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $message
    ) {
    }
}
```

Let's add the `Spiral\Prototype\Annotation\Prototyped` attribute to the repositories so that we can use them
as users and posts properties during development:

```php
// app/src/Repository/PostRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'posts')]
class PostRepository extends Repository
{
}

// app/src/Repository/UserRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'users')]
class UserRepository extends Repository
{
}
```

You can change the default directory mapping, headers, and others using [Scaffolder config](/basics/scaffolding.md).

> **Note**
> Read more about Cycle [here](/cycle/configuration.md). Configure auto-timestamps
> using [custom mapper](https://cycle-orm.dev/docs/advanced-timestamp).

Run the configure command to collect all available prototype classes:

```bash
php app.php configure
```

### Generate Migration

To generate the database schema, run:

```bash
php app.php cycle:migrate -v
```

The generated migration is located in `app/migrations/`. Execute it using:

```bash
php app.php migrate -vv
```

You can now observe the generated tables using `db:list` command.

### Create Relations

Use [attributes](https://cycle-orm.dev/docs/annotated-relations) to define the relations between entities. Configure
Post and Comment relations with type BelongsTo and add relation HasMany between User and Post.

Post:

```php
namespace App\Database;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation;
use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;

#[Entity(repository: PostRepository::class)]
class Post
{
    #[Column(type: 'primary')]
    public int $id;

    /**
     * @var Collection|Comment[]
     * @psalm-var Collection<int, Comment>
     */
    #[Relation\HasMany(target: Comment::class)]
    public Collection $comments;

    public function __construct(
        #[Column(type: 'string')]
        public string $title,

        #[Column(type: 'text')]
        public string $content,

        #[Relation\BelongsTo(target: User::class, nullable: false)]
        public User $author
    ) {
        $this->comments = new ArrayCollection();
    }
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation;

#[Entity]
class Comment
{
    #[Column(type: 'primary')]
    public int $id;

    public function __construct(
        #[Column(type: 'string')]
        public string $message,

        #[Relation\BelongsTo(target: User::class, nullable: false)]
        public User $author,

        #[Relation\BelongsTo(target: Post::class, nullable: false)]
        public Post $post
    ) {
    }
}
```

Once again generate and run the migration:

```bash
php app.php cycle:migrate -v
php app.php migrate -vv
```

> **Note**
> You can generate and run the migration in one command using `php app.php cycle:migrate -r`.

You can check the presence of Foreign Keys:

```bash
php app.php db:table comments
```

> **Note**
> Do not forget to run `php app.php cycle:migrate` when you change any of your entities.

## Factories and seeders

To generate test data, we need factories that will describe the rules for generating an entity.
And seeders that will fill the database.

We will store them separately from the application code, in the `app/database` folder.
Let's add a separate `Database` namespace to Composer autoload:

```diff
--- a/composer.json
+++ b/composer.json
"autoload": {
    "psr-4": {
        "App\\": "app/src",
+       "Database\\": "app/database"
    },
    //
},
```

```bash
composer dump-autoload
```

### CommentFactory

Let's create `CommentFactory` class, extend it from `Spiral\DatabaseSeeder\Factory\AbstractFactory` and implement
required methods:

```php
// app/database/Factory/CommentFactory.php
namespace Database\Factory;

use App\Database\Comment;
use App\Database\Post;
use App\Database\User;
use Faker\Generator;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class CommentFactory extends AbstractFactory
{
    /**
     * Returns a fully qualified database entity class name
     */
    public function entity(): string
    {
        return Comment::class;
    }

    /**
     * Returns an entity
     */
    public function makeEntity(array $definition): Comment
    {
        return new Comment($definition['message'], $definition['author'], $definition['post']);
    }

    /**
     * Generate Comment with given author
     */
    public function withAuthor(User $author): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'author' => $author,
        ]);
    }

    /**
     * Generate Comment with given post
     */
    public function withPost(Post $post): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'post' => $post,
        ]);
    }

    /**
     * Returns array with generation rules
     */
    public function definition(): array
    {
        return [
            'message' => $this->faker->sentence(12),
            'author' => UserFactory::new()->makeOne(),
            'post' => PostFactory::new()->makeOne()
        ];
    }
}
```

### PostFactory

Let's create `PostFactory` class:

```php
// app/database/Factory/PostFactory.php
namespace Database\Factory;

use App\Database\Post;
use App\Database\User;
use Faker\Generator;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class PostFactory extends AbstractFactory
{
    /**
     * Returns a fully qualified database entity class name
     */
    public function entity(): string
    {
        return Post::class;
    }

    /**
     * Returns an entity
     */
    public function makeEntity(array $definition): Post
    {
        return new Post($definition['title'], $definition['content'], $definition['author']);
    }

    /**
     * Generate Post with given author
     */
    public function withAuthor(User $author): self
    {
        return $this->state(fn(Generator $faker, array $definition) => [
            'author' => $author,
        ]);
    }

    /**
     * Returns array with generation rules
     */
    public function definition(): array
    {
        return [
            'title' => $this->faker->sentence(12),
            'content' => $this->faker->text(900),
            'author' => UserFactory::new()->makeOne()
        ];
    }
}
```

### UserFactory

Let's create `UserFactory` class:

```php
// app/database/Factory/UserFactory.php
namespace Database\Factory;

use App\Database\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

class UserFactory extends AbstractFactory
{
    /**
     * Returns a fully qualified database entity class name
     */
    public function entity(): string
    {
        return User::class;
    }

    /**
     * Returns an entity
     */
    public function makeEntity(array $definition): User
    {
        return new User($definition['name']);
    }

    /**
     * Returns array with generation rules
     */
    public function definition(): array
    {
        return [
            'name' => $this->faker->name,
        ];
    }
}
```

### BlogSeeder

Let's create `BlogSeeder` class:

```php
// app/database/Seeder/BlogSeeder.php
namespace Database\Seeder;

use Database\Factory\CommentFactory;
use Database\Factory\PostFactory;
use Database\Factory\UserFactory;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

class BlogSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        $users = UserFactory::new()->times(100)->make();
        yield from $users;

        $posts = [];
        for ($i = 0; $i < 1000; $i++) {
            $posts[] = PostFactory::new()
                ->withAuthor($users[array_rand($users)])
                ->makeOne();
        }
        yield from $posts;

        for ($i = 0; $i < 1000; $i++) {
            yield CommentFactory::new()
                ->withAuthor($users[array_rand($users)])
                ->withPost($posts[array_rand($posts)])
                ->makeOne();
        }
    }
}
```

Now let's execute a console command that will populate the database with test records:

```bash
php app.php db:seed
```

## Controller

Create a set of REST endpoints to retrieve the post data via API. We can start with a simple
controller, `App\Controller\PostController`. Create it using scaffolder:

```bash
php app.php create:controller post -a test -a get -p 
```

> **Note**
> Use option `-a` to pre-generate controller actions and option `-p` to pre-load prototype extension.

The generated code:

```php
namespace App\Controller;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class PostController
{
    use PrototypeTrait;

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function test(): ResponseInterface
    {
    }

    /**
     * Please, don't forget to configure the Route attribute or remove it and register the route manually.
     */
    #[Route(route: 'path', name: 'name')]
    public function get(): ResponseInterface
    {
    }
}
```

### Test Method

You can return various types of data from your controller methods. These are valid return values:

- string
- PSR-7 response
- array (as JSON)
- JsonSerializable object

> **Note**
> Use custom [domain core](/cookbook/domain-core.md) to perform domain-specific response transformations. You can also
> use the `$this->response` helper to write the data into PSR-7 response object.

For demo purposes, return `array`, the `status` key will be treated as response status.

```php
// ...
#[Route(route: '/api/test/<id>', name: "post.test", methods: 'GET')]
public function test(string $id): array
{
    return [
        'status' => 200,
        'data'   => [
            'id' => $id
        ]
    ];
}
``` 

Open `http://localhost:8080/api/test/123` to observe the result.

Alternatively, use the ResponseWrapper helper:

```php
use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/api/test/<id>', methods: 'GET')] 
public function test(string $id): ResponseInterface
{
    return $this->response->json(
        [
            'data' => [
                'id' => $id
            ]
        ],
        200
    );
}
```

> **Note**
> We won't use the test method going forward.

### Get Post

To get post details, use `PostRepository`, request such dependency in the constructor, `get` method, or use prototype
shortcut `posts`. You can access `id` via route parameter:

```php
namespace App\Controller;

use App\Database\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<id:\d+>', name: 'post.get', methods: 'GET')]
    public function get(string $id): array
    {
        /** @var Post $post */
        $post = $this->posts->findByPK($id);
        if ($post === null) {
            throw new NotFoundException('post not found');
        }

        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

You can replace direct repository access and use `Post` as method injection via connected `CycleInterceptor` (make sure
that `AppBootloader` is connected):

```php
namespace App\Controller;

use App\Database\Post;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<post:\d+>', name: 'post.get', methods: 'GET')]  
    public function get(Post $post): array
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }
}
```

> **Note**
> Consider using view object to map the response data into the `JsonSerializable` form.

### Post View Mapper

You can use any existing serialization solution (like `jms/serializer`) or write your own. Create a prototyped view
object to map post data into JSON format with comments:

```php
namespace App\View;

use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Core\Container\SingletonInterface;
use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;

#[Prototyped(property: 'postView')]
class PostView implements SingletonInterface
{
    use PrototypeTrait;

    public function map(Post $post): array
    {
        return [
            'post' => [
                'id'      => $post->id,
                'author'  => [
                    'id'   => $post->author->id,
                    'name' => $post->author->name
                ],
                'title'   => $post->title,
                'content' => $post->content,
            ]
        ];
    }

    public function json(Post $post): ResponseInterface
    {
        return $this->response->json($this->map($post), 200);
    }
}
```

> **Note**
> Run `php app.php configure` to generate the IDE highlight and register a prototyped class.

Modify the controller as follows:

```php
namespace App\Controller;

use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<post:\d+>', name: 'post.get', methods: 'GET')]   
    public function get(Post $post): ResponseInterface
    {
        return $this->postView->json($post);
    }
}
```

> **Note**
> You should observe no changes in the behavior.

### Get Multiple Posts

Use direct repository access to load multiple posts. To start, let's load all the available posts and their authors.

Create `findAllWithAuthors` method in `PostRepository`:

```php
namespace App\Repository;

use Cycle\ORM\Select;
use Cycle\ORM\Select\Repository;

class PostRepository extends Repository
{
    public function findAllWithAuthor(): Select
    {
        return $this->select()->load('author');
    }
}
``` 

Create method `list` in `PostController`:

```php
#[Route(route: '/api/post', name: 'post.list', methods: 'GET')]
public function list(): array
{
    $posts = $this->posts->findAllWithAuthor();

    return [
        'posts' => array_map([$this->postView, 'map'], $posts->fetchAll())
    ];
}
```

> **Note**
> You can see the JSON of all the posts using `http://localhost:8080/api/post`.

### Data Grid

The approach provided above has its limitations since you have to paginate, filter, and order the result manually.
Use the [Data Grid component](/component/data-grid.md) to handle data formatting for you:

```bash
composer require spiral/data-grid-bridge
``` 

Activate the `Spiral\DataGrid\Bootloader\GridBootloader` and `Spiral\Cycle\Bootloader\DataGridBootloader` in your
application.

To use data grids, we have to specify our data schema first, create `App\View\PostGrid` class:

```php
namespace App\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\IntValue;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'postGrid')] 
class PostGrid extends GridSchema
{
    public function __construct()
    {
        $this->addFilter('author', new Equals('author.id', new IntValue()));

        $this->addSorter('id', new Sorter('id'));
        $this->addSorter('author', new Sorter('author.id'));

        // default limit, available limits
        $this->setPaginator(new PagePaginator(10, [10, 20, 50]));
    }
}
```

> **Note**
> We have added one filter and two sorting options to the grid. The pagination is done using page limits.

Connect the bootloader to your method using `Spiral\DataGrid\GridFactory`:

```php
#[Route(route: '/api/post', name: 'post.list', methods: 'GET')] 
public function list(GridFactory $grids): array
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return [
        'posts' => \array_map(
            [$this->postView, 'map'],
            \iterator_to_array($grid->getIterator())
        )
    ];
}
```

> **Note**
> Do not forget to run `php app.php configure` after adding a prototyped class.

The grids are a very flexible component with many customization options. By default, the grid is configured to read
values from request query and data.

| URL                                                                  | Comment                                  |
|----------------------------------------------------------------------|------------------------------------------|
| `http://localhost:8080/api/post?paginate[page]=2`                    | Open second page.                        |
| `http://localhost:8080/api/post?paginate[page]=2&paginate[limit]=20` | Open second page with 20 posts per page. |
| `http://localhost:8080/api/post?sort[id]=desc`                       | Sort by post->id DESC.                   |
| `http://localhost:8080/api/post?sort[author]=asc`                    | Sort by post->author->id.                |
| `http://localhost:8080/api/post?filter[author]=1`                    | Find only posts with given author id.    |

You can use sorters, filters, and pagination in one request. Multiple filters can be activated at once.

## Validate Request

To validate the request, we will use the [spiral/validator](https://github.com/spiral/validator) package.

> **Note**
> Read more about request validation [here](../filters/configuration.md).

To install the package:

```bash
composer require spiral/validator
```

Add the bootloader to `LOAD` in `app/src/Application/Kernel.php` to activate the component:

```diff
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel

         ValidationBootloader::class,
+        \Spiral\Validator\Bootloader\ValidatorBootloader::class,
     ];
 }
```

Create `CommentFilter`:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class CommentFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public readonly string $message;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'message' => ['string', 'required']
        ]);
    }
}
```

### Service

Create `App\Service\CommentService`:

```php
namespace App\Service;

use App\Database\Comment;
use App\Database\Post;
use App\Database\User;
use Cycle\ORM\EntityManagerInterface;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'commentService')]
class CommentService
{
    public function __construct(
        private EntityManagerInterface $entityManager
    ) {
    }

    public function comment(Post $post, User $user, string $message): Comment
    {
        $comment = new Comment($message, $user, $post);

        $this->entityManager->persist($comment);
        $this->entityManager->run();

        return $comment;
    }
}
```

### Controller Action

Declare controller method and request filter instance. If a filter class
implements `Spiral\Filters\Model\HasFilterDefinition` interface, the Filters component will guarantee that the filter is
valid. Create `comment` endpoint to post a message to the given post:

```php
#[Route(route: '/api/post/<post:\d+>/comment', name: 'post.comment', methods: 'POST')]
public function comment(Post $post, CommentFilter $commentFilter): array
{
    $this->commentService->comment(
        $post,
        $this->users->findOne(),
        $commentFilter->message
    );

    return ['status' => 201];
}
```

> **Note**
> Use `spiral/toolkit` Stempler extension to create AJAX-native forms in HTML.

### Execute

Check the error format:

```bash
curl -X POST -H 'content-type: application/json' --data '{}' http://localhost:8080/api/post/1/comment
``` 

Response:

```json
{
  "status": 400,
  "errors": {
    "message": "This value is required."
  }
}
```

Or not found exception when the post can not be found:

```bash
curl -X POST -H 'content-type: application/json' --data '{"message":"test"}' http://localhost:8080/api/post/9999/comment
``` 

> **Note**
> Make sure to send `accept: application/json` to receive an error in JSON format.

To post a valid comment:

```bash
curl -X POST -H 'content-type: application/json' --data '{"message": "first comment"}' http://localhost:8080/api/post/1/comment
```

> **Note**
> Read more about filters [here](/filters/filter.md).

## Render Template

To render post information into HTML form, use [views](/views/configuration.md)
and [Stempler](/stempler/configuration.md) component. Pass a post list to the view using Grid object.

```php
#[Route(route: '/posts', name: 'post.all', methods: 'GET')] 
public function all(GridFactory $grids): string
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return $this->views->render('posts', ['posts' => $grid]);
}
```

### Create Layout

Create/edit a layout file located in `app/views/layout/base.dark.php`:

```html
<!DOCTYPE html>
<html>
<head>
    <title>${title}</title>
    <link href="https://stackpath.bootstrapcdn.com/bootstrap/4.4.1/css/bootstrap.min.css" rel="stylesheet">
</head>
<body>
<block:body/>
</body>
</html>
```  

### List Posts

Create a view file `app/views/posts.dark.php` and extend parent layout.

```html

<extends:layout.base title="Posts"/>

<define:body>
    @foreach($posts as $post)
    <div class="post">
        <div class="title">{{$post->title}}</div>
        <div class="author">{{$post->author->name}}</div>
    </div>
    @endforeach
</define:body>
```

> **Note**
> You can now see the list of posts on `http://localhost:8080/posts`, use URL Query parameters to control Data Grid
> filters, sorters (`http://localhost:8080/posts?paginate[page]=2`).

### View Post

To view the post and all of its comments, create a new controller method in `PostController`. Load the post manually via
repository to preload all author and comment information.

```php
use Spiral\Http\Exception\ClientException\NotFoundException;
// ...

#[Route(route: '/post/<id:\d+>', name: 'post.view', methods: 'GET')] 
public function view(string $id): string
{
    $post = $this->posts->findOneWithComments($id);
    if ($post === null) {
        throw new NotFoundException();
    }

    return $this->views->render('post', ['post' => $post]);
}
```

This is where the repository method is:

```php
public function findOneWithComments(string $id): ?Post
{
    return $this
        ->select()
        ->wherePK($id)
        ->load('author')
        ->load(
            'comments.author',
            [
                'load' => function (Select\QueryBuilder $q) {
                    // last comments at top
                    $q->orderBy('id', 'DESC');
                }
            ]
        )
        ->fetchOne();
}
```

And corresponding view `app/views/post.dark.php`:

```html

<extends:layout.base title="Posts"/>

<define:body>
    <div class="post">
        <div class="title">{{$post->title}}</div>
        <div class="author">{{$post->author->name}}</div>
    </div>
    <div class="comments">
        @foreach($post->comments as $comment)
        <div class="comment">
            <div class="message">{{$comment->message}}</div>
            <div class="author">{{$comment->author->name}}</div>
        </div>
        @endforeach
    </div>
</define:body>
```

Open the post page using `http://localhost:8080/post/1`.

> **Note**
> We are leaving styling and comment timestamps up to you.

### Route

Use `post.view` route name to generate a link in `app/views/posts.dark.php`:

```html

<extends:layout.base title="Posts"/>

<define:body>
    @foreach($posts as $post)
    <div class="post">
        <div class="title">
            <a href="@route('post.view', ['id' => $post->id])">{{$post->title}}</a>
        </div>
        <div class="author">{{$post->author->name}}</div>
    </div>
    @endforeach
</define:body>
```

> **Note**
> Read more about Stempler Directives [here](/stempler/directives.md).

## Next Steps

Spiral provides a lot of pre-build functionality for you. Read the following sections to gain more insights:

- [Authenticating users](/security/authentication.md)
- [Authorize Access](/security/rbac.md)
- [Background Jobs](/queue/configuration.md)

## Source Code

Source code of demo project - https://github.com/spiral/demo

Make sure to run to install the project:

```bash
vendor/bin/rr get
php app.php migrate:init
php app.php migrate
php app.php configure -vv
```
