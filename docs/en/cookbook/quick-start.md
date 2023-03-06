# Cookbook — Long Start

Spiral contains a lot of components built to operate seamlessly with each other. In this article, we will show you how
to create a demo blog application with REST API, ORM, Migrations, request validation, custom annotations (optional) and
domain interceptors.

> **Note**
> The components and approaches will be covered at basic levels only. Read the corresponding sections to gain more
> information. You can find the demo repository [here](https://github.com/spiral/demo).

## Installation

To get started with building your simple application, you can easily install the default `spiral/app` bundle with most
of the required components by running the following command:

```terminal
composer create-project spiral/app spiral-demo
```

During the installation process, you will be prompted to select various options with the Spiral installer, such as the
application preset, whether to use Cycle ORM, which collections to use, which validator component to use, and so on. For
this tutorial, we recommend choosing the options shown above:

```terminal
✔ Which application preset do you want to install? > Web
✔ Create a default application structure and demo data? > Yes
✔ Would you like to use SAPI? > No
✔ Do you need Cycle ORM? > Yes
✔ Which Collections do you want to use with Cycle ORM? > Doctrine Collections
✔ Which validator component do you want to use? > Spiral Validator
✔ Do you want to use Queue component? > No
✔ Do you want to use Cache component? > No
✔ Do you want to use Mailer Component? > No
✔ Do you want to use Storage component? > No
✔ Which template engine do you want to use? > Stempler
✔ Do you want to use the Event Dispatcher? > No
✔ Do you need a cron jobs scheduler? > No
✔ Do you need the Temporal? > No
✔ Do you need the RoadRunner Metrics? > No
✔ Do you need the Sentry? > No
```

Once the installation is complete, you can start the server and open your application immediately by running the
following command:

```terminal
cd spiral-demo

./rr serve
```

You've just started [RoadRunner](../start/server.md) server. The same server can be used on production, making your 
development environment similar to the final setup. Out of the box, the server includes instruments to write portable
applications with HTTP/2, [GRPC](../grpc/configuration.md), [Queue](../queue/configuration.md),
[WebSockets](../websockets/configuration.md), etc. and does not require external brokers to operate.

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

```yaml .rr.yaml
http:
  ...
  pool:
    debug: true
```

You can also create and use an alternative configuration file via `-c` flag of the `rr` application.

```terminal
./rr serve -c .rr.dev.yaml
```

> **See more**
> Read more about workers in the Official RoadRunner [documentation](https://roadrunner.dev/docs/app-server-cli/2.x/en).

### Lighter Up

We won't need translation, session, GRPC, cookies, CSRF, routing configuration via RoutesBootloader in our demo
application. Remove these components and their bootloaders.

Delete the following bootloaders from `app/src/Application/Kernel.php`:

```php app/src/Application/Kernel.php
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

> **Note**
> Read more about Bootloaders [here](../framework/bootloaders.md).

Remove `App\Endpoint\Web\Middleware\LocaleSelector` from method `globalMiddleware` in
the `App\Application\Bootloader\RoutesBootloader`:

```diff app/scr/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
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

```diff app/scr/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
     ...
    
-    protected function defineRoutes(RoutingConfigurator $routes): void
-    {
-        ...
-    }
 }
```

> **See more**
> Read more about routing in the [HTTP — Routing](../http/routing.md) section.

Remove all middleware groups from the method `middlewareGroups`:

```diff app/scr/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
    ...
    
    protected function middlewareGroups(): array
    {
        return [
-            'web' => [
-                ...
-            ],
-            'api' => [
-                ...
-            ],
        ];
    }
 }
```

Remove unused imports:

```diff app/scr/Application/Bootloader/RoutesBootloader.php
- use Spiral\Cookies\Middleware\CookiesMiddleware;
- use Spiral\Csrf\Middleware\CsrfMiddleware;
- use Spiral\Router\Loader\Configurator\RoutingConfigurator;
- use Spiral\Session\Middleware\SessionMiddleware;

final class RoutesBootloader extends BaseRoutesBootloader
{
     ...
}
```

Delete the following files and directories as no longer required:

- `app/locale`
- `app/routes`

> **Note**
> Note that the application won't work at the moment as we removed the dependency required to
> render `app/views/home.dark.php`.

### Database Connection

Our application needs a database to operate. By default, the database configuration is located
in `app/config/database.php` file. The demo application comes with a pre-configured SQLite database located in
`runtime/runtime.db`.

```php app/config/database.php
use Cycle\Database\Config;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'default'],
    ],
    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(
            connection: new Config\SQLite\FileConnectionConfig(
                database: directory('runtime') . '/db.sqlite'
            ),
            queryCache: true
        ),
        // ...
    ],
];
```

We can store a database name, username, password and port in `.env` file, add the following lines into it:

```dotenv .env
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

```php app/config/database.php
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

```terminal
php app.php db:list
```

> **See more**
> Read more about Databases [here](../basics/orm.md).

### Connect Database Seeder

We will need some sample data for the application.
Let's install [Database Seeder](https://github.com/spiral-packages/database-seeder).

```terminal
composer require spiral-packages/database-seeder --dev
``` 

Add the bootloader `Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader` to `LOAD` section to activate the
package:

```diff app/src/Application/Kernel.php
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class Kernel extends Kernel

         RoadRunnerBridge\CommandBootloader::class,
+        \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
     ];
 }
```

### Creating Routes

We can use this attribute in our controller as follows:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Router\Annotation\Route;

final class HomeController
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

> **See more**
> Read more about the extension [here](../http/routing.md).

Run CLI command to check the list of available routes:

```terminal
php app.php route:list
```

> **Note**
> Use additional route parameters to configure middleware, route group, etc.

In the following examples, we will stick to the routes with attributes for simplicity.

To flush route cache (when `DEBUG` disabled):

```terminal
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

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Domain\GuardInterceptor;
use Spiral\Filter\ValidationHandlerMiddleware;

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
Add the bootloader to `APP`:

```diff app/src/Application/Kernel.php
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel
+        Bootloader\AppBootloader::class,
     ];
 }
