# Quick Start [Work in Progress]
Spiral framework contains a lot of components build to operate seamlessly with each other.
In this article we will show how to create a demo blog application with REST API, ORM, Migrations,
request validation, queue and user authorization.

> The components and approaches will be covered at basic levels only. Read the corresponding 
sections to gain more information. 

## Installation
Use composer to install the default `spiral/app` bundle with most of components out of the box:

```bash
$ composer create-project spiral/app spiral-demo
$ cd spiral-demo
```

> Use the opposite build `spiral/app-cli` to install spiral with minimal dependencies.

If everything installed correctly you can open your application immediately by starting the server:

```bash
$ ./spiral serve -v -d
```

You just started an [application server](/framework/application-server.md). The same server can used on production, making your
development environment similar to the final setup. Out of the box server includes instruments to write portable applications
with HTTP/2, GRPC, Queue, WebSockets, etc and does not require external brokers to operate.

By default, the application available on `http://localhost:8080`. The build includes multiple pre-defined pages you can play with.

> Check the exception page `http://localhost:8080/exception.html`, at the right part of this page you can see all interceptors
> and middleware included in the default build. We will turn some of them off to make the application runtime smaller. 

## Configure
Spiral applications are configured using config files located in `app/config`, you can use the hardcoded values for the configuration, 
or get the values using available functions `env` and `directory`. The `spiral/app` bundle use DotEnv extension which 
will load ENV variables from the `.env` file.

> Tweak the application server and its plugins using `.rr.yaml` file.

The list of application dependencies located in `composer.json` and activated in a form of Bootloaders in `app/src/App.php`.
The default build includes quite a lot of pre-configured components.

### Developer Mode
To simplify the tweaking of the application, restart the application server in developer mode. In this mode the server uses
only one worker and reloads it after every request, it emulates the PHP-FPM flow.

```bash
$ .\spiral.exe serve -v -d -o "http.workers.pool.maxJobs=1" -o "http.workers.pool.numWorkers=1"
```

You can also create and use an alternative configuration file via `-c` flag of `spiral` application.

> Read more about Workers and Lifecycle [here](/start/workers.md).

### Lighter Up
We won't need translation, session, cookies, CSRF and encryption in our demo application. Remove these components and their 
bootloaders.

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
Our application need a database to operation. By the default, the database configuration is located in `app/config/database.php` file.
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
            'connection' => 'sqlite:' . directory('runtime') . 'runtime.db',
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

To check the the database connection was successful run:

```bash
$ php app.php db:list
```

> Read more about Databases [here](/database/configuration.md).

### Connect Faker
We will need some sample data for the application. Let's connect the library and bootload the library `fzaninotto/faker`.

```bash
$ composer require fzaninotto/faker
``` 

To generate the stub data we will need an instance of `Faker\Generator`, create bootloader in `app/src/Bootloader` to resolve such
instance as singleton. Use factory method for that purposes.

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

Add the bootloader to `LOAD` or `APP` in `app/src/App.php` to activate the component.

> You can request dependencies as method arguments in the factory method `faker`.

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
By the default, the routing rules located in `app/src/Bootloader/RoutingBootloader.php`. You have many options how
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

In the given setup the action and id are the optional parts of the URL (see the `[]`), the action defaults to `index`.
Open `HomeController`->`index` using `http://localhost:8080/` or `http://localhost:8080/index`.

The route parameters can be addressed in method injection of the controller by their name, create the following 
method in `HomeController`:

```php
public function open(string $id)
{
    dump($id);
}
```

You can invoke this method using URL `http://localhost:8080/open/123`, the `id` parameter will be hydrated automatically.

> Read more about Routing [here](/http/routing.md).

### Annotated Routing
Though the framework does not provide the annotated routing support out of the box yet, it is possible to [configure it](/cookbook/annotated-routes.md)
manually using existing instruments. 

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

In the following examples we will stick to the annotated routes for the simplicity.

### Domain Core
Connect custom controller interceptor (domain-core) to enrich your domain layer with additional functionality.
We can change the default behaviour of the application and enable Cycle Entity resolution using route parameter, 
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
The demo application comes with [Cycle ORM](https://cycle-orm.dev). By the default you can use annotations to configure
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
 * @Cycle\Entity(repository = "post")
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
 * @Cycle\Entity(repository = "user")
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

You can change the default directory mapping, headers and others using [Scaffolder config](/basics/scaffolding.md).

> Read more about Cycle [here](/cycle/configuration.md). Configure auto-timestamps using [custom mapper](https://cycle-orm.dev/docs/advanced-timestamp).

### Generate Migration
To generate the database schema run:

```bash
$ php app.php cycle:migrate -v
```

The generated migration can be found in `app/migrations/`. Execute it using:

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
 * @Cycle\Entity(repository = "post")
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
     * @Cycle\Relation\BelongsTo(target = "User", nullable = false, cascade = false)
     * @var User
     */
    public $author;
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

> Do not forget to run `php app.php cycle:migrate` when you change any of your entity.

## Service

### Prototyping

### Persist Entity

## Console Command

## Controller

### Data Grid

## Validate Request

### Hydrate Entity

### Combine 

## Render Template

### Create Layout

### Implement View

## Background Task

### Create Endpoint

### Create Job Handler

### Push Background Task

## Authenticate User

### Connect the Component

### Authenticate User

### Attach Middleware
