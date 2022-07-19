# Long Start
The Spiral Framework contains a lot of components built to operate seamlessly with each other.
In this article, we will show how to create a demo blog application with REST API, ORM, Migrations, request validation, custom attributes (optional) and domain interceptors.

> **Note**
> The components and approaches will be covered at basic levels only. Read the corresponding sections to gain more information. You can find the demo
> repository [here](https://github.com/spiral/demo).

## Installation
Use composer to install the default `spiral/app` bundle with most of the components out of the box:

```bash
composer create-project spiral/app spiral-demo
cd spiral-demo
```

> **Note**
> Use the different build `spiral/app-cli` to install the spiral with minimal dependencies.

If everything installed correctly, you could open your application immediately by starting the server:

```bash
./rr serve
```

You just started an [application server](/framework/application-server.md). The same server can be used on production, making your
development environment similar to the final setup. Out of the box, the server includes instruments to write portable applications
with HTTP/2, GRPC, Queue, WebSockets, etc. and does not require external brokers to operate.

By default, the application available on `http://localhost:8080`. The build includes multiple pre-defined pages you can play with.

> **Note**
> Check the exception page `http://localhost:8080/exception.html`, at the right part of this page you can see all interceptors
> and middleware included in the default build. We will turn some of them off to make the application runtime smaller. 

## Configure
Spiral applications configured using config files located in `app/config`, you can use the hardcoded values for the configuration, 
or get the values using available functions `env` and `directory`. The `spiral/app` bundle use DotEnv extension which 
will load ENV variables from the `.env` file.

> **Note**
> Tweak the application server and its plugins using `.rr.yaml` file.

The application dependencies defined in `composer.json` and activated in `app/src/App.php` as Bootloaders.
The default build includes quite a lot of pre-configured components.

### Developer Mode
To simplify the tweaking of the application, restart the application server in developer mode. In this mode, the server uses
only one worker and reloads it after every request.

```bash
$ ./rr serve -o "http.pool.max_jobs=1" -o "http.pool.num_workers=1" -o "http.pool.debug=true" -o "http.address=127.0.0.1:8181"
```

You can also create and use an alternative configuration file via `-c` flag of the `rr` application.

> **Note**
> Read more about Workers and Lifecycle [here](/start/workers.md).

### Lighter Up
We won't need translation, session, cookies, CSRF, and encryption in our demo application. Remove these components and their bootloaders.

Delete following bootloaders from `app/src/App.php`:

```php
Spiral\Bootloader\I18nBootloader::class,
Spiral\Bootloader\Security\EncrypterBootloader::class,

// from http
Spiral\Bootloader\Http\CookiesBootloader::class,
Spiral\Bootloader\Http\SessionBootloader::class,
Spiral\Bootloader\Http\CsrfBootloader::class,
Spiral\Bootloader\Http\PaginationBootloader::class,

// from views 
Spiral\Bootloader\Views\TranslatedCacheBootloader::class,

// from APP
App\Bootloader\LocaleSelectorBootloader::class,
```

Delete following files and directories as no longer required:
- `app/locale`
- `app/src/Bootloader/LocaleSelectorBootloader.php`
- `app/src/Middleware`.

> **Note**
> Note, the application won't work at the moment as we removed the dependency required to render `app/views/home.dark.php`.

### Database Connection
Our application needs a database to operate. By default, the database configuration located in `app/config/database.php` file.
The `spiral/app` application comes with pre-configured In-Memory SQLite database.

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

We can store the database name, username, password and port in `.env` file, add the following lines into it:

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

To check that the database connection was successful run:

```bash
php app.php db:list
```

> **Note**
> Read more about Databases [here](/database/configuration.md).

### Connect Faker
We will need some sample data for the application. Let's connect the library and bootload the library `fakerphp/faker`.

```bash
composer require fakerphp/faker
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

         Bootloader\AppBootloader::class,
+        Bootloader\FakerBootloader::class,
     ];
 }
```

> **Note**
> You can request dependencies as method arguments in the factory method `fakerGenerator`.

Use the `Faker\Generator` in your controller to view the stub data at `http://localhost:8080/`:

```php
namespace App\Controller;

use Faker\Generator;

class HomeController
{
    public function index(Generator $generator): string
    {
        return $generator->sentence(128);
    }
}
```

> **Note**
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

> **Note**
> Read more about Routing [here](/http/routing.md).

### Creating Routes as Attributes
In order to simplify the route definition we can use `spiral/annotated-routes` extension. Read more about the extension [here](/http/annotated-routes.md).
We can use this annotation in our controller as follows:

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

In the following examples, we will stick to the annotated routes for simplicity.

To flush route cache (when DEBUG disabled):

```bash
php app.php route:reset
```

### Domain Core
Connect custom controller interceptor (domain-core) to enrich your domain layer with additional functionality.
We can change the default behavior of the application and enable Cycle Entity resolution using route parameter, 
Filter validation and `Guard` attribute.

```php
namespace App\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;
use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Domain;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        Domain\GuardInterceptor::class,
        Domain\FilterInterceptor::class,
    ];
}
```

Enable the domain core in your application. We will demonstrate the use of the interceptor below.

> **Note**  
> Read more about Domain Cores [here](/cookbook/domain-core.md).

## Scaffold Database
The framework can configure the database schema using a set of migration files. To configure migrations in your application
run:

```bash
php app.php migrate:init
```

You can now observe the migration table structure using:

```bash
php app.php db:list
php app.php db:table migrations
```

You can write the migration manually, or let Cycle ORM generate it for you.

> **Note**  
> Read more about migrations [here](https://cycle-orm.dev/docs/database-migrations). Use [Scaffolder](/basics/scaffolding.md) component to create migrations manually. 

### Define ORM Entities
The demo application comes with [Cycle ORM](https://cycle-orm.dev). By default, you can use attributes to configure
your entities.

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

use Cycle\Annotated\Annotation as Cycle;

/**
 * @Cycle\Entity(repository="\App\Repository\PostRepository")
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

Scaffolder before `Spiral Framework v3.0` doesn't support attributes and property types. 
After the file is created, it's recommended to replace annotations with attributes and add types:

```php
namespace App\Database;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: PostRepository::class)]
class Post
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: 'string')]
    public string $title;

    #[Cycle\Column(type: 'text')]
    public string $content;
}

```

User, with added attributes and types:

```php
namespace App\Database;

use App\Repository\UserRepository;
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)] 
class User
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: 'string')]
    public string $name;
}
```

Comment, with added attributes and types:

```php
namespace App\Database;

use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity]
class Comment
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: 'string')]
    public string $message;
}
```

You can change the default directory mapping, headers, and others using [Scaffolder config](/basics/scaffolding.md).

> **Note**
> Read more about Cycle [here](/cycle/configuration.md). Configure auto-timestamps using [custom mapper](https://cycle-orm.dev/docs/mapper-extending).

### Generate Migration
To generate the database schema run:

```bash
php app.php cycle:migrate -v
```

The generated migration located in `app/migrations/`. Execute it using:

```bash
php app.php migrate -vv
```

You can now observe the generated tables using `db:list` command.

### Create Relations
Use [annotations](https://cycle-orm.dev/docs/annotated-relations) to define the relations between entities. Configure
Post and Comment to belong to User and Post has many Comments. 

Post:

```php
namespace App\Database;

use App\Repository\PostRepository;
use Cycle\Annotated\Annotation as Cycle;
use Doctrine\Common\Collections\ArrayCollection;

#[Cycle\Entity(repository: PostRepository::class)] 
class Post
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: 'string')]
    public string $title;

    #[Cycle\Column(type: 'text')]
    public string $content;

    #[Cycle\Relation\BelongsTo(target: User::class, nullable: false)]
    public User $author;

    /**
     * @var Collection|Comment[]
     * @psalm-var Collection<int, Comment>
     */
    #[Cycle\Relation\HasMany(target: Comment::class)]
    public Collection $comments;

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

#[Cycle\Entity]
class Comment
{
    #[Cycle\Column(type: 'primary')]
    public int $id;

    #[Cycle\Column(type: 'string')]
    public string $message;
    
    #[Cycle\Relation\BelongsTo(target: User::class, nullable: false)]
    public User $author;

    #[Cycle\Relation\BelongsTo(target: Post::class, nullable: false)]
    public Post $post;
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

## Service
Isolate the business logic into a separate service layer. Let's create `PostService` in `app/src/Service`.
We will need an instance of `Cycle\ORM\EntityManagerInterface` to persist the post.

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\EntityManagerInterface;

class PostService
{
    public function __construct(
        private EntityManagerInterface $entityManager
    ) {
    }

    public function createPost(User $user, string $title, string $content): Post
    {
        $post = new Post();
        $post->author = $user;
        $post->title = $title;
        $post->content = $content;

        $this->entityManager->persist($post);
        $this->entityManager->run();

        return $post;
    }
}
```

> **Note**
> You can reuse the transaction after the `run` method.

### Prototyping
One of the most powerful capabilities of the framework is [Prototyping](/basics/prototype.md). Declare the shortcut `postService`, which points to `PostService` using annotation.

```php
namespace App\Service;

use App\Database\Post;
use App\Database\User;
use Cycle\ORM\TransactionInterface;
use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'postService')]
class PostService
{
    // ...
}
``` 

Run the configure command to collect all available prototype classes:

```bash
php app.php configure
```

> **Note**
> Make sure to use proper IDE to gain access to the IDE tooltips.

Now you can get access to the `PostService` using `PrototypeTrait`, see the example down below.

## Console Command
Let's create three commands to generate the data for our application. Use scaffolder extension to create command to seed our database:

```bash
php app.php create:command seed/user seed:user
php app.php create:command seed/post seed:post
php app.php create:command seed/comment seed:comment
``` 

Generated commands will be available in `app/src/Command/Seed`.

### UserCommand
Use the method injection on `perform` in `UserCommand` to seed the users using Faker:

```php
// app/src/Command/Seed/UserCommand.php
namespace App\Command\Seed;

use App\Database\User;
use Cycle\ORM\EntityManagerInterface;
use Faker\Generator;
use Spiral\Console\Command;

class UserCommand extends Command
{
    protected const NAME = 'seed:user';

    protected function perform(EntityManagerInterface $em, Generator $faker): int
    {
        for ($i = 0; $i < 100; $i++) {
            $user = new User();
            $user->name = $faker->name;

            $em->persist($user);
        }

        $em->run();

        $this->output->write('<info>Database seeding completed successfully.</info>');

        return self::SUCCESS;
    }
}
```

Run the command:

```bash
php app.php seed:user
```

### PostCommand
Use the prototype extension to speed up the creation of the `seed:post` command. Call the `postService` and `users` (repository)
as class properties.

> **Note**
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

    protected function perform(Generator $faker): int
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
        
        return self::SUCCESS;
    }
}
```

Run the command with `-vv` flag to observe the SQL queries:

```bash
php app.php seed:post -vv
```

To remove prototype properties run:

```bash
php app.php prototype:inject -r
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

    public function __construct(
        private UserRepository $users,
        private PostService $postService,
        ?string $name = null
    ) {
        parent::__construct($name);
    }

    protected function perform(Generator $faker): int
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

        return self::SUCCESS;
    }
}
```

> **Note**
> You can use the prototype in any part of your codebase. Do not forget to remove the extension before going live. 

### CommentCommand
Seed comments using random user and post relation. We will receive all the needed instances using the method injection.

```php
namespace App\Command\Seed;

use App\Database\Comment;
use App\Repository\PostRepository;
use App\Repository\UserRepository;
use Cycle\ORM\EntityManagerInterface;
use Faker\Generator;
use Spiral\Console\Command;

class CommentCommand extends Command
{
    protected const NAME = 'seed:comment';

    protected function perform(
        Generator $faker,
        EntityManagerInterface $entityManager,
        UserRepository $userRepository,
        PostRepository $postRepository
    ): int {
        $users = $userRepository->findAll();
        $posts = $postRepository->findAll();

        for ($i = 0; $i < 1000; $i++) {
            $user = $users[array_rand($users)];
            $post = $posts[array_rand($posts)];

            $comment = new Comment();
            $comment->author = $user;
            $comment->post = $post;
            $comment->message = $faker->sentence(12);

            $this->sprintf("New comment: <info>%s</info>\n", $comment->message);

            $entityManager->persist($comment);
            $entityManager->run();
        }

        return self::SUCCESS;
    }
}
```

Run the command:

```bash
php app.php seed:comment -vv
```

## Controller
Create a set of REST endpoints to retrieve the post data via API. We can start with a simple controller, `App\Controller\PostController`.
Create it using scaffolder:

```bash
php app.php create:controller post -a test -a get -p 
```

> **Note**
> Use option `-a` to pre-generate controller actions and option `-p` to pre-load prototype extension.

The generated code:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

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

> **Note**
> Use custom [domain core](/cookbook/domain-core.md) to perform domain-specific response transformations. You can also
> use the `$this->response` helper to write the data into PSR-7 response object.

For demo purposes return `array`, the `status` key will be treated as response status.


```php
// ...
#[Route(route: '/api/test/<id>', name="post.test", methods: 'GET')]
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
To get post details use `PostRepository`, request such dependency in the constructor, `get` method, or use prototype shortcut 
`posts`. You can access `id` via route parameter:

```php
namespace App\Controller;

use App\Database\Post;
use Spiral\Http\Exception\ClientException\NotFoundException;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class PostController
{
    use PrototypeTrait;

    #[Route(route: '/api/post/<post:\d+>', name: 'post.get', methods: 'GET')]
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

You can replace direct repository access and use `Post` as method injection via connected `CycleInterceptor` (make sure that `AppBootloader` connected):

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
You can use any existing serialization solution (like `jms/serializer`) or write your own. Create a prototyped view object
to map post data into JSON format with comments:

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
> Run `php app.php configure` to generate the IDE highlight and register prototyped class.

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
An approach provided above has its limitations since you have to paginate, filter, and order the result manually. 
Use the [Data Grid component](/component/data-grid.md) to handle data formatting for you:

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
        'posts' => array_map(
            [$this->postView, 'map'],
            iterator_to_array($grid->getIterator())
        )
    ];
}
```

> **Note**
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
php app.php create:filter comment
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
        $comment = new Comment();
        $comment->post = $post;
        $comment->author = $user;
        $comment->message = $message;

        $this->entityManager->persist($comment);
        $this->entityManager->run();

        return $comment;
    }
}
```

### Controller Action
Declare controller method and request filter instance. Since you use `FilterInterceptor` in your domain-core, the framework
will guarantee that filter is valid. Create `comment` endpoint to post message to a given post:

```php
#[Route(route: '/api/post/<post:\d+>/comment', name: 'post.comment', methods: 'POST')] 
public function comment(Post $post, CommentFilter $commentFilter): array
{
    $this->commentService->comment(
        $post,
        $this->users->findOne(), // todo: use current user
        $commentFilter->getMessage()
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
{"status":400,"errors":{"message":"This value is required."}}
```

Or not found exception when post can not be found:

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
> Read more about filters [here](/filters/filter.md). Change the [scaffolder](/basics/scaffolding.md) configuration to 
> alter the generated request postfix or default namespace.

## Render Template
To render post information into HTML form use [views](/views/configuration.md) and [Stempler](/stempler/configuration.md) component. 
Pass post list to the view using Grid object.

```php
#[Route(route: '/posts', name: 'post.all', methods: 'GET')] 
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

> **Note**
> You can now see the list of posts on `http://localhost:8080/posts`, use URL Query parameters to control Data Grid filters,
> sorters (`http://localhost:8080/posts?paginate[page]=2`).   

### View Post
To view post and all of its comments, create a new controller method in `PostController`. Load post manually via repository
to preload all author and comment information.

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

> **Note**
> We are leaving styling and comment timestamps up to you.

### Route
 Use `post.view` route name to generate link in `app/views/posts.dark.php`:

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
Spiral provides a lot of pre-build functionality for you. Read the following sections to gain more insigns:
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
