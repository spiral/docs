# The Basics — Files and Directories

The framework provides a simple component `spiral/files` to work with the filesystem.

## Directory Registry

Most of the Spiral components rely on the directory registry instead of hard-coded paths. The registry is represented
using `Spiral\Boot\DirectoriesInterface`.

> **Note**
> Read more about application directory structure in the [Getting started — Directory Structure](../start/structure.md)
> section.

You can configure application specific directories in the app entry point (`app.php`):

```php app.php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__,
        'uploadDir' => __DIR__ . '/upload'
    ]
)->run();
```

Or using the Bootloader:

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\DirectoriesInterface;

final class AppBootloader extends Bootloader
{
    public function boot(DirectoriesInterface $dirs): void
    {
        $dirs->set(
            'uploadDir',
            $dirs->get('root') . '/upload'
        );
    }
}
```

To access the directory paths:

```php
use Spiral\Boot\DirectoriesInterface;

final class UploadService {
    public function __construct(
        private readonly DirectoriesInterface $dirs
    ) {}
    
    public function store(UploadedFile $file) {
        $filePath = $this->dirs->get('uploadDir') . $file->getFilename();
        // ...
    }
}
```

> **Note**
> You can also use the short function `directory` inside your config files. Note that this function does not work 
> outside of the framework as it relies on the global container scope.


If you would like a directory names always be constrained by a specific method, you may create a helper class, for
example `AppDirectories`.

<details>
  <summary>Click to show an example.</summary>

```php app/src/Application/AppDirectories.php
namespace App\Application;

use Spiral\Boot\DirectoriesInterface;

final class AppDirectories
{
    public function __construct(
        private readonly DirectoriesInterface $directories
    ) {
    }

    /**
     * Application root directory.
     * @return non-empty-string
     */
    public function getRoot(?string $path = null): string
    {
        return $this->buildPath('root', $path);
    }

    /**
     * Application directory.
     * @return non-empty-string
     */
    public function getApp(?string $path = null): string
    {
        return $this->buildPath('app', $path);
    }

    /**
     * Public directory.
     * @return non-empty-string
     */
    public function getPublic(?string $path = null): string
    {
        return $this->buildPath('public', $path);
    }

    /**
     * Runtime directory.
     * @return non-empty-string
     */
    public function getRuntime(?string $path = null): string
    {
        return $this->buildPath('runtime', $path);
    }

    /**
     * Runtime cache directory.
     * @return non-empty-string
     */
    public function getCache(?string $path = null): string
    {
        return $this->buildPath('cache', $path);
    }

    /**
     * Vendor libraries directory.
     * @return non-empty-string
     */
    public function getVendor(?string $path = null): string
    {
        return $this->buildPath('vendor', $path);
    }

    /**
     * Config directory.
     * @return non-empty-string
     */
    public function getConfig(?string $path = null): string
    {
        return $this->buildPath('config', $path);
    }

    /**
     * Resources directory.
     * @return non-empty-string
     */
    public function getResources(?string $path = null): string
    {
        return $this->buildPath('resources', $path);
    }

    private function buildPath(string $key, ?string $path = null): string
    {
        return \rtrim($this->directories->get($key), '/') . ($path ? '/' . \ltrim($path, '/') : '');
    }
}
```
</details>

## Files

Use the `Spiral\Files\FilesInterface` component to work with the filesystem:

```php
use Spiral\Files\FilesInterface;

final class FileService
{   
    public function __construct(
        private readonly FilesInterface $files,
        private readonly AppDirectories $dirs
    ) {}

    public function getRootFiles()
    {
        // get all files from root directory recursively
        dump(
            $files->getFiles($dirs->getRoot())
        );
    }
}
```

You can also access this instance using prototyped property `files`:

```php
use Spiral\Prototype\Traits\PrototypeTrait;

final class FileService
{
    use PrototypeTrait;
   
    public function __construct(
        private readonly AppDirectories $dirs
    ) {}
    
    public function store()
    {
        dump($this->files->exists(__FILE__)); // true
    }
}
```

> **Note**
> Read more about prototyped properties in [The Basics — Prototyping](../basics/prototype.md) section.

### Create Directory

To ensure that a given directory exists, use method `ensureDirectory`, the second argument accepts the access
permission:

```php
public function store()
{
    $this->files->ensureDirectory(
        $dirs->get('customDir'),
        FilesInterface::READONLY // or FilesInterface::RUNTIME for editable dirs and files
    );
}
```

To check if a directory exists:

```php
dump($files->isDirectory(__DIR__));
```

### Delete Directory

To delete a directory and its content:

```php
$files->deleteDirectory('custom');
```

To delete directory content only:

```php
$files->deleteDirectory('custom', true);
```

### File Stats

To check if a file exists:

```php
dump($files->exists(__FILE__)); // bool
```

To get file creation/update time:

```php
dump($files->time(__FILE__)); // unix timestamp
```

To get file MD5:

```php
dump($files->md5('filename'));
```

To get a filename extension:

```php
dump($files->extension(__FILE__)); // without leading "."
```

To check if the path is a file:

```php
dump($files->isFile(__DIR__));
```

To get the file size:

```php
dump($files->size(__DIR__));
```

### Permissions

To get file/directory permissions:

```php
dump($files->getPermissions(__FILE__)); // int
```

To set file permissions:

```php
$files->setPermissions(__FILE__, 0777)
```

Use constants to control the file mode:

| Constant                 | Value |
|--------------------------|-------|
| FilesInterface::READONLY | 644   |
| FilesInterface::RUNTIME  | 666   |

### Move and Copy

To copy a file from one path to another:

```php
$files->copy('old-path', 'new-path');
```

To move a file:

```php
$files->move('old-path', 'new-path');
```

### Temporary Files

To *issue* a temporary filename:

```php
dump($files->tempFilename());
```

To issue a temporary filename with a specific extension:

```php
dump($files->tempFilename('php'));
```

To issue a temporary filename in a specific location:

```php
dump($files->tempFilename('php', __DIR__));
```

## Read and Write operations

The component provides multiple methods to operate with file content in an atomic way (without acquiring the file
resource):

### Write/Create file

To write content to a file (exclusive lock):

```php
$files->write('filename', 'data');
```

To write/create a file and ensure a proper access mode:

```php
$files->write('filename', 'data', 0777);
```

To check and automatically create a file directory:

```php
$files->write('filename', 'data', 0777, true);
```

> **Note**
> Make sure to handle `Spiral\Files\Exception\WriteErrorException` if the file can is not writable.

### Append Content

To append file content:

```php
$files->append('filename', 'data');
```

To append and ensure a file mode:

```php
$files->append('filename', 'data', 0777);
```

To append/create a file and ensure that a target directory exists:

```php
$files->append('filename', 'data', 0777, true);
```

### Touch

To touch the file and create it, if missing:

```php
$files->touch('filename');
```

To touch the file and ensure a file mode:

```php
$files->touch('filename', 0777);
```

### Read the file

To read file content:

```php
dump($files->read('filename'));
```

> **Note**
> Make sure to handle `Spiral\Files\Exception\FileNotFoundException` when files are not found.
