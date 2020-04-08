# Long Start
The spiral framework contains a lot of components built to operate seamlessly with each other.
In this article, we will show how to create a demo blog application with REST API, ORM, Migrations, request validation, custom annotations (optional) and domain interceptors.

> The components and approaches will be covered at basic levels only. Read the corresponding sections to gain more information.

## Installation
Use composer to install the default `spiral/app` bundle with most of the components out of the box:

```bash
$ composer create-project spiral/app spiral-demo
$ cd spiral-demo
```

> Use the different build `spiral/app-cli` to install the spiral with minimal dependencies.

If everything installed correctly, you could open your application immediately by starting the server:

```bash
$ ./spiral serve -v -d
```

You just started an [application server](/framework/application-server.md). The same server can be used on production, making your
development environment similar to the final setup. Out of the box, the server includes instruments to write portable applications
with HTTP/2, GRPC, Queue, WebSockets, etc. and does not require external brokers to operate.

By default, the application available on `http://localhost:8080`. The build includes multiple pre-defined pages you can play with.

> Check the exception page `http://localhost:8080/exception.html`, at the right part of this page you can see all interceptors
> and middleware included in the default build. We will turn some of them off to make the application runtime smaller. 

## Configure
Spiral applications configured using config files located in `app/config`, you can use the hardcoded values for the configuration, 
or get the values using available functions `env` and `directory`. The `spiral/app` bundle use DotEnv extension which 
will load ENV variables from the `.env` file.

> Tweak the application server and its plugins using `.rr.yaml` file.

The application dependencies defined in `composer.json` and activated in `app/src/App.php` as Bootloaders.
The default build includes quite a lot of pre-configured components.

### Developer Mode
To simplify the tweaking of the application, restart the application server in developer mode. In this mode, the server uses
only one worker and reloads it after every request.

```bash
$ ./spiral serve -v -d -o "http.workers.pool.maxJobs=1" -o "http.workers.pool.numWorkers=1"
```

You can also create and use an alternative configuration file via `-c` flag of the `spiral` application.

> Read more about Workers and Lifecycle [here](/start/workers.md).

### Lighter Up
We won't need translation, session, cookies, CSRF, and encryption in our demo application. Remove these components and their bootloaders.

Delete following bootloaders from `app/src/App.php`:

```php
Framework\I18nBootloader::class,
Framework\Security\EncrypterBootloader::class,

// from http
Framework\Http\CookiesBootloader::class,
Framework\Http\SessionBootloader::class,
Framework\Http\CsrfBootloader::class,
Framework\Http\PaginationBootloader::class,

// from views 
Framework\Views\TranslatedCacheBootloader::class,

// from APP
Bootloader\LocaleSelectorBootloader::class,
```

You can delete the corresponding dependencies in `composer.json` as well:

```bash
"spiral/cookies": "^1.0",
"spiral/csrf": "^1.0",
"spiral/session": "^1.1",
"spiral/translator": "^1.2",
"spiral/encrypter": "^1.1",
```

Delete following files and directories as no longer required:
- `app/locale`
- `app/src/Bootloader/LocaleSelectorBootloader.php`
- `app/src/Middleware`.

> Note, the application won't work at the moment as we removed the dependency required to render `app/views/home.dark.php`.

### Database Connection
Our application needs a database to operate. By default, the database configuration located in `app/config/database.php` file.
The demo application comes with pre-configured SQLite database located in `runtime/runtime.db`.

```php
// database.php 

use Spiral\Database\Driver;

return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'runtime'],
    ],
    'drivers'   => [
        'runtime' => [
            'driver'     => Driver\SQLite\SQLiteDriver::class,
            'options'    => [
                'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
            ]
        ],
    ]
];
```

We can store the database name, username and password in `.env` file, add the following lines into it:

```dotenv
DB_HOST = localhost
DB_NAME = name
DB_USER = username
DB_PASSWORD = password
```

> Change the values to match your database parameters.

To change the default database to MySQL, change the `drivers` section of the configuration, use `env` function to access
the ENV variables.

