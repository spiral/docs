# Компонент — Хранилище и облачное распределение

Spiral предлагает комплексное решение для хранения файлов и распределения через компоненты `spiral/storage` и
`spiral/distribution`. Компонент `spiral/storage` предоставляет мощную абстракцию хранилища, используя
возможности PHP-пакета Flysystem, предлагая удобные драйверы для работы как с локальными файловыми системами, так и
с Amazon S3. Компонент `spiral/distribution`, который интегрирован с компонентом `spiral/storage`, отвечает
за генерацию публичных HTTP-ссылок для ресурсов, хранящихся через компонент хранилища.

## Хранилище

Компонент `spiral/storage` предоставляет мощную абстракцию хранилища благодаря
замечательному PHP-пакету [Flysystem](https://github.com/thephpleague/flysystem) от Frank de Jonge. Интеграция компонента Storage
предоставляет простые драйверы для работы с локальными файловыми системами и Amazon S3. Более того, невероятно легко
переключаться между этими опциями хранения между вашей локальной машиной разработки и production-сервером, поскольку API
остается одинаковым для каждой системы.

> **Примечание**
> В отличие от классических файловых систем, компонент хранилища предоставляет API, который обеспечивает операции
> записи файла, проверки его существования, чтения и получения публичного адреса этого файла. Все операции (которые
> есть у классических файловых систем) для работы с директориями или списком файлов недоступны.

### Установка

Для установки компонента:

```terminal
composer require spiral/storage
```

Убедитесь, что добавили `Spiral\Storage\Bootloader\StorageBootloader` в список загрузчиков (Bootloader) в вашем приложении:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Storage\Bootloader\StorageBootloader::class,
        // ...
    ];
}
```

Узнайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Storage\Bootloader\StorageBootloader::class,
    // ...
];
```

Узнайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

### Конфигурация

Файл конфигурации хранилища по умолчанию расположен в `app/config/storage.php`. Этот файл позволяет настраивать
серверы и конкретные хранилища, также известные как `buckets`. Секция `servers` — это место, где вы можете настроить
серверы, которые будет использовать ваше приложение, в то время как секция `buckets` позволяет указать, какие хранилища
будут использоваться вашими серверами.

Например, простейшая конфигурация с одним сервером и двумя хранилищами может выглядеть так:

```php app/config/storage.php
return [

    /**
     * -------------------------------------------------------------------------
     *  Имя Bucket по умолчанию для хранилища
     * -------------------------------------------------------------------------
     *
     * Здесь вы можете указать, какой из buckets вы хотите использовать по
     * умолчанию для всей работы с хранилищами.
     *
     */

    'default' => env('STORAGE_SERVER', 'uploads'),

    /**
     * -------------------------------------------------------------------------
     *  Серверы хранилищ
     * -------------------------------------------------------------------------
     *
     * Здесь настроен каждый из серверов для вашего приложения. Конечно,
     * примеры настройки каждого доступного сервера, поддерживаемого
     * Spiral, показаны ниже для упрощения разработки.
     *
     */

    'servers' => [
        'static' => [
            'adapter' => 'local',
            'directory' => __DIR__ . '/../../runtime/static',
        ],

        's3' => [
            'adapter' => 's3', // or "s3-async"
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

    /**
     * -------------------------------------------------------------------------
     *  Buckets хранилищ
     * -------------------------------------------------------------------------
     *
     * Здесь список конкретных buckets (или хранилищ), которые используют
     * настройки серверов выше. Каждая секция "server" в этом списке должна
     * ссылаться на действующее имя сервера в списке выше.
     *
     * Список настроек в данном случае также является примером использования.
     * Вы можете свободно изменять количество buckets и тип настроек по
     * желанию.
     *
     */

    'buckets' => [
        'uploads' => [
            'server' => 'static',
            'prefix' => 'upload',
        ],

        'images' => [
            'server' => 'static',
            'prefix' => 'img',
        ],

        'videos' => [
            'server' => 's3',
        ],
    ],
];
```

> **Примечание**
> Обратите внимание, что эта конфигурация доступна только при использовании с Spiral Framework.

#### Ручная конфигурация (вне фреймворка)

Этот способ использования компонента требуется только в случае, если он установлен отдельно, вне фреймворка.

Сначала вам нужно создать экземпляр хранилища, где будут храниться все ваши buckets. После этого вы можете добавлять и
получать произвольный bucket из него по желаемому имени.

```php
$storage = new \Spiral\Storage\Storage();

$storage->add('example', \Spiral\Storage\Bucket::fromAdapter(
    new \League\Flysystem\Local\LocalFilesystemAdapter(__DIR__ . '/path/to/directory')
));

$file = $storage->bucket('example')
    ->write('file.txt', 'content');
```

