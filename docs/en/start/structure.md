# Getting started — Directory Structure

Spiral does not impose any directory structure for your application, so you have the flexibility to organize your files 
and directories. However, it provides a recommended structure that can be used as a starting point. This structure can 
also be easily modified.

## Directories

The default directory structure can be controlled via `mapDirectories` method of Kernel class. By default, all
application directories will be calculated based on `root` using the following pattern:

| Directory | Value                  |
|-----------|------------------------|
| root      | **set by user**        |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| vendor    | **root**/vendor        |

Some components will declare their own directories such as:

| Component         | Directory  | Value              |
|-------------------|------------|--------------------|
| spiral/views      | views      | **app**/views      |
| spiral/translator | locale     | **app**/locale     |
| spiral/migrations | migrations | **app**/migrations |

## Init Directories

You can set the value of the `root` directory, or any other directory, in your `app.php` file.

```php
$app = \App\Application\Kernel::create(
    directories: ['root' => __DIR__]
)->run();
```

For example, if you wanted to change the location of the `runtime` directory to the system's temporary directory, you
would do this:

```php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__, 
        'runtime' => \sys_get_temp_dir()
    ]
)->run();
```

You can access the paths of various directories in your application using the `Spiral\Boot\DirectoriesInterface` class.
This interface provides methods to access the paths of the various directories that are defined through
the `mapDirectories` method.

Here's an example of how you can use the `DirectoriesInterface` class to access the path of the `runtime` directory:

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('runtime') . 'uploads/' . $file->getFilename();
        // ...
    }
}
```

You can also use the function `directory` inside the global IoC scope (config files, controllers, service code).

```php app/config/cache.php
return [
    'storages' => [
        'file' => [
            'path' => directory('runtime') . 'cache',
        ],   
    ],
];
```

## Namespaces

By default, all skeleton applications use `App` root namespace pointing to `app/src` directory. You can change the base
to any desired namespace in `composer.json`:

```json composer.json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```

## Application Structure

The structure we provided below is a common structure used in many PHP applications, and it can serve as a starting
point for your projects. By following this structure, you can organize your code in a logical and maintainable
way, making it easier to build and scale your applications over time. Of course, you may need to make adjustments to fit
the specific needs of your project, but this structure provides a solid foundation for most applications.

```
- Endpoint
    - Web
        - ...
        - Filter
            - ...
        - Middleware
            - ...
        - Interceptor
            - ...
        - DataGrid
            - ...
        - routes.php
    - Console
        - Interceptor
            - ...
        - ...
    - RPC
        - Interceptor
            - ...
        - ...
    - Temporal
        - Workflow
            - ...
        - Activity
            - ...
    - Centrifugo
        - Interceptor
        - ...
- Application
    - Bootloader
        - ...
    - Exception
        - SomeException.php
        - Renderer
            - ViewRenderer.php
    - Kernel.php
- Domain
    - User
        - Entity
            - User.php
        - Service
            - StoreUserService.php
        - Repository
            - UserRepositoryInterface.php
        - Exception
            - UserNotFoundException.php
- Infrastructure
    - Persistence
        - CycleUserRepository.php
    - CycleORM
        - Typecaster
            - UuidTypecast.php
    - Interceptor
        - LogInterceptor.php
```

#### Here's a brief explanation of the directories and files in this structure:

- **Endpoint**: This directory contains the entry points for your application, including HTTP endpoints (in the Web
  subdirectory), command-line interfaces (in the Console subdirectory), and gRPC services (in the RPC subdirectory).

- **Application**: This directory contains the core of your application, including the Kernel class that boots your
  application, the Bootloader classes that register services with the container, and the Exception directory that
  contains exception handling logic.

- **Domain**: This directory contains your domain logic, organized by subdomains. For example, an Entity for the User
  model, a Service for storing new users, a Repository for fetching users from the database, and an Exception for
  handling user-related errors.

- **Infrastructure**: This directory contains the infrastructure code for your application, including the Persistence
  directory for database-related code, the CycleORM directory for ORM-related code, and the Interceptor directory for
  global interceptors.

<hr>

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Kernel and Environment](../framework/kernel.md)
* [Files and Directories](../basics/files.md)