```php
return [
    'default'   => 'default',
    'databases' => [
        'default' => ['driver' => 'default'],
    ],
    'drivers'   => [
        'default' => [
            'driver'     => Driver\MySQL\MySQLDriver::class,
            'connection' => sprintf('mysql:host=%s;dbname=%s', env('DB_HOST'), env('DB_NAME')),
            'username'   => env('DB_USER'),
            'password'   => env('DB_PASSWORD'),
        ],
    ]
];
```

> Note that `default` database now points to the `default` connection.

To check that the database connection was successful run:

```bash
$ php app.php db:list
```

> Read more about Databases [here](/database/configuration.md).

### Connect Faker
We will need some sample data for the application. Let's connect the library and bootload the library `fzaninotto/faker`.

```bash
$ composer require fzaninotto/faker
``` 

To generate the stub data, we will need an instance of `Faker\Generator`, create bootloader in `app/src/Bootloader` to resolve such
instance as a singleton. Use a factory method for that purpose.

```php
namespace App\Bootloader;

use Faker\Factory;
use Faker\Generator;
use Spiral\Boot\Bootloader\Bootloader;

class FakerBootloader extends Bootloader
{
    protected const SINGLETONS = [
        Generator::class => [self::class, 'fakerGenerator']
    ];

    private function fakerGenerator(): Generator
    {
        return Factory::create(Factory::DEFAULT_LOCALE);
    }
}
```

Add the bootloader to `LOAD` or `APP` in `app/src/App.php` to activate the component:

```diff
--- a/app/src/App.php
+++ b/app/src/App.php
@@ -85,5 +85,6 @@ class App extends Kernel

         // fast code prototyping
         Prototype\PrototypeBootloader::class,
+        Bootloader\FakerBootloader::class,
     ];
 }
```

> You can request dependencies as method arguments in the factory method `fakerGenerator`.

Use the `Faker\Generator` in your controller to view the stub data at `http://localhost:8080/`:

```php
namespace App\Controller;

use Faker\Generator;

class HomeController
{
    public function index(Generator $generator)
    {
        return $generator->sentence(128);
    }
}
```

> Read more about Bootloaders [here](/framework/bootloaders.md).

### Routing
By default, the routing rules located in `app/src/Bootloader/RoutesBootloader.php`. You have many options on how
to configure routing. Point route to actions, controllers, controller groups, set the default pattern parameters, 
verbs, middleware, etc.

Create a simple route to point all of the URLs to the `App\Controller\HomeController`:

```php
namespace App\Bootloader;

use App\Controller\HomeController;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Controller;

class RoutesBootloader extends Bootloader
{
    public function boot(RouterInterface $router): void
    {
        $route = new Route('/[<action>[/<id>]]', new Controller(HomeController::class));
        $route = $route->withDefaults(['action' => 'index']);

        $router->setRoute('home', $route);
    }
}
```

In the given setup, the action and id are the optional parts of the URL (see the `[]`), the action defaults to `index`.
Open `HomeController`->`index` using `http://localhost:8080/` or `http://localhost:8080/index`.

The route parameters can be addressed in method injection of the controller by their name, create the following method in `HomeController`:

```php
public function open(string $id)
{
    dump($id);
}
```

You can invoke this method using URL `http://localhost:8080/open/123`. The `id` parameter will be hydrated automatically.

> Read more about Routing [here](/http/routing.md).

### Annotated Routing
Though the framework does not provide the annotated routing support out of the box yet, it is possible to [configure it](/cookbook/annotated-routes.md)
manually using existing instruments. 

> This is optional segment for Symfony users.

#### Annotation
Build simple annotation which later can be attached to public controller methods:

```php
namespace App\Annotation;

use Doctrine\Common\Annotations\Annotation;

/**
 * @Annotation()
 * @Annotation\Target({"METHOD"})
 * @Annotation\Attributes({
 *      @Annotation\Attribute("action", type="string", required=true),
 *      @Annotation\Attribute("verbs", type="array"),
 * })
 */
class Route
{
    /** @var string */
    public $action;

    /** @var string[]|null */
    public $verbs;
}
```

> Default Web build of application enables annotation support by default.

