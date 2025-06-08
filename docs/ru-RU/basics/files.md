# Основы — Файлы и директории

Фреймворк предоставляет простой компонент `spiral/files` для работы с файловой системой.

## Реестр директорий

Большинство компонентов Spiral полагаются на реестр директорий вместо жестко заданных путей. Реестр представлен
с помощью `Spiral\Boot\DirectoriesInterface`.

> **Узнать больше**
> Прочитайте больше о структуре директорий приложения в разделе [Начало работы — Структура директорий](../start/structure.md).

Вы можете сконфигурировать специфичные для приложения директории в точке входа приложения (`app.php`):

```php app.php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__,
        'uploadDir' => __DIR__ . '/upload'
    ]
)->run();
```

Или используя Bootloader:

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

Для доступа к путям директорий:

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

> **Примечание**
> Вы также можете использовать короткую функцию `directory` внутри ваших конфигурационных файлов. Обратите внимание, что эта функция не работает 
> вне фреймворка, поскольку полагается на глобальную область видимости контейнера.


Если вы хотите, чтобы имена директорий всегда были ограничены определенным методом, вы можете создать вспомогательный класс, например `AppDirectories`.

<details>
  <summary>Нажмите, чтобы показать пример.</summary>

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
     * Корневая директория приложения.
     * @return non-empty-string
     */
    public function getRoot(?string $path = null): string
    {
        return $this->buildPath('root', $path);
    }

    /**
     * Директория приложения.
     * @return non-empty-string
     */
    public function getApp(?string $path = null): string
    {
        return $this->buildPath('app', $path);
    }

    /**
     * Публичная директория.
     * @return non-empty-string
     */
    public function getPublic(?string $path = null): string
    {
        return $this->buildPath('public', $path);
    }

    /**
     * Директория выполнения.
     * @return non-empty-string
     */
    public function getRuntime(?string $path = null): string
    {
        return $this->buildPath('runtime', $path);
    }

    /**
     * Директория кэша выполнения.
     * @return non-empty-string
     */
    public function getCache(?string $path = null): string
    {
        return $this->buildPath('cache', $path);
    }

    /**
     * Директория библиотек vendor.
     * @return non-empty-string
     */
    public function getVendor(?string $path = null): string
    {
        return $this->buildPath('vendor', $path);
    }

    /**
     * Директория конфигурации.
     * @return non-empty-string
     */
    public function getConfig(?string $path = null): string
    {
        return $this->buildPath('config', $path);
    }

    /**
     * Директория ресурсов.
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

## Файлы

Используйте компонент `Spiral\Files\FilesInterface` для работы с файловой системой:

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
        // получить все файлы из корневой директории рекурсивно
        dump(
            $files->getFiles($dirs->getRoot())
        );
    }
}
```

Вы также можете получить доступ к этому экземпляру, используя prototype-свойство `files`:

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

> **Узнать больше**
> Прочитайте больше о prototype-свойствах в разделе [Основы — Прототипирование](../basics/prototype.md).

### Создание директории

Чтобы убедиться, что данная директория существует, используйте метод `ensureDirectory`, второй аргумент принимает права
доступа:

```php
public function store()
{
    $this->files->ensureDirectory(
        $dirs->get('customDir'),
        FilesInterface::READONLY // или FilesInterface::RUNTIME для редактируемых директорий и файлов
    );
}
```

Чтобы проверить, существует ли директория:

```php
dump($files->isDirectory(__DIR__));
```

### Удаление директории

Чтобы удалить директорию и её содержимое:

```php
$files->deleteDirectory('custom');
```

Чтобы удалить только содержимое директории:

```php
$files->deleteDirectory('custom', true);
```

### Статистика файлов

Чтобы проверить, существует ли файл:

```php
dump($files->exists(__FILE__)); // bool
```

Чтобы получить время создания/обновления файла:

```php
dump($files->time(__FILE__)); // unix timestamp
```

Чтобы получить MD5 файла:

```php
dump($files->md5('filename'));
```

Чтобы получить расширение файла:

```php
dump($files->extension(__FILE__)); // без ведущей "."
```

Чтобы проверить, является ли путь файлом:

```php
dump($files->isFile(__DIR__));
```

Чтобы получить размер файла:

```php
dump($files->size(__DIR__));
```

### Права доступа

Чтобы получить права доступа файла/директории:

```php
dump($files->getPermissions(__FILE__)); // int
```

Чтобы установить права доступа файла:

```php
$files->setPermissions(__FILE__, 0777)
```

Используйте константы для управления режимом файла:

| Константа                | Значение |
|--------------------------|----------|
| FilesInterface::READONLY | 644      |
| FilesInterface::RUNTIME  | 666      |

### Перемещение и копирование

Чтобы скопировать файл из одного пути в другой:

```php
$files->copy('old-path', 'new-path');
```

Чтобы переместить файл:

```php
$files->move('old-path', 'new-path');
```

### Временные файлы

Чтобы *создать* имя временного файла:

```php
dump($files->tempFilename());
```

Чтобы создать имя временного файла с определенным расширением:

```php
dump($files->tempFilename('php'));
```

Чтобы создать имя временного файла в определенном месте:

```php
dump($files->tempFilename('php', __DIR__));
```

## Операции чтения и записи

Компонент предоставляет множество методов для работы с содержимым файла атомарным способом (без захвата файлового
ресурса):

### Запись/создание файла

Чтобы записать содержимое в файл (эксклюзивная блокировка):

```php
$files->write('filename', 'data');
```

Чтобы записать/создать файл и обеспечить правильный режим доступа:

```php
$files->write('filename', 'data', 0777);
```

Чтобы проверить и автоматически создать директорию файла:

```php
$files->write('filename', 'data', 0777, true);
```

> **Примечание**
> Убедитесь, что обрабатываете `Spiral\Files\Exception\WriteErrorException`, если файл недоступен для записи.

### Добавление содержимого

Чтобы добавить содержимое к файлу:

```php
$files->append('filename', 'data');
```

Чтобы добавить и обеспечить режим файла:

```php
$files->append('filename', 'data', 0777);
```

Чтобы добавить/создать файл и убедиться, что целевая директория существует:

```php
$files->append('filename', 'data', 0777, true);
```

### Touch

Чтобы "коснуться" файла и создать его, если отсутствует:

```php
$files->touch('filename');
```

Чтобы "коснуться" файла и обеспечить режим файла:

```php
$files->touch('filename', 0777);
```

### Чтение файла

Чтобы прочитать содержимое файла:

```php
dump($files->read('filename'));
```

> **Примечание**
> Убедитесь, что обрабатываете `Spiral\Files\Exception\FileNotFoundException`, когда файлы не найдены.
