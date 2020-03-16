# 文件和目录

Spiral 提供了简单易用的文件系统操作组件。该组件在所有的项目类型中均可使用。

## Directory Registry
## 目录注册表

绝大多数的 Spiral 组件都依赖于目录注册表，而不是硬编码路径。目录注册表可以通过 `Spiral\Boot\DirectoriesInterface` 提供的方法来使用。

配置目录注册表的操作是在应用入口文件（app.php）中进行：

```php
$app = \App\App::init([
    'root'      => __DIR__,
    'customDir' => __DIR__ . '/custom'
]);
```

或者在引导程序中也可以进行目录注册表的配置：

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

在需要访问注册表中的某个目录的路径时，可以依赖注入 `DirectoriesInterface` 接口：

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

> 在配置文件中，也可以使用 Spiral 提供的全局 `directory` 函数。但是要注意，这个函数只在 Spiral 框架范围内可用，因为它依赖全局容器的作用域。

## 文件

使用 `Spiral\Files\FilesInterface` 组件可以对文件系统进行操作：

```php
namespace App\Controller;

use Spiral\Files\FilesInterface;

class HomeController
{
    public function index(FilesInterface $files)
    {
        // 递归获取根目录下的所有文件
        dump($files->getFiles(directory('root')));
    }
}
```

在可以使用 `PrototypeTraits` 的地方，还可以通过它提供的 `files` 属性来访问文件对象实例：

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

### 创建文件夹

要确保目标文件夹存在（不存在时自动创建），可以通过 `ensureDirectory` 方法，该方法的第二个参数可以用来指定访问权限：

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
            FilesInterface::READONLY // 或者 FilesInterface::RUNTIME 使目录和文件可编辑
        );
    }
}
```

检查目录是否存在：

```php
dump($files->isDirectory(__DIR__));
```

### 删除目录

删除指定目录及该目录下的内容：

```php
$files->deleteDirectory('custom');
```

只删除指定目录下的内容，不删除目录本身：

```php
$files->deleteDirectory('custom', true);
```

## 文件信息

检查文件是否存在：

```php
dump($files->exists(__FILE__)); // bool
```

获取文件的创建或更新时间：

```php
dump($files->time(__FILE__)); // unix timestamp
```

获取文件的 MD5 值；

```php
dump($files->md5('filename'));
```

获取文件名中的扩展名：

```php
dump($files->extension(__FILE__)); // without leading "."
```

检查指定的路径是否文件：

```php
dump($files->isFile(__DIR__));
```

获取文件的大小：

```php
dump($files->size(__DIR__));
```

### 权限

获取指定文件/目录的权限：

```php
dump($files->getPermissions(__FILE__)); // int
```

设置文件权限：

```php
$files->setPermissions(__FILE__, 0777)
```

常量与文件权限掩码关系：

常量                     | 掩码
---                      | ---
FilesInterface::READONLY | 644
FilesInterface::RUNTIME  | 666

## 移动和复制

复制文件：

```php
$files->copy('old-path', 'new-path');
```

移动文件：

```php
$files->move('old-path', 'new-path');
```

### 临时文件：

*生成*临时文件名：

```php
dump($files->tempFilename());
```

生成特定后缀的临时文件名：

```php
dump($files->tempFilename('php'));
```

在指定的路径下生成临时文件名：

```php
dump($files->tempFilename('php', __DIR__));
```

## 读写操作

文件组件还提供了多个方法，用来以原子方式（无需获取文件资源）处理文件内容：

### 创建/写入文件

把内容写入到指定文件（排它锁）：

```php
$files->write('filename', 'data');
```

创建/写入文件并制定文件的访问权限：

```php
$files->write('filename', 'data', 0777);
```

检查并自动创建路径中不存在的目录：

```php
$files->write('filename', 'data', 0777, true);
```

> 请注意，如果文件不可写，需要处理 `Spiral\Files\Exception\WriteErrorException` 异常。

### 增加内容

把内容追加到文件：

```php
$files->append('filename', 'data');
```

增加内容到文件并指定文件权限：

```php
$files->append('filename', 'data', 0777);
```

追加/创建文件，并自动创建路径中不存在的目录：

```php
$files->append('filename', 'data', 0777, true);
```

### Touch

Linux 下的 touch 命令用于修改文件或目录的时间属性，如果文件不存在则会自动创建一个新的。Spiral 中文件对象的 `touch` 方法功能与之类似：

```php
$files->touch('filename');
```

Touch 文件并确保它具有指定的权限：

```php
$files->touch('filename', 0777);
```

### 读取文件

读取指定文件的内容：

```php
dump($files->read('filename'));
```

> 注意：如果文件不存在，需要处理 `Spiral\Files\Exception\FileNotFoundException` 异常。