#### Bootloader
Change `RoutesBootloader` to convert annotations into routes. Use class `Spiral\Annotations\AnnotationLocator`
to find available annotations across application codebase.

```php
namespace App\Bootloader;

use Spiral\Annotations\AnnotationLocator;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Router\Route;
use Spiral\Router\RouterInterface;
use Spiral\Router\Target\Action;

class RoutesBootloader extends Bootloader
{
    public function boot(AnnotationLocator $annotationLocator, RouterInterface $router): void
    {
        $methods = $annotationLocator->findMethods(\App\Annotation\Route::class);

        foreach ($methods as $method) {
            $name = sprintf(
                "%s.%s",
                $method->getClass()->getShortName(),
                $method->getMethod()->getShortName()
            );

            $route = new Route(
                $method->getAnnotation()->action,
                new Action(
                    $method->getClass()->getName(),
                    $method->getMethod()->getName()
                )
            );

            $route = $route->withVerbs(...(array)$method->getAnnotation()->verbs);

            $router->setRoute($name, $route);
        }
    }
}
```

#### Controller
We can use this annotation in our controller as follows:

```php
namespace App\Controller;

use App\Annotation\Route;

class HomeController
{
    /**
     * @Route(action="/", verbs={"GET"})
     */
    public function index()
    {
        return 'hello world';
    }
    
    /**
     * @Route(action="/open/<id>", verbs={"GET"})
     */
    public function open(string $id)
    {
        dump($id);
    }
}
```

Run CLI command to check the list of available routes:

```bash
$ php app.php route:list
```

> Use additional route parameters to configure middleware, common prefix, etc.

In the following examples, we will stick to the annotated routes for simplicity.

> Disable `ErrorHandlerBootloader` in `App` to view full error log.

### Domain Core
Connect custom controller interceptor (domain-core) to enrich your domain layer with additional functionality.
We can change the default behavior of the application and enable Cycle Entity resolution using route parameter, 
Filter validation and @Guard annotation.

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        Domain\FilterInterceptor::class, 
        Domain\CycleInterceptor::class,
        Domain\GuardInterceptor::class,
    ];
}
```

Enable the domain core in your application. We will demonstrate the use of the interceptor below.
  
> Read more about Domain Cores [here](/cookbook/domain-core.md).

## Scaffold Database
The framework can configure the database schema using a set of migration files. To configure migrations in your application
run:

```bash
$ php app.php migrate:init
```

You can now observe the migration table structure using:

```bash
$ php app.php db:list
$ php app.php db:table migrations
```

You can write the migration manually, or let Cycle ORM generate it for you.

> Read more about migrations [here](/database/migrations.md). Use [Scaffolder](/basics/scaffolding.md) component to create migrations manually. 

### Define ORM Entities
The demo application comes with [Cycle ORM](https://cycle-orm.dev). By default, you can use annotations to configure
your entities.

Let's create `Post`, `User` and `Comment` entities and their repositories using the Scaffolder extension:

```bash
$ php app.php create:entity post -f id:primary -f title:string -f content:text -e
$ php app.php create:entity user -f id:primary -f name:string -e
$ php app.php create:entity comment -f id:primary -f message:string
``` 

> Observe the classes generated in `app/src/Database` and `app/src/Repository`.

Post: 

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository = "App\Repository\PostRepository")
 */
class Post
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $title;

    /**
     * @Cycle\Column(type = "text")
     */
    public $content;
}

```

User:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository = "App\Repository\UserRepository")
 */