```

> **Note**  
> Read more about Http interceptors [here](../http/interceptors.md).

## Scaffold Database

The framework can configure the database schema using a set of migration files. To configure migrations in your
application, run:

```terminal
php app.php migrate:init
```

You can now observe the migration table structure by using:

```terminal
php app.php db:list
php app.php db:table migrations
```

You can write the migration manually, or let Cycle ORM generate it for you.

> **Note**  
> Read more about migrations [here](https://cycle-orm.dev/docs/database-migrations).
> Use [Scaffolder](../basics/scaffolding.md) component to create migrations manually.

### Define ORM Entities

The demo application comes with [Cycle ORM](https://cycle-orm.dev). By default, you can use attributes to configure
your entities.

Enable the Cycle Bridge `Spiral\Cycle\Bootloader\ScaffolderBootloader` bootloader to use commands that create
entities and repositories. Add the bootloader to `LOAD`:

```diff app/src/Application/Kernel.php
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel
        Scaffolder\ScaffolderBootloader::class,
+       CycleBridge\ScaffolderBootloader::class,
     ];
 }
```

Let's create `Post`, `User` and `Comment` entities and their repositories using the Scaffolder extension:

```terminal
php app.php create:entity post -f id:primary -f title:string -f content:text -e
php app.php create:entity user -f id:primary -f name:string -e
php app.php create:entity comment -f id:primary -f message:string
``` 

> **Note**
> Observe the classes generated in `app/src/Database` and `app/src/Repository`.

Post:

```php app/src/Domain/Blog/Entity/Post.php
namespace App\Domain\Blog\Entity;

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

```php app/src/Domain/Blog/Entity/Post.php
use App\Domain\Blog\Repository\PostRepository;
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

```php app/src/Domain/Blog/Entity/User.php
use App\Domain\Blog\Repository\UserRepository;
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

```php app/src/Domain/Blog/Entity/User.php
use App\Domain\Blog\Repository\UserRepository;
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

```php app/src/Domain/Blog/Entity/Comment.php
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

```php app/src/Domain/Blog/Entity/Comment.php
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

:::: tabs

::: tab PostRepository

```php app/src/Domain/Blog/Repository/PostRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'posts')]
class PostRepository extends Repository
{
}
```

:::

::: tab UserRepository

```php app/src/Domain/Blog/Repository/UserRepository.php
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'users')]
class UserRepository extends Repository
{
}
```

:::

::::

You can change the default directory mapping, headers, and others using [Scaffolder config](../basics/scaffolding.md).

> **Note**
> Read more about Cycle [here](../basics/orm.md).

Run the configure command to collect all available prototype classes:

```terminal
php app.php configure
```

### Generate Migration

To generate the database schema, run:

```terminal
php app.php cycle:migrate -v
```

The generated migration is located in `app/migrations/`. Execute it using:

```terminal
php app.php migrate -vv
```

You can now observe the generated tables using `db:list` command.

### Create Relations

Use [attributes](https://cycle-orm.dev/docs/annotated-relations) to define the relations between entities. Configure
Post and Comment relations with type BelongsTo and add relation HasMany between User and Post.

Post:

```php app/src/Domain/Blog/Entity/Post.php
use App\Domain\Blog\Repository\PostRepository;
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

```php app/src/Domain/Blog/Entity/Comment.php
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

```terminal
php app.php cycle:migrate -v
php app.php migrate -vv
```

> **Note**
> You can generate and run the migration in one command using `php app.php cycle:migrate -r`.

You can check the presence of Foreign Keys:

```terminal
php app.php db:table comments
```

> **Note**
> Do not forget to run `php app.php cycle:migrate` when you change any of your entities.

## Factories and seeders

To generate test data, we need factories that will describe the rules for generating an entity.
And seeders that will fill the database.

We will store them separately from the application code, in the `app/database` folder.
Let's add a separate `Database` namespace to Composer autoload:

```diff composer.json
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

```terminal
composer dump-autoload
```

### CommentFactory

Let's create `CommentFactory` class, extend it from `Spiral\DatabaseSeeder\Factory\AbstractFactory` and implement
required methods:

