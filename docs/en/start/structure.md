# Getting started — Directory Structure

The Spiral Framework does not impose any directory structure for your application, so you have the flexibility to
organize your files and directories. However, it provides a recommended structure that can be used as
a starting point. This structure can also be easily modified.

## Directories

The default directory structure can be controlled via `mapDirectories` method of Kernel class. By default, all
application directories will be calculated based on `root` using the following pattern:

| Directory | Value                  |
|-----------|------------------------|
| root      | **set by user**        |
| vendor    | **root**/vendor        |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |

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
]
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

## What's Next?

Now, dive deeper into the fundamentals by reading some articles:

* [Kernel and Environment](../framework/kernel.md)
* [Files and Directories](../basics/files.md)