class User
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $name;
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class Comment
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $message;
}
```

You can change the default directory mapping, headers, and others using [Scaffolder config](/basics/scaffolding.md).

> Read more about Cycle [here](/cycle/configuration.md). Configure auto-timestamps using [custom mapper](https://cycle-orm.dev/docs/advanced-timestamp).

### Generate Migration
To generate the database schema run:

```bash
$ php app.php cycle:migrate -v
```

The generated migration located in `app/migrations/`. Execute it using:

```bash
$ php app.php migrate -vv
```

You can now observe the generated tables using `db:list` command.

### Create Relations
Use [annotations](https://cycle-orm.dev/docs/annotated-relations) to define the relations between entities. Configure
Post and Comment to belong to User and Post has many Comments. 

Post:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;
use Doctrine\Common\Collections\ArrayCollection;

/**
 * @Cycle\Entity(repository = "App\Repository\PostRepository")
 */
class Post
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $title;

    /**
     * @Cycle\Column(type = "text")
     */
    public $content;

    /**
     * @Cycle\Relation\BelongsTo(target = "User", nullable = false)
     * @var User
     */
    public $author;

    /**
     * @Cycle\Relation\HasMany(target = "Comment")
     * @var ArrayCollection|Comment
     */
    public $comments;

    public function __construct()
    {
        $this->comments = new ArrayCollection();
    }
}
```

Comment:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity()
 */
class Comment
{
    /**
     * @Cycle\Column(type = "primary")
     */
    public $id;

    /**
     * @Cycle\Column(type = "string")
     */
    public $message;

    /**
     * @Cycle\Relation\BelongsTo(target = "User", nullable = false)
     * @var User
     */
    public $author;

    /**
     * @Cycle\Relation\BelongsTo(target = "Post", nullable = false)
     * @var Post
     */
    public $post;
}
``` 

Once again generate and run the migration:

```bash
$ php app.php cycle:migrate -v
$ php app.php migrate -vv
```

> You can generate and run the migration in one command using `php app.php cycle:migrate -r`.

You can check the presence of Foreign Keys:

```bash
$ php app.php db:table comments
```

> Do not forget to run `php app.php cycle:migrate` when you change any of your entities.

## Service
Isolate the business logic into a separate service layer. Let's create `PostService` in `app/src/Service`.
We will need an instance of `Cycle\ORM\TransactionInterface` to persist the post.

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;

class PostService
{
    private $tr;

    public function __construct(TransactionInterface $tr)
    {
        $this->tr = $tr;
    }

    public function createPost(User $user, string $title, string $content): Post
    {
        $post = new Post();
        $post->author = $user;
        $post->title = $title;
        $post->content = $content;

        $this->tr->persist($post);
        $this->tr->run();

        return $post;
    }
}
```

> You can reuse the transaction after the `run` method.

### Prototyping
One of the most powerful capabilities of the framework is [Prototyping](/basics/prototype.md). Declare the shortcut `postService`, which points to `PostService` using annotation.

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="postService")
 */
class PostService
{
    // ...
}
``` 

Run the configure command to collect all available prototype classes:

```bash
$ php app.php configure
```

> Make sure to use proper IDE to gain access to the IDE tooltips.

Now you can get access to the `PostService` using `PrototypeTrait`, see the example down below.

## Console Command
Let's create three commands to generate the data for our application. Use scaffolder extension to create command to seed our database:

```bash
$ php app.php create:command seed/user seed:user
$ php app.php create:command seed/post seed:post
$ php app.php create:command seed/comment seed:comment
``` 

Generated commands will be available in `app/src/Command/Seed`.

### UserCommand
Use the method injection on `perform` in `UserCommand` to seed the users using Faker:

```php
// app/src/Command/Seed/UserCommand.php
namespace App\Command\Seed;

use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Faker\Generator;
use Spiral\Console\Command;

class UserCommand extends Command
{
    protected const NAME = 'seed:user';

    protected function perform(TransactionInterface $tr, Generator $faker): void
    {
        for ($i = 0; $i < 100; $i++) {
            $user = new User();
            $user->name = $faker->name;

            $tr->persist($user);
        }

        $tr->run();
    }
}
```

Run the command:

```bash
$ php app.php seed:user
```

### PostCommand
Use the prototype extension to speed up the creation of the `seed:post` command. Call the `postService` and `users` (repository)
as class properties.

> Run `php app.php configure` if your IDE does not highlight repositories or other services.

```php
// app/src/Command/Seed/PostCommand.php
namespace App\Command\Seed;

