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

### Routing



### Annotated Routing

### Domain Core

## Scaffold Database

### Define ORM Entities

### Generate Migration

### Create Relations 

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
