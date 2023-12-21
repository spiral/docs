# Real-time chat application

Real-time chat applications have become increasingly popular in recent years, and implementing WebSocket servers to
enable bidirectional communication has become essential for building such applications. However, creating this kind of
can be a challenging task.

Fortunately, there are new frameworks and tools available that make it easier to set up WebSocket servers. In this
tutorial, we will demonstrate how to create a real-time chat application using the Spiral Framework, RoadRunner, and
Centrifugo with authentication and bidirectional communication.

Spiral Framework offers an array of seamlessly integrated components, which makes it an ideal choice for building
complex applications. In this tutorial, we will guide you on creating a simple real-time chat application using
Spiral Framework, Centrifugo, RoadRunner, and ORM.

> **Note**
> This tutorial covers the basics of the components and approaches used in creating the chat application. For more
> detailed information, we suggest referring to the relevant sections. Additionally, a demo repository is available
> at [here](https://github.com/spiral/simple-chat).

## Installation

### Spiral application

To get started with building your real-time chat application, you can easily install the default `spiral/app` bundle
with most of the required components by running the following command:

```terminal
composer create-project spiral/app realtime-chat
```

During the installation process, you will be prompted to select various options with the Spiral installer, such as the
application preset, whether to use Cycle ORM, which collections to use, which validator component to use, and so on. For
this tutorial, we recommend choosing the options shown above:

```terminal
✔ Which application preset do you want to install? > Web
✔ Create a default application structure and demo data? > No
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
cd realtime-chat

./rr serve
```

Your application will be available on http://127.0.0.1:8080 by default.

### Centrifugo

Centrifugo is a powerful real-time messaging server. To install it, we have prepared a simple bash script that downloads
the latest version of the Centrifugo binary and installs it in the `bin` directory of your application. You can run the
script using the following command:

```bash
wget --timeout=10 https://github.com/centrifugal/centrifugo/releases/download/v4.1.2/centrifugo_4.1.2_linux_amd64.tar.gz
mkdir -p bin
tar xvfz centrifugo_4.1.2_linux_amd64.tar.gz centrifugo 
rm -rf centrifugo_4.1.2_linux_amd64.tar.gz
mv centrifugo bin/
chmod +x ./bin/centrifugo
```

After installing Centrifugo, the next step is to create a `centrifugo.json` configuration file at the root directory of
your project, which will contain the necessary configuration details:

```json centrifugo.json
{
  "allowed_origins": [
    "*"
  ],
  "proxy_connect": true,
  "address": "127.0.0.1",
  "port": 8081,
  "grpc_api": true,
  "grpc_api_address": "127.0.0.1",
  "grpc_api_port": 10000,
  "proxy_connect_endpoint": "grpc://127.0.0.1:10001",
  "proxy_connect_timeout": "10s",
  "proxy_rpc_endpoint": "grpc://127.0.0.1:10001",
  "proxy_rpc_timeout": "10s"
}
```

To enable communication between Centrifugo and RoadRunner, you need to configure RoadRunner in the `.rr.yaml` file by
specifying the details of the gRPC server and its connection to Centrifugo. This allows RoadRunner to handle Centrifugo
events and send them to your application, and vice versa.

```yaml .rr.yaml
#...

service:
  # Create a new service that will run Centrifugo server
  cetrifugo:
    service_name_in_log: true
    remain_after_exit: true
    restart_sec: 1
    command: "./bin/centrifugo --config=centrifugo.json"

centrifuge:
  proxy_address: tcp://127.0.0.1:10001
  grpc_api_address: tcp://127.0.0.1:10000
  pool:
    reset_timeout: 10
    num_workers: 5
```

With these configurations in place, your application will be able to send events to Centrifugo, and Centrifugo will be
able to send events to your application, enabling bidirectional communication between them.

## Configuration

The configuration of Spiral applications is accomplished through configuration files located in the `app/config`
directory. You can use the predefined values in these files, or you can obtain the values programmatically using the
`env` and `directory` functions.

The application dependencies are defined in the `composer.json` file, and they are activated as bootloaders in the
`app/src/Application/Kernel.php` file.

### Bootloaders

To optimize our application and make it more lightweight, we need to add the required bootloaders and remove some of the
default ones. Let's make some changes in the `app/src/Application/Kernel.php` file:

```diff app/src/Application/Kernel.php
// ... 

// RoadRunner
RoadRunnerBridge\LoggerBootloader::class,
RoadRunnerBridge\HttpBootloader::class,
+ RoadRunnerBridge\CentrifugoBootloader::class,

// ... 

// Security and validation
Framework\Security\EncrypterBootloader::class,
Framework\Security\FiltersBootloader::class,
- Framework\Security\GuardBootloader::class,

// ...

Framework\Http\CsrfBootloader::class,
- Framework\Http\PaginationBootloader::class,

// ...

// ORM
CycleBridge\SchemaBootloader::class,
CycleBridge\CycleOrmBootloader::class,
CycleBridge\AnnotatedBootloader::class,
+ CycleBridge\AuthTokensBootloader::class,

// Views and view translation
ViewsBootloader::class,
- TranslatedCacheBootloader::class,

// ...

// Fast code prototyping
PrototypeBootloader::class,
+ CycleBridge\PrototypeBootloader::class,
```

> **Note**
> Read more about Bootloaders [here](../framework/bootloaders.md).

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

Since we won't be using the REST API in this example, let's remove the `api` middleware group and add the
`Spiral\Filter\ValidationHandlerMiddleware` for handling validation errors. In
the `app/src/Application/Bootloader/RoutesBootloader.php` file, update the `middlewareGroups` method as follows:

```diff app/src/Application/Bootloader/RoutesBootloader.php
class RoutesBootloader extends BaseRoutesBootloader
{
    ...
    
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                 ...
+                \Spiral\Filter\ValidationHandlerMiddleware::class,
+                \Spiral\Auth\Middleware\AuthMiddleware::class,
            ],
-           'api' => [
-                ...
-           ],
        ];
    }
 }
```

> **Note**
> As a reminder, when making changes to the code, it's important to remove any unused imports. These can clutter up the
> code and make it harder to read and maintain.

### Broadcasting

To enable broadcasting functionality in the application, it is necessary to configure the `broadcasting.php`
configuration file. This configuration will allow the application to transmit events to the Centrifugo server.

```php app/config/broadcasting.php
return [
    'connections' => [
        'centrifugo' => [
            'driver' => 'centrifugo',
        ],
    ],
];
```

Additionally, it is necessary to set the `BROADCAST_CONNECTION` environment variable to the newly created connection:

```dotenv .env
# Broadcast
BROADCAST_CONNECTION=centrifugo
```

Following these steps will enable the application to broadcast events to the Centrifugo server.

> **Note**
> The `centrifugo` driver is provided by the `spiral/roadrunner-bridge` package.

### Database Connection

In order for our application to function properly, a database connection must be established. The database configuration
file can be found in the `app/config/database.php` file. For this particular application, we will be utilizing a
PostgreSQL database.

```diff app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DRIVER'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

+    'default' => env('DB_CONNECTION', 'default'),
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    'drivers' => [
-        'runtime' => new Config\SQLiteDriverConfig(
-            connection: new Config\SQLite\FileConnectionConfig(
-                database: directory('runtime') . '/db.sqlite'
-            ),
-            queryCache: true
-        ),
+        'runtime' => new Config\PostgresDriverConfig(
+            connection: new Config\Postgres\TcpConnectionConfig(
+                database: env('DB_DATABASE', 'homestead'),
+                user: env('DB_USERNAME', 'homestead'),
+                password: env('DB_PASSWORD', 'secret'),
+                port: (int) env('DB_PORT', 5432),
+            ),
+            schema: 'public',
+            queryCache: true,
+            options: [
+                'withDatetimeMicroseconds' => true,
+                'logQueryParameters' => env('DB_LOG_QUERY_PARAMETERS', false),
+            ],
+        ),
    ],
];
```

We can store a database name, username, password and port in `.env` file, add the following lines into it:

```dotenv .env
DB_HOST=localhost
DB_NAME=homestead
DB_USER=homestead
DB_PASSWORD=secret
DB_PORT=5432
```

To verify that the database connection has been successfully established, the following command should be executed:

```terminal
php app.php db:list
```

> **See more**
> For further information regarding database configuration, please refer to
> the [database configuration](../database/configuration.md) documentation.

### Connect Database Seeder

We will need some sample data for the application. Let's install [Database Seeder](../testing/database.md).

```terminal
composer require spiral-packages/database-seeder --dev
``` 

To activate the package, the `Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader` bootloader must be added to
the `LOAD` section:

```diff app/src/Application/Kernel.php
         PrototypeBootloader::class,
+        \Spiral\DatabaseSeeder\Bootloader\DatabaseSeederBootloader::class,
```

Once these steps have been completed, the Database Seeder package will be enabled and can be utilized to provide sample
data for the application.

## Scaffold Database

The framework provides the ability to configure the database schema using a collection of migration files. To initiate
the migration configuration process within your application, execute the following command:

```terminal
php app.php migrate:init
```

After executing the previous command, you can observe the structure of the migration table by utilizing the following
commands:

```terminal
php app.php db:list
php app.php db:table migrations
```

Once the migration configuration has been initialized, you may manually write the migration files or allow Cycle ORM to
generate them for you.

### Define ORM Entities

Let's create `Thread`, `Message` and `User` entities and their repositories using 
the [scaffolder](../basics/scaffolding.md) component:

```terminal
php app.php create:entity thread -f id:primary -f name:string -e
php app.php create:entity message -f id:primary -f message:string -e
php app.php create:entity user -f id:primary -f username:string -f password:string -e
```

> **Note**
> Upon successful execution of the previous commands, the generated classes may be located within 
> the `app/src/Database` and `app/src/Repository` directories.

### Thread Entity

After the `Thread` entity has been created, it looks like this:

```php app/src/Database/Thread.php
namespace App\Database;

use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: '\App\Repository\ThreadRepository')]
class Thread
{
    #[Column(type: 'primary')]
    public int $id;

    #[Column(type: 'string')]
    public string $name;
}
```

Let's bring it in order:

```php app/src/Database/Thread.php
namespace App\Database;

use App\Repository\ThreadRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;

#[Entity(repository: ThreadRepository::class)]
class Thread implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[Column(type: "string")]
        private string $name,
    ) {
    }

    public function getName(): string
    {
        return $this->name;
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
        ];
    }
}
```

The generated `Message` and `User` entities and its repositories may be similarly modified to ensure that their 
properties and functionality aligns with the application requirements.

### Message Entity

```php app/src/Database/Message.php
namespace App\Database;

use App\Repository\MessageRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Relation\BelongsTo;

#[Entity(repository: MessageRepository::class)]
class Message implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[BelongsTo(target: Thread::class)]
        private Thread $thread,
        #[BelongsTo(target: User::class)]
        private User $user,
        #[Column(type: "text")]
        private string $text,
    ) {
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'user' => $this->user,
            'text' => $this->text,
        ];
    }
}
```

#### Message Repository

```php app/src/Repository/MessageRepository.php
namespace App\Repository;

use App\Database\Message;
use Cycle\ORM\Select\Repository;

final class MessageRepository extends Repository
{
    /**
     * @return Message[]
     */
    public function findAllByThread(int $threadId): array
    {
        return $this->findAll([
            'thread_id' => $threadId,
        ], [
            'id' => 'ASC',
        ]);
    }
}
```

### User Entity

```php app/src/Database/User.php
namespace App\Database;

use App\Repository\UserRepository;
use Cycle\Annotated\Annotation\Column;
use Cycle\Annotated\Annotation\Entity;
use Cycle\Annotated\Annotation\Table\Index;

#[Entity(repository: UserRepository::class)]
#[Index(columns: ['username'], unique: true)]
class User implements \JsonSerializable
{
    #[Column(type: 'primary')]
    private int $id;

    public function __construct(
        #[Column(type: "string")]
        private string $username,
        #[Column(type: "string")]
        private string $password,
    ) {
    }

    public function getId(): int
    {
        return $this->id;
    }

    public function getPassword(): string
    {
        return $this->password;
    }

    public function jsonSerialize(): array
    {
        return [
            'id' => $this->id,
            'username' => $this->username,
        ];
    }
}
```

#### User Repository

```php app/src/Repository/UserRepository.php
namespace App\Repository;

use App\Database\User;
use Cycle\ORM\Select\Repository;

final class UserRepository extends Repository
{
    public function findByUsername(string $username): ?User
    {
        return $this->findOne(['username' => $username]);
    }
}
```

> **Note**
> Read more about Cycle [here](../basics/orm.md).

Run the configure command to collect all available prototype classes:

```terminal
php app.php configure
```

### Generate Migration

To generate the database schema, use the following command:

```terminal
php app.php cycle:migrate -v
```

The generated migration can be found in the `app/migrations/` directory. You can execute the migration using the 
following command:

```terminal
php app.php migrate -vv
```

You can use the `db:list` command to observe the generated tables.

```terminal
php app.php db:list
```

## Factories and seeders

To generate test data, we need factories that describe the rules for generating an entity and seeders that fill the 
database. To maintain separation from the application code, these factories and seeders should be stored in a separate 
folder named `app/database`.

Let's add a separate `Database` namespace to Composer autoload, you can update the `composer.json` file as follows:

```diff composer.json
--- a/composer.json
+++ b/composer.json
"autoload-dev": {
    "psr-4": {
        "Tests\\": "tests",
+       "Database\\": "app/database"
    },
    //
},
```

After updating the `composer.json` file, run the following command to update the autoloader:

```terminal
composer dump-autoload
```

### UserFactory

The next step is to create the `UserFactory` class, which will be responsible for generating user entities. To achieve 
this, extend the `AbstractFactory` class provided by `Spiral\DatabaseSeeder\Factory`, and implement the required 
methods. 

To create it, run the following command:

```teriminal
php app.php create:factory UserFactory
```

Once it's created, modify the contents of the `app/database/Factory/UserFactory.php` file to look like the following:

```php app/database/Factory/UserFactory.php
namespace Database\Factory;

use App\Database\User;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class UserFactory extends AbstractFactory
{
    public function entity(): string
    {
        return User::class;
    }

    public function makeEntity(array $definition): User
    {
        return new User(
            username: $definition['username'],
            password: $definition['password'],
        );
    }

    public function definition(): array
    {
        return [
            'username' => $this->faker->userName(),
            'password' => \password_hash('secret', \PASSWORD_BCRYPT),
        ];
    }
}
```

### ThreadFactory

To generate test data for threads, create a `ThreadFactory` class. 

To create it, run the following command:

```teriminal
php app.php create:factory ThreadFactory
```

Once it's created, modify the contents of the `app/database/Factory/ThreadFactory.php` file to look like the following:

```php app/database/Factory/ThreadFactory.php
namespace Database\Factory;

use App\Database\Thread;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class ThreadFactory extends AbstractFactory
{
    public function entity(): string
    {
        return Thread::class;
    }

    public function makeEntity(array $definition): Thread
    {
        return new Thread(
            name: $definition['name']
        );
    }

    public function definition(): array
    {
        return [
            'name' => $this->faker->sentence,
        ];
    }
}
```

### MessageFactory

To generate test data for messages, create a `MessageFactory` class.

To create it, run the following command:

```teriminal
php app.php create:factory MessageFactory
```

Once it's created, modify the contents of the `app/database/Factory/MessageFactory.php` file to look like the following:

```php app/database/Factory/MessageFactory.php
namespace Database\Factory;

use App\Database\Message;
use Spiral\DatabaseSeeder\Factory\AbstractFactory;

final class MessageFactory extends AbstractFactory
{
    public function makeEntity(array $definition): object
    {
        return new Message(
            $definition['thread'],
            $definition['user'],
            $definition['text'],
        );
    }

    public function entity(): string
    {
        return Message::class;
    }

    public function definition(): array
    {
        return [
            'thread' => ThreadFactory::new()->make(),
            'user' => UserFactory::new()->make(),
            'text' => $this->faker->paragraph,
        ];
    }
}
```

To fill the database with test data, create seeders that use the previously created factories. 

### UserTableSeeder

To create a `UserTableSeeder` class, run the following command:

```teriminal
php app.php create:seeder UserTableSeeder
```

Once the `UserTableSeeder` class is created, modify the contents of the `app/database/Seeder/UserTableSeeder.php` file 
to look like the following:

```php app/database/Seeder/UserTableSeeder.php
namespace Database\Seeder;

use Database\Factory\UserFactory;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

final class UserTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        yield UserFactory::new(['username' => 'john'])->makeOne();
        yield UserFactory::new(['username' => 'bill'])->makeOne();
    }
}
```

> **Note**
> We will create only 2 users: `john` and `bill`, if you need more, you can do it in the same way.

### ThreadTableSeeder

To create a `ThreadTableSeeder` class, run the following command:

```teriminal
php app.php create:seeder ThreadTableSeeder
```

Once the `ThreadTableSeeder` class is created, modify the contents of the `app/database/Seeder/ThreadTableSeeder.php` 
file to look like the following:

```php app/database/Seeder/ThreadTableSeeder.php
namespace Database\Seeder;

use Database\Factory\ThreadFactory;
use Spiral\DatabaseSeeder\Seeder\AbstractSeeder;

final class ThreadTableSeeder extends AbstractSeeder
{
    public function run(): \Generator
    {
        yield ThreadFactory::new(['name' => 'First thread'])->makeOne();
    }
}
```

> **Note**
> We will create only 1 thread. It's enough for our example.

Now let's execute a console command that will populate the database with test records:

```terminal
php app.php db:seed
```

## Controller

### Login controller

At first, we need to create a controller that will authenticate users in our chat. 

Let's create it using scaffolder:

```terminal
php app.php create:controller login -a loginForm -a login -p 
```

> **Note**
> Use option `-a` to pre-generate controller actions and option `-p` to pre-load prototype extension.

The generated code:

```php app/src/Endpoint/Web/LoginController.php
namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class LoginController
{
    use PrototypeTrait;

    #[Route(route: 'path', name: 'name')]
    public function loginForm(): ResponseInterface
    {
    }

    #[Route(route: 'path', name: 'name')]
    public function login(): ResponseInterface
    {
    }
}
```

#### Login form

To render the login form, we will use the `Stempler` templating engine.

Here is a code for the login form.

```php app/src/Endpoint/Web/LoginController.php
use Psr\Http\Message\ServerRequestInterface;

final class LoginController
{
    // ...
    
    #[Route('/login', methods: ['GET'])]
    public function loginForm(ServerRequestInterface $request): ResponseInterface|string
    {
        return $this->views->render('login', [
            'csrf' => $request->getAttribute('csrfToken'),
            'errors' => [],
        ]);
    }
}
```

We use csrf token to protect the login form from CSRF attacks. The token is generated by
the `Spiral\Csrf\Middleware\CsrfMiddleware` middleware and stored in request attributes.

Let's create a view template `app/views/login.dark.php` for the login form:

```html app/views/login.dark.php

<html>
<head>
    <title>Login</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body>
<div class="min-h-screen bg-gray-100 flex flex-col justify-center sm:py-12">
    <div class="p-10 xs:p-0 mx-auto md:w-full md:max-w-md">
        <h1 class="font-bold text-center text-2xl mb-5">Your Logo</h1>
        <form action="/login" method="POST" class="bg-white shadow w-full rounded-lg divide-y divide-gray-200">
            <input type="hidden" name="csrf-token" value="{{ $csrf }}"/>

            @foreach ($errors ?? [] as $error)
            <div class="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded relative" role="alert">
                <strong class="font-bold">Error!</strong>
                <span class="block sm:inline">{{ $error }}</span>
            </div>
            @endforeach

            <div class="px-5 py-7">
                <label class="font-semibold text-sm text-gray-600 pb-1 block">Username</label>
                <input type="text" class="border rounded-lg px-3 py-2 mt-1 mb-5 text-sm w-full" name="username"/>

                <label class="font-semibold text-sm text-gray-600 pb-1 block">Password</label>
                <input type="password" name="password" class="border rounded-lg px-3 py-2 mt-1 mb-5 text-sm w-full"/>

                <button type="submit"
                        class="bg-blue-500 hover:bg-blue-600 text-white w-full py-2.5 rounded-lg text-sm text-center inline-block">
                    <span class="inline-block mr-2">Login</span>
                </button>
            </div>
        </form>
    </div>
</div>
</body>
</html>
```

#### Login handler

Now let's create a login handler. It will authenticate users and redirect them to the chat page.

We will store authentication tokens using Cycle ORM, so we need to set `AUTH_TOKEN_STORAGE` environment variable
to `cycle`:

```dotenv .env
AUTH_TOKEN_STORAGE=cycle
```

Now we need to create a request filter that will validate the login form data.

```php app/src/Endpoint/Web/Filter/LoginRequest.php
namespace App\Entrypoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class LoginRequest extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    #[Post]
    public string $password;

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => ['notEmpty', 'string'],
            'password' => ['notEmpty', 'string'],
        ]);
    }
}
```

and `InvalidCredentialsException` exception that will be thrown if the user credentials are invalid:

```php app/src/Application/Exception/InvalidCredentialsException.php
namespace App\Application\Exception;

final class InvalidCredentialsException extends \Exception
{

}
```

Now we can implement the login action:

```php app/src/Endpoint/Web/LoginController.php
use App\Application\Exception\InvalidCredentialsException;
use App\Entrypoint\Web\Filter\LoginRequest;

final class LoginController
{
    // ...
    
    #[Route('/login', methods: ['POST'])]
    public function login(LoginRequest $filter): ResponseInterface
    {
        $user = $this->users->findByUsername($filter->username);

        if (!$user || !\password_verify($filter->password, $user->getPassword())) {
            throw new InvalidCredentialsException('Invalid username or password!');
        }
    
        $token = $this->authTokens->create($user->jsonSerialize());
        $this->auth->start($token);
    
        return $this->response->redirect('/');
    }
}
```

> **Note**
> It would be a good idea to verify the user password in a special service. But for our example, it's enough.

After the user is authenticated, we create a new authentication token with a payload that contains the user data.

```php
[
    'id' => ...,
    'username' => ...,
]
```

> **Note**
> You can store any data in the token payload. Basically, a token should contain all the data that you need to identify
> the user. In most cases, `id` or `username` is enough.

#### Handle invalid credentials exception

Now we need to handle the `InvalidCredentialsException` exception and display an error message on the login form.

In our application we will store errors in the session.

> **Warning**
> Session is not the best place to store errors for sharing between requests. But for our example, it's enough.

Let's create a new service that will store errors in the session:

```php app/src/Endpoint/Web/SessionErrors.php
namespace App\Entrypoint\Web;

use Spiral\Prototype\Annotation\Prototyped;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Session\SessionSectionInterface;

#[Prototyped(property: 'errors')]
final class SessionErrors
{
    use PrototypeTrait;
    
    public function clear(): void
    {
        $this->session()->clear();
    }

    /**
     * @return array<non-empty-string, non-empty-string[]>
     */
    public function getErrors(): array
    {
        $errors = $this->session()->getAll();
        
        // clear errors after reading
        $this->clear();

        return $errors;
    }

    /**
     * @param non-empty-string $key
     * @param non-empty-string $error
     */
    public function addError(string $key, string $error): void
    {
        $this->session()->set($key, $error);
    }
    
    private function session(): SessionSectionInterface
    {
        return $this->session->getSection('errors');
    }
}
```

And run the following command to register the service as a prototype:

```bash
php app.php prototype:dump
```

Ok, now we can use our service as a prototype using `Spiral\Prototype\Traits\PrototypeTrait` with the property
name `errors`.

Let's create a new middleware that will handle the exception:

```php app/src/Endpoint/Web/Middleware/HandleInvalidCredentialsMiddleware.php
namespace App\Entrypoint\Web\Middleware;

use App\Application\Exception\InvalidCredentialsException;
use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use Psr\Http\Server\MiddlewareInterface;
use Psr\Http\Server\RequestHandlerInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class HandleInvalidCredentialsMiddleware implements MiddlewareInterface
{
    use PrototypeTrait;

    public function process(ServerRequestInterface $request, RequestHandlerInterface $handler): ResponseInterface
    {
        try {
            $response = $handler->handle($request);
            
            // Flush errors after successful request
            $this->errors->clear();

            return $response;
        } catch (InvalidCredentialsException $e) {
            // Add error to the session and redirect to the login form
            $this->errors->addError('username', $e->getMessage());

            return $this->response->redirect('/login');
        }
    }
}
```

And register the middleware in the `RoutesBootloader`:

```diff app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
    
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
+               \App\Entrypoint\Web\Middleware\HandleInvalidCredentialsMiddleware::class,
            ],
        ];
    }
}
```

To display errors on the login form we need to get them from the `SessionErrors` service and pass to the view:

```diff app/src/Endpoint/Web/LoginController.php
final class LoginController
{
    // ...
    
    #[Route('/login', methods: ['GET'])]
    public function loginForm(ServerRequestInterface $request): ResponseInterface|string
    {
        return $this->views->render('login', [
            'csrf' => $request->getAttribute('csrfToken'),
+           'errors' => $this->errors->getErrors(),
        ]);
    }
}
```

### Chat page

#### Application logic

To access the chat page, the user must be authenticated. So we need to create a middleware group for authenticated
users:

```diff app/src/Application/Bootloader/RoutesBootloader.php
+use Spiral\Core\Container\Autowire;
+use Spiral\Auth\Middleware\Firewall\RedirectFirewall;

final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...
    
    protected function middlewareGroups(): array
    {
        return [
            'web' => [
                // ...
            ],

+           'auth' => [
+               'middleware:web',
+               new Autowire(RedirectFirewall::class, ['uri' => new \Nyholm\Psr7\Uri('/login')]),
+           ],
        ];
    }
}
```

> **Note**
> A middleware group can extend another group. In our case, the `auth` group extends the `web` group. All the registered
> groups name will have the `middleware:` prefix.

Now let's create a chat controller:

```terminal
php app.php create:controller chat -a index -p 
```

Here is final code of the controller:

```php app/src/Endpoint/Web/ChatController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

final class ChatController
{
    use PrototypeTrait;

    #[Route('/', group: 'auth')]
    public function index(): string
    {
        return $this->views->render('chat', [
            'token' => $this->auth->getToken()->getID(),
        ]);
    }
}
```

> **Note**
> As you can see, we use the `auth` middleware group in `Route` attribute to protect the chat page.

Now let's create a view for the chat page:

```html app/views/chat.dark.php
<html>
<head>
    <title>Chat</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <script src="https://cdn.tailwindcss.com"></script>
    <meta name="x-bearer" content="{{ $token }}">
</head>
<body>
<div id="app" class="w-full h-screen"></div>
<script type="module" src="/frontend.js"></script>
</body>
</html>
```

As you noticed, we use the `x-bearer` meta tag to pass the authentication token to the frontend application. This
token will be used to authenticate the user in the Centrifugo server.

## Centrifugo

Spiral Framework provides a way to handle events from the Centrifugo server. In our example, we will handle two
types of events:

* `connect` - when a user connects to the Centrifugo server.
* `rpc` - when a user sends a message from the chat to the Centrifugo server.

### Connect event

When a user connects to the Centrifugo server it also sends an authentication token. We use this token to authenticate
identity of the user. If the token is valid, we return the user's ID and channels to which the user should be
automatically subscribed after connecting to the server, otherwise, user will be disconnected from the server.

To find a user by the token, we need to create a user provider that implements
the `Spiral\Auth\ActorProviderInterface`:

```php app/src/Endpoint/Centrifugo/UserProvider.php
namespace App\Endpoint\Centrifugo;

use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\TokenInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class UserProvider implements ActorProviderInterface
{
    use PrototypeTrait;

    public function getActor(TokenInterface $token): ?object
    {
        if (!isset($token->getPayload()['id'])) {
            return null;
        }

        return $this->users->findByPK($token->getPayload()['id']);
    }
}
```

And register it in the `AuthBootloader`:

```terminal
php app.php create:bootloader AuthBootloader
```

```php app/src/Application/Bootloader/AuthBootloader.php
namespace App\Application\Bootloader;

use App\Endpoint\Centrifugo\UserProvider;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Auth\AuthBootloader as BaseAuthBootloader;

final class AuthBootloader extends Bootloader
{
    public function init(BaseAuthBootloader $auth): void
    {
        $auth->addActorProvider(UserProvider::class);
    }
}
```

To activate the bootloader we need to register it in the `Kernel`:

```diff app/src/Application/Kernel.php
// ORM
CycleBridge\AuthTokensBootloader::class,
+\App\Application\Bootloader\AuthBootloader::class,
```

That's it. Now we can get the authenticated user (`Actor`) by a token from the `ActorProviderInterface`.

To handle the `connect` event, we need to create a `ConnectHandler` class. Let's create it:

```terminal
php app.php create:centrifugo-handler Connect
```

And modify the generated code:

```php app/src/Endpoint/Centrifugo/ConnectHandler.php
namespace App\Endpoint\Centrifugo;

use App\Database\User;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class ConnectHandler implements ServiceInterface
{
    use PrototypeTrait;

    public function __construct(
        private readonly ActorProviderInterface $actorProvider,
    ) {
    }

    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
            // Authenticate user with a given token from the connection request
            $authToken = $request->getData()['authToken'] ?? null;
            
            if ($authToken && $user = $this->getActor($authToken)) {
                $userId = $user->getId();
            } else {
                $request->error(403, 'Forbidden');
                return;
            }

            $request->respond(
                new ConnectResponse(
                    user: (string)$userId,
                    data: ['user' => $user->jsonSerialize()],
                    channels: ['chat'],
                ),
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }

    public function getActor(?string $authToken): ?User
    {
        if ($authToken && $token = $this->authTokens->load($authToken)) {
            return $this->actorProvider->getActor($token);
        }

        return null;
    }
}
```

As you can notice in the example above, we use `channels` field to automatically subscribe the user to the
desired channels after connecting to the server. And we don't need to subscribe the user to the channel manually
on the client side.

> **Note**
> When we respond to the request, we also pass the user's data to the `data` field to be able to use it in the
> frontend application.

Now we need to register the `ConnectHandler` as a service that should handle the `connect` event:

```php app/config/centrifugo.php
use App\Endpoint\Centrifugo\ConnectHandler;
use RoadRunner\Centrifugo\Request\RequestType;

return [
    'services' => [
        RequestType::Connect->value => ConnectHandler::class,
    ],
];
```

Now we will be able to handle the `connect` event.

### RPC event

RPC event is used to send messages from the client to the server. In our example, we will use this event to send list of
available messages in a given thread and to store a new message in the database when a user sends a message from the
chat.

To handle the `rpc` event, we need to create a `RpcHandler` class. Let's create it:

```terminal
php app.php create:centrifugo-handler Rpc -t=rpc
```

```php app/src/Endpoint/Centrifugo/RpcHandler.php
namespace App\Endpoint\Centrifugo;

use App\Database\Message;
use App\Database\User;
use RoadRunner\Centrifugo\Payload\RPCResponse;
use RoadRunner\Centrifugo\Request\RequestInterface;
use RoadRunner\Centrifugo\Request\RPC;
use Spiral\Core\InvokerInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class RpcHandler implements ServiceInterface
{
    use PrototypeTrait;

    public function __construct(
        private readonly InvokerInterface $invoker,
    ) {
    }

    /**
     * @param RPC $request
     */
    public function handle(RequestInterface $request): void
    {
        // Invoke a method based on the request method
        $result = match ($request->method) {
            // Return a list of messages in a given thread
            'thread:history' => $this->invoker->invoke(
                [$this, 'threadHistory'],
                $request->getData(),
            ),
            // Store a new message in the database
            'thread:publish' => $this->invoker->invoke(
                [$this, 'threadNewMessage'],
                $request->getData(),
            ),
            // Return an error if the method is not found
            default => ['error' => 'Not found', 'code' => 404]
        };

        try {
            $request->respond(new RPCResponse(data: $result));
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }

    private function threadNewMessage(
        int $id,
        string $message,
    ): array {
        $thread = $this->threads->findByPK($id);

        /** @var User $user */
        $user = $this->auth->getActor();

        $message = new Message($thread, $user, $message);
        $this->entityManager->persist($message)->run();

        $this->broadcast->publish(
            'chat',
            \json_encode([
                'type' => 'message',
                'message' => $message,
                'thread' => $thread,
            ]),
        );

        return ['message' => $message, 'thread' => $thread];
    }

    private function threadHistory(int $id): array
    {
        return ['messages' => $this->messages->findAllByThread($id)];
    }
}
```

All the requests to the `rpc` event will contain authentication token in the `authToken` field. Let's try to use
another approach to authenticate the user. We will use an interceptor to authenticate the user. Use interceptors is a 
good practice because it allows you to reuse the same authentication logic for different events.

Let's create the interceptor that will authenticate the user based on the `authToken` field:

```php app/src/Endpoint/Centrifugo/Interceptor/AuthInterceptor.php
namespace App\Endpoint\Centrifugo\Interceptor;

use Psr\EventDispatcher\EventDispatcherInterface;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Auth\AuthContext;
use Spiral\Auth\AuthContextInterface;
use Spiral\Core\CoreInterceptorInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\ScopeInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

final class AuthInterceptor implements CoreInterceptorInterface
{
    use PrototypeTrait;

    public function __construct(
        private readonly ScopeInterface $scope,
        private readonly ActorProviderInterface $actorProvider,
        private readonly ?EventDispatcherInterface $eventDispatcher = null,
    ) {
    }

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        $request = $parameters['request'];
        \assert($request instanceof RequestInterface);

        $authToken = $request->getData()['authToken'] ?? null;

        if (!$authToken || !$token = $this->authTokens->load($authToken)) {
            $request->error(403, 'Unauthorized');
            return null;
        }

        $auth = new AuthContext($this->actorProvider, $this->eventDispatcher);
        $auth->start($token);

        if ($auth->getActor() === null) {
            $request->error(403, 'Unauthorized');
            return null;
        }

        return $this->scope->runScope([
            AuthContextInterface::class => $auth,
        ], fn () => $core->callAction($controller, $action, $parameters));
    }
}
```

As we found out earlier, interceptors can be reused in different events. So we can use it not only for the `rpc` event
but also for the `connect` event.

Let's register the `AuthInterceptor` and the `RpcHandler` in the `centrifugo.php` config file:

```diff app/config/centrifugo.php
+use App\Endpoint\Centrifugo\Interceptor\AuthInterceptor;
use App\Endpoint\Centrifugo\ConnectHandler;
+use App\Endpoint\Centrifugo\RpcHandler;
use RoadRunner\Centrifugo\Request\RequestType;

return [
    'services' => [
        RequestType::Connect->value => ConnectHandler::class,
+       RequestType::RPC->value => RpcHandler::class,
    ],
+   'interceptors' => [
+       RequestType::RPC->value => [
+           AuthInterceptor::class,
+       ],
+       RequestType::Connect->value => [
+           AuthInterceptor::class,
+       ],
+   ],
];
```

> **Note**
> You can also use key `*` as wildcard to register the same interceptor for multiple events.

And modify the `ConnectHandler` to make it cleaner:

```diff app/src/Endpoint/Centrifugo/ConnectHandler.php
namespace App\Endpoint\Centrifugo;

use App\Entity\User;
use RoadRunner\Centrifugo\Payload\ConnectResponse;
use RoadRunner\Centrifugo\Request\Connect;
use RoadRunner\Centrifugo\Request\RequestInterface;
use Spiral\Auth\ActorProviderInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\RoadRunnerBridge\Centrifugo\ServiceInterface;

final class ConnectHandler implements ServiceInterface
{
    use PrototypeTrait;

-   public function __construct(
-       private readonly ActorProviderInterface $actorProvider,
-   ) {
-   }

    /** @param Connect $request */
    public function handle(RequestInterface $request): void
    {
        try {
-           // Authenticate user with a given token from the connection request
-           $authToken = $request->getData()['authToken'] ?? null;
-           if ($authToken && $user = $this->getActor($authToken)) {
-               $userId = $user->getId();
-           } else {
-               $request->error(403, 'Forbidden');
-               return;
-           }
+           $user = $this->auth->getActor();

            $request->respond(
                new ConnectResponse(
+                   user: (string)$user->getId(),
                    data: ['user' => $user->jsonSerialize()],
                    channels: ['chat'],
                ),
            );
        } catch (\Throwable $e) {
            $request->error($e->getCode(), $e->getMessage());
        }
    }

-   public function getActor(?string $authToken): ?User
-   {
-       ...
-   }
}
```

> **Note**
> Using the same approach, you can also use interceptors to handle events exceptions.

## VueJs application

We will use Vue.js to create a chat application. So let's create a new one.

### Installation

```terminal
npm init vue@latest
```

You can see the answers that I used in the example below:

```terminal
✔ Project name: … frontend
✔ Add TypeScript? … No
✔ Add JSX Support? … No
✔ Add Vue Router for Single Page Application development? … No
✔ Add Pinia for state management? … No
✔ Add Vitest for Unit testing? … No
✔ Add Cypress for both Unit and End-to-End testing? … No
✔ Add ESLint for code quality? … No
✔ Add Prettier for code formatting? … No

Scaffolding project in ./frontend...
Done.
```

> **See more**
> You can find out more information about the Vue.js in
> the [Official documentation](https://vuejs.org/guide/quick-start.html#try-vue-online).

We need also to install the `centrifugo` package from npm:

```terminal
cd frontend
npm install centrifuge -s
```

### Configuration

Let's open `vite.config.js` and modify it:

```js frontend/vite.config.js
import {fileURLToPath, URL} from 'node:url';
import {resolve} from 'path';
import {defineConfig} from 'vite';
import vue from '@vitejs/plugin-vue';

export default defineConfig({
    plugins: [
        vue()
    ],
    build: {
        lib: {
            entry: resolve(__dirname, 'src/main.js'),
            name: 'Chat',
            formats: ['cjs']
        },
        outDir: '../public'
    },
    resolve: {
        alias: {
            '@': fileURLToPath(new URL('./src', import.meta.url))
        }
    },
    define: {
        'process.env': {}
    }
})
```

And delete `frontend/public` and `frontend/src/` folders. We will create everything from scratch.

Ok, now we are ready to create a chat application.

Let's create `frontend/src/main.js` file:

```js frontend/src/main.js
import {createApp} from 'vue'
import App from './App.vue'
import {Centrifuge} from "centrifuge";

const app = createApp(App)

app.use({
    install(app, options) {
        // Get the auth token from x-bearer meta tag
        const authToken = document.getElementsByName('x-bearer')[0].getAttribute('content')
        
        // Register the auth token as a global property
        app.config.globalProperties.authToken = authToken

        // Create a new Centrifuge instance
        const centrifuge = new Centrifuge('ws://127.0.0.1:8081/connection/websocket', {
            data: {authToken}
        });

        // Store the user data in the global property when the client is connected
        centrifuge.on('connected', (ctx) => {
            app.config.globalProperties.user = ctx.data.user
        })

        centrifuge.connect();

        // Register the centrifuge instance as a global property
        app.config.globalProperties.centrifuge = centrifuge
    }
})

app.mount('#app')
```

When a client is connected to the Centrifugo server, it will receive an authenticated user data from the
`ConnectHandler` that we created earlier. We will register it as a `user` global property to use it in our components.

In this file we will connect to the Centrifugo server and register `centrifuge` as a global property to use it in our
components.

As you can see we use `authToken` from the `meta` tag. It will be used to authenticate the user on the server side via
Centrifugo.

### Components

#### Message

Let's create a `Message` component:

```html frontend/src/components/Message.vue
<template>
    <div class="chat-message">
        <div class="flex items-end" :class="{'justify-end': isOwnMessage}">
            <div class="flex flex-col space-y-2 text-xs mx-2 order-2 items-start">
                <div>
          <span class="px-4 py-2 rounded-lg inline-block rounded-bl-none"
                :class="{'bg-blue-600 text-white': isOwnMessage, 'bg-gray-300 text-gray-600': !isOwnMessage}">
             {{ message.text }}
          </span>
                </div>
                <span>
           {{ message.user.username }}
        </span>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        props: {
            message: Object
        },
        computed: {
            isOwnMessage() {
                return this.message.user.id === this.user.id
            }
        }
    }
</script>
```

We use `isOwnMessage` computed property to determine if the message is sent by the current user or not. If it is sent by
the current user, we will display it on the right side of the chat.

#### MessageForm

The `MessageForm` component will be used to send messages to the server. Let's create it:

```html frontend/src/components/MessageForm.vue
<template>
    <div class="border-t-2 border-gray-200 px-4 pt-4 mb-2 sm:mb-0">
        <div class="relative flex">
            <input v-model="message"
                   type="text"
                   @keyup.enter="sendMessage"
                   placeholder="Write your message!"
                   class="w-full focus:outline-none focus:placeholder-gray-400 text-gray-600 placeholder-gray-600 pl-6 bg-gray-200 rounded-md py-3">
            <div class="absolute right-0 items-center inset-y-0 hidden sm:flex" v-if="message.length > 1">
                <button type="button"
                        @click="sendMessage"
                        class="inline-flex items-center justify-center rounded-lg px-4 py-3 transition duration-500 ease-in-out text-white bg-blue-500 hover:bg-blue-400 focus:outline-none">
                    <span class="font-bold">Send</span>
                </button>
            </div>
        </div>
    </div>
</template>

<script>
    export default {
        props: {
            thread: Number
        },
        data() {
            return {
                message: ''
            }
        },
        methods: {
            sendMessage() {
                this.centrifuge.rpc('thread:publish', {
                    id: this.thread,
                    message: this.message,
                    authToken: this.authToken
                }).then(() => {
                    this.message = ''
                })
            }
        }
    }
</script>
```

The component will send a message to the server
via [centrifuge.rpc](https://github.com/centrifugal/centrifuge-js#rpc-method) with the `thread:publish` method. As we 
need to pass the `authToken` to authenticate the user on the server side.

#### Messages

The `Messages` component will be used to display all messages from the thread:

```html frontend/src/components/Messages.vue

<template>
    <div id="messages"
         ref="messages"
         class="flex flex-col space-y-4 p-3 overflow-y-auto scrollbar-thumb-blue scrollbar-thumb-rounded scrollbar-track-blue-lighter scrollbar-w-2 scrolling-touch">
        <Message v-for="message in messages" :key="message.id" :message="message"/>
    </div>

    <MessageForm :thread="thread"/>
</template>

<script>
    import MessageForm from "./MessageForm.vue";
    import Message from "./Message.vue"

    function delay(time) {
        return new Promise(resolve => setTimeout(resolve, time));
    }

    export default {
        components: {
            Message, MessageForm
        },
        props: {
            thread: Number
        },
        mounted() {
            // Get all messages for the thread from the server
            this.centrifuge.rpc('thread:history', {
                id: this.thread,
                authToken: this.authToken
            }).then(ctx => {
                this.messages = ctx.data.messages;
                delay(10).then(() => this.scrollToBottom());
            })

            // Listen for new messages from the server
            this.centrifuge.on('publication', ctx => {
                if (
                        ctx.channel !== 'chat' ||
                        ctx.data.type !== 'message' ||
                        ctx.data.thread.id !== this.thread
                ) {
                    return;
                }

                this.messages.push(ctx.data.message);

                delay(10).then(() => this.scrollToBottom());
            })
        },
        methods: {
            scrollToBottom() {
                this.$refs.messages.scrollTop = this.$refs.messages.scrollHeight;
            }
        },
        data() {
            return {
                messages: []
            }
        },
    }
</script>
```

To listen for new messages we will use the `publication` event of the `centrifuge` object. This event will be triggered
every time a new message is published to the `chat` channel. We check if the message is from the current thread and if
yes we add it to the `messages` array.

To display the history of the thread we will use the `thread:history` method of the `centrifuge.rpc` method. It will
return the all the messages from the thread.

And the last thing is to scroll to the bottom of the messages list when a new message is received. But whe have to do it
with a delay to give Vue time to update the DOM.

#### App

The `App` component is the root component of our frontend application.

```html frontend/src/App.vue
<script setup>
import Messages from "./components/Messages.vue";
</script>

<template>
    <div class="flex-1 p:2 sm:p-6 justify-between flex flex-col h-screen">
        <Messages :thread="1"/>
    </div>
</template>
```

It's super simple. It just renders the `Messages` component.

> **Note**
> For our example we will use only one thread with id `1`. We won't implement the ability to create new threads or
> switch between them. But you can easily do it by yourself.

### Build the frontend

Now we can build the frontend:

```terminal
cd ./frontend
npm run build
```

After the build is finished we will have the `frontend.js` file in the `/public` folder in the root of the project.

## Run the application

Ok, now we have the backend and the frontend. Let's run the application:

```terminal
./rr serve
```

Open the application in your browser: http://127.0.0.1:8080 and use `bill` or `john` as a username and `secret` as a
password to login. After authentication, you will be redirected to the chat page where you can send messages to the
server.

<hr>

## What's next?

Spiral provides a lot of pre-build functionality for you. Read the following sections to gain more insights:

- [Routing](../http/routing.md)
- [Authenticating users](../security/authentication.md)
- [WebSockets](../websockets/configuration.md)
- [Broadcasting](../websockets/broadcasting.md)
- [Deployment](../start/deployment.md)