use Faker\Generator;
use Spiral\Console\Command;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostCommand extends Command
{
    use PrototypeTrait;

    protected const NAME = 'seed:post';

    protected function perform(Generator $faker): void
    {
        $users = $this->users->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];

            $post = $this->postService->createPost(
                $user,
                $faker->sentence(12),
                $faker->text(900)
            );

            $this->sprintf("New post: <info>%s</info>\n", $post->title);
        }
    }
}
```

Run the command with `-vv` flag to observe the SQL queries:

```bash
$ php app.php seed:post -vv
```

To remove prototype properties run:

```bash
$ php app.php prototype:inject -r
```

You command will be converted into the following form:

```php
namespace App\Command\Seed;

use App\Repository\UserRepository;
use App\Service\PostService;
use Faker\Generator;
use Spiral\Console\Command;

class PostCommand extends Command
{
    protected const NAME = 'seed:post';
    
    /** @var UserRepository */
    private $users;

    /** @var PostService */
    private $postService;

    /**
     * @param UserRepository $users2
     * @param PostService    $postService
     * @param string|null    $name
     */
    public function __construct(UserRepository $users2, PostService $postService, ?string $name = null)
    {
        parent::__construct($name);
        $this->postService = $postService;
        $this->users = $users2;
    }

    protected function perform(Generator $faker): void
    {
        $users = $this->users->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];

            $post = $this->postService->createPost(
                $user,
                $faker->sentence(12),
                $faker->text(900)
            );

            $this->sprintf("New post: <info>%s</info>\n", $post->title);
        }
    }
}
```

> You can use the prototype in any part of your codebase. Do not forget to remove the extension before going live. 

### CommentCommand
Seed comments using random user and post relation. We will receive all the needed instances using the method injection.

```php
namespace App\Command\Seed;

use App\Database\Comment;
use App\Repository\PostRepository;
use App\Repository\UserRepository;
use Cycle\ORM\TransactionInterface;
use Faker\Generator;
use Spiral\Console\Command;

class CommentCommand extends Command
{
    protected const NAME = 'seed:comment';

    protected function perform(
        Generator $faker,
        TransactionInterface $tr,
        UserRepository $userRepository,
        PostRepository $postRepository
    ): void {
        $users = $userRepository->findAll();
        $posts = $postRepository->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];
            $post = $posts[array_rand($posts)];

            $comment = new Comment();
            $comment->author = $user;
            $comment->post = $post;
            $comment->message = $faker->sentence(12);

            $tr->persist($comment);
            $tr->run();
        }
    }
}
```

Run the command:

```bash
$ php app.php seed:comment -vv
```

## Controller
Create a set of REST endpoints to retrieve the post data via API. We can start with a simple controller, `App\Controller\PostController`.
Create it using scaffolder:

```bash
$ php .\app.php create:controller post -a test -a get -p 
```

> Use option `-a` to pre-generate controller actions and option `-p` to pre-load prototype extension.

The generated code:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    public function test()
    {
    }

    public function get()
    {
    }
}
```

### Test Method
You can return various types of data from your controller methods. Following return values are valid:
- string
- PSR-7 response
- array (as JSON)
- JsonSerializable object

> Use custom [domain core](/cookbook/domain-core.md) to perform domain-specific response transformations. You can also
> use the `$this->response` helper to write the data into PSR-7 response object.

For demo purposes return `array`, the `status` key will be treated as response status.


```php
/**
 * @Route(action="/api/test/<id>", verbs={"GET"})
 * @param string $id
 * @return array
 */
public function test(string $id)
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

// ...

/**
 * @Route(action="/api/test/<id>", verbs={"GET"})
 * @param string $id
 * @return ResponseInterface
 */
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

> We won't use the test method going forward.

### Get Post
To get post details use `PostRepository`, request such dependency in the constructor, `get` method, or use prototype shortcut 
`posts`. You can access `id` via route parameter:

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<id:\d+>", verbs={"GET"})
     * @param string $id
     * @return array
     */
    public function get(string $id)
    {
        /** @var Post $post */
        $post = $this->posts->findByPK($id);
        if ($post === null) {
            throw new NotFoundException("post not found");
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

You can replace direct repository access and use `Post` as method injection via connected `CycleInterceptor` (make sure that `AppBootloader` connected):

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<post:\d+>", verbs={"GET"})
     * @param Post $post
     * @return array
     */
    public function get(Post $post)
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

> Consider using view object to map the response data into the `JsonSerializable` form.

### Post View Mapper
You can use any existing serialization solution (like `jms/serializer`) or write your own. Create a prototyped view object
to map post data into JSON format with comments:

```php
namespace App\View;