```php app/database/Factory/CommentFactory.php
namespace Database\Factory;

use App\Domain\Blog\Entity\Comment;
use App\Domain\Blog\Entity\Post;
use App\Domain\Blog\Entity\User;
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

```php app/database/Factory/PostFactory.php
namespace Database\Factory;

use App\Domain\Blog\Entity\Post;
use App\Domain\Blog\Entity\User;
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

```php app/database/Factory/UserFactory.php
namespace Database\Factory;

use App\Domain\Blog\Entity\User;
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

```php app/database/Seeder/BlogSeeder.php
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

```terminal
php app.php db:seed
```

## Controller

Create a set of REST endpoints to retrieve the post data via API. We can start with a simple
controller, `App\Endpoint\Web\PostController`. Create it using scaffolder:

```terminal
php app.php create:controller post -a test -a get -p 
```

> **Note**
> Use option `-a` to pre-generate controller actions and option `-p` to pre-load prototype extension.

The generated code:

```php app/src/Endpoint/Web/PostController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
{
    use PrototypeTrait;

    #[Route(route: 'path', name: 'name')]
    public function test(): ResponseInterface
    {
    }

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
> Use custom [domain core](../http/interceptors.md) to perform domain-specific response transformations. You can also
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

```php app/src/Endpoint/Web/PostController.php
use App\Domain\Blog\Entity\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
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

```php app/src/Endpoint/Web/PostController.php
use App\Domain\Blog\Entity\Post;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
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

```php app/src/Endpoint/Web/View/PostView.php
namespace App\Endpoint\Web\View;

use App\Domain\Blog\Entity\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Core\Container\SingletonInterface;
use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;

#[Prototyped(property: 'postView')]
final class PostView implements SingletonInterface
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

```php app/src/Endpoint/Web/PostController.php
use App\Domain\Blog\Entity\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class PostController
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

```php app/src/Domain/Blog/Repository/PostRepository.php
use Cycle\ORM\Select;
use Cycle\ORM\Select\Repository;

final class PostRepository extends Repository
{
    public function findAllWithAuthor(): Select
    {
        return $this->select()->load('author');
    }
}
``` 

Create method `list` in `PostController`:

```php app/src/Endpoint/Web/PostController.php
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
Use the [Data Grid component](../component/data-grid.md) to handle data formatting for you:

```terminal
composer require spiral/data-grid-bridge
``` 

Activate the `Spiral\DataGrid\Bootloader\GridBootloader` and `Spiral\Cycle\Bootloader\DataGridBootloader` in your
application.

To use data grids, we have to specify our data schema first, create `App\View\PostGrid` class:

```php app/src/Endpoint/Web/View/PostGrid.php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\IntValue;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'postGrid')] 
final class PostGrid extends GridSchema
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

```php app/src/Endpoint/Web/PostController.php
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

Add the bootloader to `LOAD` section to activate the component:

```diff app/src/Application/Kernel.php
--- a/app/src/Application/Kernel.php
+++ b/app/src/Application/Kernel.php
@@ -85,5 +85,6 @@ class App extends Kernel

         ValidationBootloader::class,
+        \Spiral\Validator\Bootloader\ValidatorBootloader::class,
     ];
 }
```

Create `CommentFilter`:

```php app/src/Endpoint/Web/Filter/CommentFilter.php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CommentFilter extends Filter implements HasFilterDefinition
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

Create `App\Domain\Blog\Service\CommentService`:

```php app/src/Domain/Blog/Service/CommentService.php
use App\Domain\Blog\Entity\Comment;
use App\Domain\Blog\Entity\Post;
use App\Domain\Blog\Entity\User;
use Cycle\ORM\EntityManagerInterface;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'commentService')]
final class CommentService
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

```php app/src/Endpoint/Web/PostController.php
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

```json HTTP response
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
> Read more about filters [here](../filters/filter.md).

## Render Template

To render post information into HTML form, use [views](../views/configuration.md)
and [Stempler](../stempler/configuration.md) component. Pass a post list to the view using Grid object.

```php app/src/Endpoint/Web/PostController.php
#[Route(route: '/posts', name: 'post.all', methods: 'GET')] 
public function all(GridFactory $grids): string
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return $this->views->render('posts', ['posts' => $grid]);
}
```

### Create Layout

Create/edit a layout file located in `app/views/layout/base.dark.php`:

```html app/views/layout/base.dark.php
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

```html app/views/posts.dark.php
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

```php app/src/Endpoint/Web/PostController.php
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

```php app/src/Domain/Blog/Repository/PostRepository.php
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

```html app/views/post.dark.php
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

```html app/views/posts.dark.php
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
> Read more about Stempler Directives [here](../stempler/directives.md).

<hr>

## What's next?

Spiral provides a lot of pre-build functionality for you. Read the following sections to gain more insights:

- [Authenticating users](../security/authentication.md)
- [Authorize Access](../security/rbac.md)
- [Background Jobs](../queue/configuration.md)

## Source Code

Source code of demo project - https://github.com/spiral/demo

Make sure to run to install the project:

```terminal
./vendor/bin/rr get
php app.php migrate:init
php app.php migrate
php app.php configure -vv
```