Как вы могли заметить, вы можете использовать
существующие [адаптеры flysystem](https://flysystem.thephpleague.com/v2/docs/adapter/local/)
для создания bucket. Просто установите тот, который вам нужен, и добавьте его в хранилище с помощью метода `Bucket::fromAdapter()`.

#### Локальный сервер

Локальный сервер, как следует из названия, расположен в локальной файловой системе (в том же месте, где находится
исполняемый код вашего приложения).

Вы уже видели пример настроек локального сервера ранее, однако для их упрощения некоторые необязательные секции были
специально удалены. Теперь давайте рассмотрим полную конфигурацию этого типа сервера, оставив все
возможные секции конфигурации.

<details>
  <summary>Нажмите, чтобы показать пример конфигурации.</summary>

```php app/config/storage.php
return [
    'servers' => [
        'profiles' => [
            //
            // Имя типа сервера. Для локального сервера это значение должно быть
            // строковым значением "local".
            //
            'adapter' => 'local',

            //
            // Обязательный путь к локальной директории, где будут храниться
            // ваши файлы.
            //
            'directory'  => '/app/storage/user-profiles',

            //
            // Сопоставление видимости. Здесь вы можете установить видимость
            // по умолчанию для файлов и разрешения для файлов и директорий,
            // соответствующие определенному типу видимости.
            //
            // Значение видимости может быть только "private" или "public".
            //
            'visibility' => [
                'public'  => ['file' => 0644, 'dir' => 0755],
                'private' => ['file' => 0600, 'dir' => 0700],

                'default' => 'public',
            ],
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Связь с существующим локальным сервером. Обратите внимание, что все
            // дальнейшие опции bucket применимы только для этого (т.е. profiles)
            // типа сервера.
            //
            'server' => 'profiles',

            //
            // Для buckets, которые используют локальные серверы, вы можете
            // добавить префикс директории. В этом случае реальный физический
            // путь к файлу будет выглядеть как: "/app/storage/user-profiles/avatars",
            // где "/example/directory" строка - это физическая директория,
            // указанная в сервере, а "avatars" - префикс для директорий файлов,
            // указанный в bucket.
            //
            'prefix' => 'avatars',
        ]
    ]
];
```

</details>

Плагин RoadRunner предлагает [плагин Fileserver](https://roadrunner.dev/docs/http-static#file-server-plugin), который облегчает
обслуживание файлов из различных локальных хранилищ через использование URL-префиксов. Эта функция обеспечивает
детальный контроль над видимостью и доступом к конкретным файлам и директориям.

Например, вы можете сопоставить URL-префикс http://127.0.0.1:10101/avatars с директорией `/app/storage/user-profiles/avatars`
для предоставления публичного доступа к статическим ресурсам.

Вот пример конфигурации для плагина Fileserver:

```yaml .rr.yaml
fileserver:
  address: 127.0.0.1:10101
  calculate_etag: true
  weak: false
  stream_request_body: true
  serve:
    - prefix: "/avatars"
      root: "/app/storage/user-profiles/avatars"
```

#### S3 сервер

Этот тип сервера предназначен для взаимодействия с внешней распределенной файловой системой по протоколу S3. S3 — это
основной протокол для связи с серверами [Amazon](https://s3.console.aws.amazon.com/s3/home). Помимо самого Amazon
S3, существуют бесплатные альтернативы, которые вы можете установить и использовать на своем собственном сервере, например
[Minio Server](https://docs.min.io/).

Для настройки этого типа сервера вам понадобится существующий bucket и персональные данные аутентификации. Кроме того, для
полноты, пример ниже будет включать параметры, которые являются необязательными и имеют значения по умолчанию.

Обратите внимание, что для взаимодействия с этим типом серверов у вас должен быть установлен один из двух доступных
пакетов (любой). Вы должны установить пакет `league/flysystem-aws-s3-v3` или `league/flysystem-async-aws-s3` с помощью
Composer.

```terminal
composer require league/flysystem-aws-s3-v3 ^2.0
// ИЛИ
composer require league/flysystem-async-aws-s3 ^2.0
```

Во время конфигурации вы должны указать, какой из пакетов вы будете использовать в секции "adapter" сервера.
Значение `"s3"` соответствует пакету `league/flysystem-aws-s3-v3`, в то время как значение `"s3-async"` соответствует
пакету `league/flysystem-async-aws-s3`.

<details>
  <summary>Нажмите, чтобы показать пример конфигурации.</summary>

```php app/config/storage.php
return [
    'servers' => [
        'local' => [
            //
            // Имя типа сервера. Для S3 сервера это значение должно быть строковым
            // значением "s3" или "s3-async".
            //
            'adapter' => 's3',

            //
            // Обязательный строковый ключ региона S3, например "eu-north-1".
            //
            // Регион можно найти на странице "Amazon S3" здесь:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'region' => env('S3_REGION'),

            //
            // Необязательный ключ версии S3 API.
            //
            'version' => env('S3_VERSION', 'latest'),

            //
            // Обязательный ключ bucket S3.
            //
            // Имя bucket можно найти на странице "Amazon S3" здесь:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'bucket' => env('S3_BUCKET'),

            //
            // Обязательный ключ учетных данных S3, например "AAAABBBBCCCCDDDDEEEE".
            //
            // Ключ учетных данных можно найти на странице "Security Credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('S3_KEY'),

            //
            // Обязательный ключ приватного ключа учетных данных S3.
            // Это должно быть строковое значение приватного ключа или путь к файлу приватного ключа.
            //
            // Идентификатор также можно найти на странице "Security Credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'secret' => env('S3_SECRET'),

            //
            // Необязательный ключ токена учетных данных S3.
            //
            'token' => env('S3_TOKEN', null),

            //
            // Необязательный ключ времени истечения учетных данных S3.
            //
            'expires' => env('S3_EXPIRES', null),

            //
            // Необязательный ключ видимости файлов S3. Видимость "public"
            // по умолчанию.
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // Для buckets, которые используют S3 серверы, вы можете добавить
            // префикс директории.
            //
            'prefix' => '',

            //
            // Необязательный ключ URI конечной точки S3 API. Это значение требуется
            // при использовании сервера, отличного от Amazon.
            //
            'endpoint' => env('S3_ENDPOINT', null),

            //
            // Необязательные дополнительные опции S3.
            // Например, опция "use_path_style_endpoint" требуется для работы
            // с Minio S3 Server.
            //
            // Примечание: Эта секция "options" доступна начиная с framework >= 2.8.5
            // См. также https://github.com/spiral/framework/issues/416
            //
            'options' => [
                'use_path_style_endpoint' => true,
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Связь с существующим S3 сервером. Обратите внимание, что все
            // дальнейшие опции bucket применимы только для этого (т.е. s3 или s3-async)
            // типа сервера.
            //
            'server' => 's3',

            //
            // Значение видимости для конкретного типа bucket. Хотя вы можете
            // указать видимость для всего сервера, вы также можете переопределить
            // это значение для конкретного типа bucket.
            //
            'visibility' => env('S3_VISIBILITY', 'public'),

            //
            // В случае, если вы хотите использовать другой bucket, используя
            // основные настройки сервера, вы можете переопределить его, указав
            // соответствующий ключ конфигурации.
            //
            'bucket' => env('S3_BUCKET', null),

            //
            // Если новый bucket находится в другом регионе, вы также можете
            // переопределить это значение.
            //
            'region' => env('S3_REGION', null),

            //
            // Аналогичную вещь можно сделать с префиксом директории в случаях,
            // когда конкретный bucket должен ссылаться на какую-то другую
            // корневую директорию.
            //
            'prefix' => 'custom-directory',
        ]
    ]
];
```

</details>

#### Пользовательский сервер

В некоторых случаях стандартных адаптеров может быть недостаточно, и в этом случае вам может потребоваться указать свой
собственный. Вы также можете использовать свой файл конфигурации для настройки пользовательского адаптера.

В этом случае секция adapter должна ссылаться на реализацию `League\Flysystem\FilesystemAdapter`, а
секция "options" будет содержать массив аргументов, передаваемых в конструктор этого адаптера.

<details>
  <summary>Нажмите, чтобы показать пример конфигурации.</summary>

```php app/config/storage.php
return [
    'servers' => [
        'custom' => [
            //
            // Имя типа сервера, которое содержит имя класса адаптера.
            //
            'adapter' => \Custom\FlysystemAdapter::Class,

            //
            // Аргументы конструктора адаптера.
            //
            'options' => [
                // ...
            ]
        ],
    ],

    'buckets' => [
        'bucket' => [
            //
            // Связь с пользовательским сервером.
            //
            'server' => 'custom'
        ]
    ]
];
```

</details>

### Использование

Наконец, после того как мы ознакомились с тем, какие типы серверов поддерживает компонент, мы можем перейти к
их использованию.

Архитектура хранилища предполагает 3 различных уровня доступа к идентичным операциям: Storage, Bucket и File. На
каждом из этих уровней вы можете работать с файлами, но различия заключаются в том, сколько данных вам нужно передать
конкретному методу. На самом высоком уровне "Storage" вы должны передавать информацию о bucket и файле. На
уровне "Bucket" вы передаете информацию только о файле. Наконец, на уровне "File" вам не придется передавать никакой
дополнительной информации.

На практике это будет выглядеть так. Давайте попробуем создать файл `example.txt` тремя разными способами внутри
некоторого контроллера нашего приложения.

```php
use Spiral\Storage\StorageInterface;

class UploadController
{
    public function createFile(StorageInterface $storage): array
    {
        $result = [];
        
        // 1. Уровень Storage
        $result[] = $storage->create('bucket://example.txt');

        // 2. Уровень Bucket
        $result[] = $storage->bucket('bucket')
            ->create('example.txt');

        // 3. Уровень File
        $result[] = $storage->bucket('bucket')
            ->file('example.txt')
            ->create();

        return $result;
    }
}
```

Различия между методами заключаются в следующем:

- При работе с хранилищем вы должны передавать имя файла в URI-подобном формате `[BUCKET_NAME]://[FILE_NAME]`.
  Во всех методах работы с хранилищем это первый строковый аргумент.

- Во всех методах работы с bucket это имя файла в произвольном формате. Обратите также внимание, что ведущий
  слеш никак не влияет на расположение файла, и имена `file.txt` и `/file.txt` будут полностью
  идентичными.

- Во всех методах работы с конкретным файлом никаких дополнительных аргументов не требуется.

Каждый из методов имеет как свои преимущества, так и недостатки. Просто используйте то, что вам больше всего нравится.

Поскольку в примере выше мы используем контроллер, то в то же время мы будем использовать альтернативную зависимость,
которая подходит в случаях простого сохранения файлов, когда не требуется много реализаций buckets.

В таких случаях, как показано ниже, будет выбран bucket, используемый в системе по умолчанию. Вы можете определить bucket в
секции `'default'` вашей конфигурации.

```php
use Spiral\Storage\BucketInterface;
use Psr\Http\Message\ServerRequestInterface;

class UploadingController
{
    public function upload(ServerRequestInterface $request, BucketInterface $bucket): string
    {
        /** @var \Psr\Http\Message\UploadedFileInterface $file */
        foreach ($request->getUploadedFiles() as $i => $file) {
            $bucket->write("file-{$i}.txt", $file->getStream());
        }

        return \count($request->getUploadedFiles()) . ' files uploaded';
    }
}
```

С каждого из уровней вы можете обратиться к дочернему.

**Из Storage:**

- `$storage->bucket('[bucket-name]'): BucketInterface`
- `$storage->file('[bucket-name]://[file-name]'): FileInterface`

**Из Bucket:**

- `$bucket->file('[file-name]'): FileInterface`

После того как мы познакомились с вариантами использования одних и тех же методов на разных уровнях хранилища, мы
можем перейти к описанию доступных возможностей. Начнем!

#### Создание и запись

Если вы хотите создать файл, вы можете использовать один из двух доступных методов: `create()` или `write()`. Первый создает
пустой файл там, где он не создан, а второй позволяет дополнительно записать туда произвольный `string`
или `resource` поток содержимого.

Например, код, который создает файл, может выглядеть так.

```php
// Создание файла из bucket
$file = $bucket->create('file.txt');

// Создание файла из bucket со строковым содержимым
$file = $bucket->write('file.txt', 'message');

// Создание файла из bucket с содержимым потока ресурса
$file = $bucket->write('file.txt', fopen(__DIR__ . '/local/file.txt', 'rb+'));
```

#### Копирование и перемещение

Для копирования файлов используйте метод `copy()`, который содержит один обязательный аргумент с именем нового файла и один
необязательный - bucket, куда файл должен быть скопирован. Если второй аргумент не указан, bucket копирования
будет идентичен оригинальному.

Метод `move()` полностью аналогичен использованию метода `copy()`, но вместо копирования он перемещает файл.

```php
$backup = $firstBucket->copy('from.txt', 'backup.txt');

$moved  = $firstBucket->move('backup.txt', 'to.txt', $secondBucket);
```

> Обратите внимание, что когда файл перемещается (или копируется) из bucket с приватными разрешениями в распределение с публичными
> разрешениями, видимость файла также изменится на ту, которая соответствует этому bucket.

#### Удаление

Для удаления файла просто используйте метод `delete()`. Этот метод принимает необязательный булевый аргумент, который означает удаление
пустой директории, где находился файл.

```php
$bucket->delete('file.txt');
```

#### Чтение содержимого

Существует два способа чтения содержимого существующего файла: использование методов `getContents()` и `getStream()`. Первый
возвращает строковое содержимое файла, а второй ресурс - это поток для работы с потоковыми данными.

```php
$string = $bucket->getContents('text.txt');

$resource = $bucket->getStream('music.mp3');
```

#### Проверка существования

Для проверки существования файла используйте метод `exists()`, который возвращает булево значение `true`, если файл существует
в bucket, или `false`, если он не существует.

```php
$isExists = $bucket->exists('file.txt');
```

#### Размер файла

Для получения информации о размере файла используйте метод `getSize()`, который возвращает размер существующего файла в
байтах.

```php
$bytes = $bucket->getSize('file.txt');
```

#### Время последней модификации

Для получения информации о дате последней модификации файла используйте метод `getLastModified()`, который
возвращает время в формате UNIX timestamp.

```php
$timestamp = $bucket->getLastModified('file.txt');
```

#### Mime Type

Для получения mime типа файла используйте метод `getMimeType()`, который возвращает строковое представление mime типа файла.

```php
$mime = $bucket->getMimeType('file.txt');
```

#### Видимость

В дополнение к таким характеристикам, как существование файла, есть также видимость файла. Вы можете получить
информацию о видимости файла, используя метод `getVisibility()`, и обновить видимость, используя метод
`setVisibility()`.

Эти методы работают с константами, которые были определены в enum-подобном интерфейсе `Spiral\Storage\Visibility`.
Так, пример кода с контролем видимости файла будет выглядеть следующим образом.

```php
$visibility = $bucket->getVisibility('file.txt');

// Если файл "private", то опубликуем его
if ($visibility === Visibility::VISIBILITY_PRIVATE) {
    $bucket->setVisibility('file.txt', Visibility::VISIBILITY_PUBLIC);
}
```

> **Примечание**
> В случае использования bucket, расположенного на ОС Windows, эта функциональность может не работать.

#### Публикация URI

Компонент хранилища изначально предоставляет только операции для работы с файлами на произвольных файловых системах. Однако
некоторые системы, помимо хранения, позволяют HTTP организовать точку доступа к таким файлам для получения их содержимого
через браузер.

Компонент хранилища позволяет добавить произвольный URI resolver публичных адресов для этих файлов с помощью
[компонента distribution](../component/distribution.md).

Для настройки resolver вам следует ознакомиться с конфигурацией этого компонента. После этого для конкретного bucket
просто добавьте секцию, содержащую ссылку на конкретный distribution resolver.

Каждый bucket способен указать distribution.

```php
return [
    // ...
    'buckets' => [
        'uploads' => [
            'server' => '...',

            //
            // + Добавить связь с существующим distribution.
            //
            'distribution' => 'NAME_OF_DISTRIBUTION'
        ],
    ],
];
```

Для использования этого URI resolver просто вызовите метод `toUri()`. В случае, если вам нужен какой-либо другой distribution, вы можете указать его
явно в методе `toUriFrom(...)`.

> **Примечание**
> В отличие от других методов, этот можно вызывать только для конкретного файла.

```php
use Spiral\Storage\BucketInterface;
use Spiral\Distribution\UriResolverInterface;

class UriController
{
    // Использование URI resolver по умолчанию
    public function getUri(BucketInterface $bucket): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUri();
    }

    // Использование другого URI resolver
    public function getAnotherUri(BucketInterface $bucket, UriResolverInterface $resolver): string
    {
        return (string)$bucket
            ->file('picture.jpg')
            ->toUriFrom($resolver);
    }
}
```

Некоторые генераторы могут принимать дополнительные опции. Такие аргументы можно передать в методы `toUri([...$arguments])`
или `toUriFrom($resolver, [...$arguments])`. Например, если вы создаете ссылку на CloudFront, вы можете
дополнительно указать время истечения этой ссылки.

> **Узнать больше**
> Вы можете прочитать больше о возможных дополнительных аргументах на соответствующей странице секции distribution.

```php
$uri = $file->toUri(new \DateInterval('PT30S'));
// И
$uri = $file->toUriFrom($resolver, new \DateInterval('PT30S'));
```


## Distribution

Компонент `spiral/distribution` отвечает за предоставление публичных HTTP-ссылок на произвольные ресурсы. В большинстве
случаев это будет тот же адрес, что и адрес самого сайта, однако в некоторых случаях ресурсы могут быть расположены
на внешних серверах, таких как [Amazon CloudFront](https://aws.amazon.com/cloudfront/) или какой-либо другой CDN. В этих случаях
генерация публичной ссылки на ресурс требует использования специфического API провайдера или написания собственного кода для
используемого CDN. Компонент упрощает это взаимодействие и предоставляет ряд встроенных драйверов для генерации URI к
внешним поставщикам.

### Установка

Используйте Composer для установки компонента:

```terminal
composer require spiral/distribution
```

Для включения компонента вам просто нужно добавить класс `Spiral\Distribution\Bootloader\DistributionBootloader` в
список загрузчиков (Bootloader):

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Distribution\Bootloader\DistributionBootloader::class,
        // ...
    ];
}
```

Узнайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Distribution\Bootloader\DistributionBootloader::class,
    // ...
];
```

Узнайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

### Конфигурация

Файл конфигурации distribution по умолчанию расположен в `app/config/distribution.php`.

```php app/config/distribution.php
<?php

return [

    /**
     * -------------------------------------------------------------------------
     *  Имя Distribution Resolver по умолчанию
     * -------------------------------------------------------------------------
     *
     * Здесь вы можете указать, какой из resolvers вы хотите использовать по
     * умолчанию для всей работы с генерацией URI. Конечно, вы можете использовать
     * несколько resolvers одновременно, используя библиотеку distribution.
     *
     */

    'default' => env('DISTRIBUTION_RESOLVER', 'local'),

    /**
     * -------------------------------------------------------------------------
     *  Distribution Resolvers
     * -------------------------------------------------------------------------
     *
     * Здесь настроен каждый из resolvers для вашего приложения.
     * Конечно, примеры настройки каждого доступного distribution, поддерживаемого
     * Spiral, показаны ниже для упрощения разработки.
     *
     */

    'resolvers' => [
        'local' => [
            'type' => 'static',
            'uri'  => env('APP_URL', 'http://localhost')
        ],

        'cloudfront' => [
            'type' => 'cloudfront',
            'key' => env('AWS_CF_KEY'),
            'domain' => env('AWS_CF_KEY'),
            'private' => env('AWS_CF_PRIVATE_KEY'),
        ],

        's3' => [
            'type' => 's3',
            'region' => env('S3_REGION'),
            'bucket' => env('S3_BUCKET'),
            'key' => env('S3_KEY'),
            'secret' => env('S3_SECRET'),
        ],
    ],

];
```

> **Предупреждение**
> Приведенный пример конфигурации специфичен для Spiral Framework и не будет применим вне этого
> контекста. Опции конфигурации и структура будут отличаться в зависимости от конкретного фреймворка или используемой системы.

#### Ручная конфигурация (вне фреймворка)

Этот способ использования компонента требуется только в случае, если он установлен отдельно, вне фреймворка.

Сначала вам нужно создать экземпляр менеджера, где будут храниться все ваши uri resolvers. После этого вы можете добавлять и
получать произвольные resolvers из него по желаемому имени.

```php
$manager = new \Spiral\Distribution\Manager();

$manager->add('resolver-name', new CustomResolver());

$manager->resolver('resolver-name'); // object(CustomResolver)
```

После этого вы можете добавить туда либо свои собственные менеджеры, либо предоставленные компонентом, такие как, например, "static".

```php
use Nyholm\Psr7\Uri;
use Spiral\Distribution\Manager;
use Spiral\Distribution\Resolver\StaticResolver;

$manager = new Manager();
$manager->add('local', new StaticResolver(new Uri('https://static.example.com')));
```

### Использование

После того как вы настроили ваш компонент, вы можете начать его использовать.

Если вы используете приложение Spiral, менеджер уже настроен. Вы можете получить его из
[контейнера](../container/overview.md) или через [внедрение зависимостей](../container/overview.md#dependency-injection).

```php
use Spiral\Distribution\DistributionInterface;

class FilesController
{
    public function showImage(DistributionInterface $dist): string
    {
        $resolver = $dist->resolver('local');

        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

Если вам нужен resolver по умолчанию, определенный в секции конфигурации "default", вам не нужно
получать весь экземпляр менеджера. Вы можете получить нужный resolver из контейнера сразу.

```php
use Spiral\Distribution\UriResolverInterface;

class FilesController
{
    public function showImage(UriResolverInterface $resolver): string
    {
        return (string)$resolver->resolve('example/image.jpg');
    }
}
```

Вы могли заметить, что после получения resolver в примерах выше используется метод `resolve()` с
относительным путем к файлу. Он принимает строковое значение в качестве аргумента и возвращает реализацию
[PSR-7 `Psr\Http\Message\UriInterface`](https://www.php-fig.org/psr/psr-7/).

```php
$uri = $resolver->resolve('path/to/file.txt');
//
// Ожидается:
//  object(Psr\Http\Message\UriInterface)
//
```

> **Примечание**
> Некоторые resolvers поддерживают дополнительные опции при получении ссылки,
> например: `$cloudfront->resolve('path/to/file.txt', expiration: new \DateInterval('PT60S'));`

#### Static URI Resolver

Этот тип resolver генерирует адрес к ресурсу, просто добавляя переданную ссылку файла в конец URI,
указанного в конфигурации resolver.

Для настройки этого типа resolver вам нужно указать только два обязательных поля.

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        'local' => [
            //
            // Обязательный ключ типа resolver.
            // Для static resolver он должен содержать строковое значение "static".
            //
            'type' => 'static',

            //
            // Обязательный ключ URL статического сервера.
            //
            'uri'  => env('APP_URL', 'http://localhost')
        ],
    ]
];
```

В отличие от аналогичного метода, используемого для генерации адреса страницы в компоненте роутера [генератор url](../http/routing.md#url-generation),
ссылки могут быть произвольными и настроенными на отдельном сервере, предназначенном для обслуживания статического контента.

Таким образом, если вы передаете произвольную строку файла в метод `resolve()`, вы получите физическую http-ссылку на этот
файл. Если базовый uri определен как "`http://localhost`", результат будет следующим:

```php
/** @var \Spiral\Distribution\Resolver\StaticResolver $resolver */
$resolver = $manager->resolver('local');

echo $resolver->resolve('path/to/file.txt');
//
// Ожидается:
//  string(33) "http://localhost/path/to/file.txt"
//
```

#### CloudFront URI Resolver

CloudFront — это популярный сервис статического распространения, используемый в сочетании с сервисами Amazon. Для его использования вы должны
установить пакет `aws/aws-sdk-php` с помощью Composer.

```terminal
composer require aws/aws-sdk-php ^3.0
```

После регистрации и создания вашего статического сервера в сервисах AWS
[вы получите](https://console.aws.amazon.com/cloudfront/home) параметры для настройки. Кроме того,
вам понадобится "файл приватного ключа" и "access key id", которые вы можете найти на вкладке "CloudFront key pairs"
на странице "[Security Credentials](https://console.aws.amazon.com/iam/home#/security_credentials)".

Для настройки этого resolver просто укажите параметры подключения в секциях конфигурации:

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        'cloudfront' => [
            //
            // Обязательный ключ типа resolver.
            // Для CloudFront он должен содержать строковое значение "cloudfront".
            //
            'type' => 'cloudfront',

            //
            // Обязательный ключ CloudFront access key id.
            // Он должен содержать строковое значение вроде "AAAABBBBCCCCDDDDEEEE".
            //
            // Идентификатор можно найти на вашей персональной странице "security credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('AWS_CF_KEY'),
            
            //
            // Обязательный ключ CloudFront приватного ключа.
            // Это должно быть строковое значение приватного ключа или путь к файлу приватного ключа.
            //
            // Идентификатор также можно найти на странице "Security Credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            // Обратите внимание, что вы можете скачать файл приватного ключа только
            // во время его генерации!
            //
            'private' => env('AWS_CF_PRIVATE_KEY'),

            //
            // Обязательный ключ доменного имени CloudFront.
            // Он должен содержать строковое значение вроде "example.cloudfront.net".
            //
            // Домен можно найти на странице "CloudFront Distributions" здесь:
            //  - https://console.aws.amazon.com/cloudfront/home
            //
            'domain' => env('AWS_CF_DOMAIN'),
            
            //
            // Необязательный ключ префиксов файлов CloudFront.
            // Он должен содержать строку вроде "path/to/directory". В этом случае
            // этот префикс будет добавлен для каждого файла при генерации url.
            //
            'prefix' => env('AWS_CF_PREFIX'),
        ],
    ]
];
```

Если вы решите создать resolver самостоятельно, вы можете использовать те же настройки, переданные в конструктор
resolver, используемого для работы с сервисом CloudFront.

```php
//
// Использование именованных аргументов PHP 8 в конструкторе для ясности
//
$cloudfront = new \Spiral\Distribution\Resolver\CloudFrontResolver(
    keyPairId: 'AAAABBBBCCCCDDDDEEEE',
    privateKey: \file_get_contents(__DIR__ . '/path/to/key.pem'),
    domain: 'example.cloudfront.net',
    prefix: 'path/to/files'
);

$url = $cloudfront->resolve(...);
```

CloudFront resolver получает в качестве первого аргумента метода `resolve()` ссылку на файл, для которого должен быть сгенерирован публичный
адрес, и в качестве второго, необязательного, время жизни (истечения) этой ссылки.

Время истечения может быть указано в нескольких форматах. Это может быть:

- Встроенный PHP объект [`\DateInterval`](https://www.php.net/manual/en/class.dateinterval.php).
- Экземпляр интерфейса [`\DateTimeInterface`](https://www.php.net/manual/en/class.datetimeinterface.php).
  В этом случае интервал истечения считается с момента генерации ссылки.
- Строковое значение в [формате длительности времени PHP](https://www.php.net/manual/en/dateinterval.construct.php).
- Целочисленное значение в секундах.

Ниже приведены примеры каждого из допустимых форматов:

```php
$file = 'path/to/file.txt';

// Объект DateInterval
$url = $cloudfront->resolve($file, new DateInterval('PT30S'));

// Экземпляр DateTimeInterface
$url = $cloudfront->resolve($file, new DateTime('+30 sec'));

// Длительность в строковом формате
$url = $cloudfront->resolve($file, 'PT30S');

// Длительность в int формате
$url = $cloudfront->resolve($file, 30);
```

В случае любых особых обстоятельств вы можете заменить текущий генератор времени и парсер истечения. Кроме того,
вы также можете установить значение по умолчанию для всех генерируемых ссылок в рамках данного resolver.

```php
$cloudfront = (new \Spiral\Distribution\Resolver\CloudFrontResolver(...))
    //
    // С пользовательским генератором "текущего времени".
    //
    // Генератор времени должен быть реализацией интерфейса
    // \Spiral\Distribution\Internal\DateTimeFactoryInterface.
    //
    ->withDateTimeFactory(new CustomCurrentDateGenerator())

    //
    // С пользовательским парсером "времени истечения".
    //
    // Парсер формата "времени истечения" должен быть реализацией интерфейса
    // \Spiral\Distribution\Internal\DateTimeIntervalFactoryInterface.
    //
    ->withDateTimeIntervalFactory(new CustomExpirationParser())

    //
    // Со значением "времени истечения" по умолчанию.
    //
    // Значение должно быть корректным для времени, указанного в
    // парсере данного URI resolver.
    //
    ->withExpirationDate('PT30S');
```

#### S3 URI Resolver

Если по какой-то причине вы не можете использовать CloudFront resolver (например, в случае использования
[Minio Server](https://docs.min.io/)), вы можете использовать resolver, который генерирует ссылки на S3 сервер. Для его использования вы
также должны установить пакет `aws/aws-sdk-php` с помощью Composer.

```terminal
composer require aws/aws-sdk-php ^3.0
```

Для использования с AWS S3 вам нужны учетные данные аккаунта и рабочий bucket, который вы можете
создать [на странице "Amazon S3"](https://s3.console.aws.amazon.com/s3/home). После создания bucket вам нужно будет
заполнить следующие параметры конфигурации.

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        's3' => [
            //
            // Обязательный ключ типа resolver.
            // Для S3 он должен содержать строковое значение "s3".
            //
            'type' => 's3',

            //
            // Обязательный строковый ключ региона S3 вроде "eu-north-1".
            //
            // Регион можно найти на странице "Amazon S3" здесь:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'region' => env('S3_REGION'),

            //
            // Необязательный ключ версии S3 API.
            //
            'version' => env('S3_VERSION', 'latest'),

            //
            // Обязательный ключ bucket S3.
            //
            // Имя bucket можно найти на странице "Amazon S3" здесь:
            //  - https://s3.console.aws.amazon.com/s3/home
            //
            'bucket' => env('S3_BUCKET'),

            //
            // Обязательный ключ учетных данных S3 вроде "AAAABBBBCCCCDDDDEEEE".
            //
            // Ключ учетных данных можно найти на странице "Security Credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'key' => env('S3_KEY'),

            //
            // Обязательный ключ приватного ключа учетных данных S3.
            // Это должно быть строковое значение приватного ключа или путь к файлу приватного ключа.
            //
            // Идентификатор также можно найти на странице "Security Credentials" здесь:
            //  - https://console.aws.amazon.com/iam/home#/security_credentials
            //
            'secret' => env('S3_SECRET'),

            //
            // Необязательный ключ токена учетных данных S3.
            //
            'token' => env('S3_TOKEN', null),

            //
            // Необязательный ключ времени истечения учетных данных S3.
            //
            'expires' => env('S3_EXPIRES', null),

            //
            // Необязательный ключ URI конечной точки S3 API.
            //
            'endpoint' => env('S3_ENDPOINT', null),
            
            //
            // Необязательный ключ префиксов файлов S3 API.
            // Он должен содержать строку вроде "path/to/directory".
            //
            // В этом случае этот префикс будет добавлен для каждого файла при
            // генерации url.
            //
            'prefix' => env('S3_PREFIX'),

            //
            // Необязательные дополнительные опции S3.
            // Например, опция "use_path_style_endpoint" требуется для работы
            // с Minio S3 Server.
            //
            // Примечание: Эта секция "options" доступна начиная с framework >= 2.8.6
            //
            'options' => [
                'use_path_style_endpoint' => true,
            ]
        ],
    ]
];
```

Если вы решите создать resolver самостоятельно, вы можете использовать те же настройки, переданные в конструктор
resolver, используемого для работы с сервисом S3.

```php
//
// Использование именованных аргументов PHP 8 в конструкторе для ясности
//
$s3 = new \Spiral\Distribution\Resolver\S3SignedResolver(
    client: new \Aws\S3\S3Client([
        'version' => 'latest',
        'region'  => 'eu-north-1',
        'credentials' => new \Aws\Credentials\Credentials(
            key: 'key',
            secret: file_get_contents(__DIR__ . '/path/to/secret.pem')
        )
    ]),
    bucket: 'bucket-name',
    prefix: 'path/to/files'
);

$url = $s3->resolve(...);
```

После регистрации resolver вы сможете создать URI к файлу, используя метод `resolve()`. По аналогии с
реализацией CloudFront, вы также можете передать второй аргумент `expiration` в этот метод, который означает время жизни
сгенерированного URI.

```php
$url = $s3->resolve($file, new DateTime('+30 sec'));
```

Все аналогичные методы для указания глобального истечения URI, генератора "текущего времени" и парсеров "времени истечения"
также доступны.

#### Пользовательский URI Resolver

В некоторых случаях вы можете найти задачи для генерации URI, которые не подходят к существующим реализациям resolvers. В
этом случае вы можете зарегистрировать свой собственный класс resolver в конфигурации. Для передачи дополнительных аргументов в конструктор
этого resolver просто укажите секцию `options` в файле конфигурации.

```php app/config/distribution.php
return [
    // ...
    'resolvers' => [
        // ...
        'custom' => [
            //
            // Обязательный ключ класса resolver. Этот класс должен реализовывать
            // интерфейс \Spiral\Distribution\UriResolverInterface.
            //
            'type' => \Example\CustomResolver::class,

            //
            // Необязательная секция массива "options".
            //
            'options' => [
                // список аргументов конструктора...
            ],
        ],
    ]
];
```

В некоторых случаях этот метод регистрации может вам не подойти. Если в параметрах конструктора требуются какие-либо зависимости из контейнера, вам следует использовать [загрузчик (Bootloader)](../framework/bootloaders.md).