use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Core\Container\SingletonInterface;
use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;

/**
 * @Prototyped(property="postView")
 */
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

> Run `php app.php configure` to generate the IDE highlight and register prototyped class.

Modify the controller as follows:

```php
namespace App\Controller;

use App\Annotation\Route;
use App\Database\Post;
use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class PostController
{
    use PrototypeTrait;

    /**
     * @Route(action="/api/post/<post:\d+>", verbs={"GET"})
     * @param Post $post
     * @return ResponseInterface
     */
    public function get(Post $post): ResponseInterface
    {
        return $this->postView->json($post);
    }
}
```

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
/**
     * @Route(action="/api/post", verbs={"GET"})
     * @return array
     */
    public function list(): array
    {
        $posts = $this->posts->findAllWithAuthor();

        return [
            'posts' => array_map([$this->postView, 'map'], $posts->fetchAll())
        ];
    }
```

> You can see the JSON of all the posts using `http://localhost:8080/api/post`.

### Data Grid
An approach provided above has its limitations since you have to paginate, filter, and order the result manually. 
Use the [Data Grid component](/component/data-grid.md) to handle data formatting for you:

```bash
$ composer require spiral/data-grid-bridge
```

Activate the `Spiral\DataGrid\Bootloader\GridBootloader` in your application.

To use data grids, we have to specify our data schema first, create `App\View\PostGrid` class:

```php
namespace App\View;

use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\IntValue;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="postGrid")
 */
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

> We have added one filter and two sorting options to the grid. The pagination is done using page limits.

Connect the bootloader to your method using `Spiral\DataGrid\GridFactory`:

```php
/**
 * @Route(action="/api/post", verbs={"GET"})
 * @param GridFactory $grids
 * @return array
 */
public function list(GridFactory $grids): array
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return [
        'posts' => array_map(
            [$this->postView, 'map'],
            iterator_to_array($grid->getIterator())
        )
    ];
}
```

> Do not forget to run `php app.php configure` after adding prototyped class.

The grids are very flexible component with many customization options. By the default, grid configured to read
values from request query and data.
 
URL | Comment
--- | ---
`http://localhost:8080/api/post?paginate[page]=2` | Open second page.
`http://localhost:8080/api/post?paginate[page]=2&paginate[limit]=20` | Open second page with 20 posts per page.
`http://localhost:8080/api/post?sort[id]=desc` | Sort by post->id DESC.
`http://localhost:8080/api/post?sort[author]=asc` | Sort by post->author->id.
`http://localhost:8080/api/post?filter[author]=1` | Find only posts with given author id.

You can use sorters, filters, and pagination in one request. Multiple filters can activate at once.

## Validate Request
Make sure to [validate](/security/validation.md) data from the client. Use low level validation interfaces or 
[Request Filters](/filters/configuration.md) to validate, filter and map your data.

Create `CommentFilter` using the [scaffolder extension](/basics/scaffolding.md):

```bash
$ php app.php create:filter comment
```

Configure filter as following:

```php
namespace App\Filter;

use Spiral\Filters\Filter;

class CommentFilter extends Filter
{
    protected const SCHEMA = [
        'message' => 'data:message'
    ];

    protected const VALIDATES = [
        'message' => ['notEmpty']
    ];

    protected const SETTERS = [
        'message' => 'strval'
    ];

    public function getMessage(): string
    {
        return $this->message;
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
use Cycle\ORM\TransactionInterface;
use Spiral\Prototype\Annotation\Prototyped;

/**
 * @Prototyped(property="commentService")
 */
class CommentService
{
    private $tr;

    public function __construct(TransactionInterface $tr)
    {
        $this->tr = $tr;
    }

    public function comment(Post $post, User $user, string $message): Comment
    {
        $comment = new Comment();
        $comment->post = $post;
        $comment->author = $user;
        $comment->message = $message;

        $this->tr->persist($comment);
        $this->tr->run();

        return $comment;
    }
}
```

