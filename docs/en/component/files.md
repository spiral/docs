# Files and Directories

The framework provides a simple component to work with the filesystem. The component is available in all of the
application bundles.

## Directory Registry

Most of the spiral components rely on the directory registry instead of hard-coded paths. The registry represented 
using `Spiral\Boot\DirectoriesInterface`.

You can configure application specific directories in the app entry point (app.php):

```php
$app = \App\App::create([
    'root'      => __DIR__,
    'customDir' => __DIR__ . '/custom'
])->run();
```

Or using the Bootloader:

```php
namespace App\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function boot(DirectoriesInterface $directories)
    {
        $directories->set(
            'customDir',
            $directories->get('root') . '/custom'
        );
    }
}
```

To access the directory paths:

```php
namespace App\Controller;

use Spiral\Boot\DirectoriesInterface;

class HomeController
{
    public function index(DirectoriesInterface $dirs)
    {
        dump($dirs->get('root'));
        dump($dirs->get('customDir'));

        dump($dirs->getAll());
    }
}
```

> **Note**
> You can also use the short function `directory` inside your config files. Note, this
> function does not work outside of the framework as it relies on the global container scope.

## Files

Use the `Spiral\Files\FilesInterface` component to work with the filesystem:

```php
namespace App\Controller;

use Spiral\Files\FilesInterface;

class HomeController
{
    public function index(FilesInterface $files)
    {
        // get all files from root directory recursively
        dump($files->getFiles(directory('root')));
    }
}
```

You can also access this instance using prototyped property `files`:

```php
namespace App\Controller;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        dump($this->files->exists(__FILE__)); // true
    }
}
```

### Create Directory

To ensure that given directory exists use method `ensureDirectory`, the second argument accepts the access permission:

```php
namespace App\Controller;

use Spiral\Boot\DirectoriesInterface;
use Spiral\Files\FilesInterface;
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(DirectoriesInterface $dirs)
    {
        $this->files->ensureDirectory(
            $dirs->get('customDir'),
            FilesInterface::READONLY // or FilesInterface::RUNTIME for editable dirs and files
        );
    }
}
```

To check if directory exists:

```php
dump($files->isDirectory(__DIR__));
```

### Delete Directory

To delete directory and its content:

```php
$files->deleteDirectory('custom');
```

To delete directory content only:

```php
$files->deleteDirectory('custom', true);
```

### File Stats

To check if file exists:

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

To get filename extension:

```php
dump($files->extension(__FILE__)); // without leading "."
```

To check if path is file:

```php
dump($files->isFile(__DIR__));
```

To get file size:

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

Use constants to control file mode:

| Constant                 | Value |
|--------------------------|-------|
| FilesInterface::READONLY | 644   |
| FilesInterface::RUNTIME  | 666   |

### Move and Copy

To copy file from one path to another:

```php
$files->copy('old-path', 'new-path');
```

To move file:

```php
$files->move('old-path', 'new-path');
```

### Temporary Files

To *issue* temporary filename:

```php
dump($files->tempFilename());
```

To issue temporary filename with a specific extension:

```php
dump($files->tempFilename('php'));
```

To issue temporary filename in a specific location:

```php
dump($files->tempFilename('php', __DIR__));
```

## Read and Write operations

The component provides multiple methods to operate with file content in an atomic way (without acquiring the file 
resource):

### Write/Create file

To write the content to the file (exclusive lock):

```php
$files->write('filename', 'data');
```

To write/create file and ensure proper access mode:

```php
$files->write('filename', 'data', 0777);
```

To check and automatically create the file directory:

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

To append and ensure file mode:

```php
$files->append('filename', 'data', 0777);
```

To append/create file and ensure that target directory exists:

```php
$files->append('filename', 'data', 0777, true);
```

### Touch

To touch the file and create it if missing:

```php
$files->touch('filename');
```

To touch file and ensure file mode:

```php
$files->touch('filename', 0777);
```

### Read the file

To read file content:

```php
dump($files->read('filename'));
```

> **Note**
> Make sure to handle `Spiral\Files\Exception\FileNotFoundException` when files not found.