### Controller Action
Declare controller method and request filter instance. Since you use `FilterInterceptor` in your domain-core, the framework
will guarantee that filter is valid. Create `comment` endpoint to post message to a given post:

```php
/**
 * @Route(action="/api/post/<post:\d+>/comment", verbs={"POST"})
 * @param Post          $post
 * @param CommentFilter $commentFilter
 * @return array
 */
public function comment(Post $post, CommentFilter $commentFilter)
{
    $this->commentService->comment(
        $post,
        $this->users->findOne(), // todo: use current user
        $commentFilter->getMessage()
    );

    return ['status' => 201];
}
```

> Use `spiral/toolkit` Stempler extension to create AJAX-native forms in HTML. 

### Execute
Check the error format:

```bash
$ curl -X POST -H 'content-type: application/json' --data '{}' http://localhost:8080/api/post/1/comment
``` 

Response:

```json
{"status":400,"errors":{"message":"This value is required."}}
```

Or not found exception when post can not be found:

```bash
$ curl -X POST -H 'content-type: application/json' --data '{}' http://localhost:8080/api/post/9999/comment
``` 

> Make sure to send `accept: application/json` to receive an error in JSON format.

To post a valid comment: 

```bash
$ curl -X POST -H 'content-type: application/json' --data '{"message": "first comment"}' http://localhost:8080/api/post/1/comment
```

> Read more about filters [here](/filters/filter.md). Change the [scaffolder](/basics/scaffolding.md) configuration to 
> alter the generated request postfix or default namespace.

## Render Template
To render post information into HTML form use [views](/views/configuration.md) and [Stempler](/stempler/configuration.md) component. 
Pass post list to the view using Grid object.

```php
/**
 * @Route(action="/posts", verbs={"GET"})
 * @param GridFactory $grids
 * @return string
 */
public function all(GridFactory $grids): string
{
    $grid = $grids->create($this->posts->findAllWithAuthor(), $this->postGrid);

    return $this->views->render('posts', ['posts' => $grid]);
}
```

### Create Layout
Create/edit layout file located in `app/views/layout/base.dark.php`:

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

> You can now see the list of posts on `http://localhost:8080/posts`, use URL Query parameters to control Data Grid filters,
> sorters (`http://localhost:8080/posts?paginate[page]=2`).   

### View Post
To view post and all of its comments, create a new controller method in `PostController`. Load post manually via repository
to preload all author and comment information.

```php
use Spiral\Http\Exception\ClientException\NotFoundException;
// ...

/**
 * @Route(action="/post/<id:\d+>", verbs={"GET"})
 * @param string $id
 * @return string
 */
public function view(string $id): string
{
    $post = $this->posts->findOneWithComments($id);
    if ($post === null) {
        throw new NotFoundException();
    }

    return $this->views->render('post', ['post' => $post]);
}
```

Where the repository method is:

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

> We are leaving styling and comment timestamps up to you.

### Route
We used pattern `ControllerName.methodName` in `RoutesBootloader` to automatically name routes. Use `PostController.view`
route name to generate link in `app/views/posts.dark.php`:

```html
<extends:layout.base title="Posts"/>
       
<define:body>
   @foreach($posts as $post)
       <div class="post">
           <div class="title">
               <a href="@route('PostController.view', ['id' => $post->id])">{{$post->title}}</a>
           </div>
           <div class="author">{{$post->author->name}}</div>
       </div>
   @endforeach
</define:body>
```

> Read more about Stempler Directives [here](/stempler/directives.md).

## Next Steps
Spiral provides a lot of pre-build functionality for you. Read the following sections to gain more insigns:
- [Authenticating users](/security/authentication.md)
- [Authorize Access](/security/rbac.md)
- [Background Jobs](/queue/configuration.md)

## Source Code
Source code of demo project - https://github.com/spiral/demo 

Make sure to run to install the project:

```bash
$ vendor/bin/spiral get
$ php app.php configure
```
