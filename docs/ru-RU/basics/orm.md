# Основы — База данных и ORM

Для использования функциональности ORM и баз данных в вашем приложении, Spiral предлагает
компонент [spiral/cycle-bridge](https://github.com/spiral/cycle-bridge).

## Установка

Этот компонент автоматически включен в `spiral/app` и также может быть установлен в существующие проекты с помощью
Composer, выполнив следующую команду:

```terminal
composer require spiral/cycle-bridge
```

После успешной установки пакета необходимо добавить `Spiral\Cycle\Bootloader\BridgeBootloader`
загрузчик (Bootloader) в Kernel:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\BridgeBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cycle\Bootloader\BridgeBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

В качестве альтернативы, для более детального контроля, `BridgeBootloader` может быть исключен и выбраны только
необходимые загрузчики (Bootloader).

Соответствующий код для этого в Kernel будет выглядеть следующим образом:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

public function defineBootloaders(): array
{
    return [
        // ...
    
        // База данных
        CycleBridge\DatabaseBootloader::class,
        CycleBridge\MigrationsBootloader::class,
    
        // Автоматическое закрытие соединения с базой данных после каждого запроса (Опционально)
        // CycleBridge\DisconnectsBootloader::class,
    
        // ORM
        CycleBridge\SchemaBootloader::class,
        CycleBridge\CycleOrmBootloader::class,
        CycleBridge\AnnotatedBootloader::class,
        CycleBridge\CommandBootloader::class,
    
        // Валидация (Опционально)
        // CycleBridge\ValidationBootloader::class,
    
        // DataGrid (Опционально)
        // CycleBridge\DataGridBootloader::class,
    
        // Хранилище токенов в базе данных (Опционально)
        CycleBridge\AuthTokensBootloader::class,
    
        // Миграции и скаффолдеры Cycle (Опционально)
        CycleBridge\ScaffolderBootloader::class,
        
        // Прототипирование (Опционально)
        CycleBridge\PrototypeBootloader::class,
    ];
}
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
use Spiral\Cycle\Bootloader as CycleBridge;

protected const LOAD = [
    // ...

    // База данных
    CycleBridge\DatabaseBootloader::class,
    CycleBridge\MigrationsBootloader::class,

    // Автоматическое закрытие соединения с базой данных после каждого запроса (Опционально)
    // CycleBridge\DisconnectsBootloader::class,

    // ORM
    CycleBridge\SchemaBootloader::class,
    CycleBridge\CycleOrmBootloader::class,
    CycleBridge\AnnotatedBootloader::class,
    CycleBridge\CommandBootloader::class,

    // Валидация (Опционально)
    // CycleBridge\ValidationBootloader::class,

    // DataGrid (Опционально)
    // CycleBridge\DataGridBootloader::class,

    // Хранилище токенов в базе данных (Опционально)
    CycleBridge\AuthTokensBootloader::class,

    // Миграции и скаффолдеры Cycle (Опционально)
    CycleBridge\ScaffolderBootloader::class,
    
    // Прототипирование (Опционально)
    CycleBridge\PrototypeBootloader::class,
];
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

#### Загрузчик (Bootloader) разрывов соединений

Загрузчик (Bootloader) служит для автоматического закрытия соединения с базой данных после каждого запроса в
долго работающих приложениях. Это опциональный загрузчик (Bootloader), который может быть включен или исключен в
зависимости от требований конкретного приложения.

## Конфигурация

### База данных

Конфигурация для сервисов базы данных находится в файле конфигурации `app/config/database.php`. В этом файле вы
можете определить все подключения к базам данных, а также указать, какое подключение должно использоваться по
умолчанию. Большинство опций конфигурации в этом файле определяются значениями переменных окружения вашего приложения.

Вот пример файла конфигурации, который определяет подключение к базе данных:

```php app/config/database.php
use Cycle\Database\Config;

return [
    'logger' => [
        'default' => env('DB_LOGGER_DEFAULT'),
        'drivers' => [
            // 'runtime' => 'stdout'
        ],
    ],

    'default' => env('DB_DEFAULT', 'default'),

    /**
     * Модуль Spiral/Database предоставляет поддержку для управления несколькими базами данных
     * в одном приложении, использования соединений для чтения/записи и логического разделения
     * нескольких баз данных в рамках одного соединения с использованием префиксов.
     *
     * Чтобы зарегистрировать новую базу данных, просто добавьте новую
     * в раздел "databases" ниже.
     */
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],

    /**
     * Каждый экземпляр базы данных должен иметь связанный объект соединения.
     * Соединения используются для предоставления низкоуровневой функциональности и обертывания различных
     * драйверов баз данных. Чтобы зарегистрировать новое соединение, вы должны указать
     * класс драйвера и его опции соединения.
     */
    'drivers' => [
        'runtime' => new Config\MySQLDriverConfig(
            connection: new Config\MySQL\TcpConnectionConfig(
                database: 'homestead',
                host: '127.0.0.1',
                port: 3307,
                user: 'root',
                password: 'secret',
            ),
            queryCache: true
        ),
        // ...
    ],
];
```

> **Предупреждение**
> Обратите внимание, что использование SQLite в качестве серверной части базы данных с несколькими воркерами может
> представлять значительные проблемы и ограничения. Важно знать об этих ограничениях, чтобы обеспечить правильную
> функциональность и производительность вашего приложения. Читайте больше об этом в разделе
> [ограничения SQLite](./#sqlite-limitations).

> **Смотрите больше**
> Читайте больше о конфигурации базы данных в
> [Database - Installation and Configuration](https://cycle-orm.dev/docs/database-configuration) на официальном
> сайте.

### ORM

Конфигурация для сервисов ORM фреймворка Spiral находится в файле `app/config/cycle.php` вашего приложения

```php app/config/cycle.php
use Cycle\ORM\SchemaInterface;

return [
    'schema' => [
        /**
         * true (По умолчанию) - Схема будет сохранена в кэше после компиляции.
         * Она не изменится после модификации сущности. Используйте `php app.php cycle` для обновления схемы.
         *
         * false - Схема не будет сохранена в кэше после компиляции.
         * Она будет автоматически изменена после модификации сущности. (Режим разработки)
         */
        'cache' => false,

        /**
         * CycleORM предоставляет возможность управлять настройками по умолчанию для
         * каждой схемы с неопределенными сегментами
         */
        'defaults' => [
            SchemaInterface::MAPPER => \Cycle\ORM\Mapper\Mapper::class,
            SchemaInterface::REPOSITORY => \Cycle\ORM\Select\Repository::class,
            SchemaInterface::SCOPE => null,
            SchemaInterface::TYPECAST_HANDLER => [
                \Cycle\ORM\Parser\Typecast::class
            ],
        ],

        'collections' => [
            'default' => 'array',
            'factories' => [
                'array' => new \Cycle\ORM\Collection\ArrayCollectionFactory(),
                // 'doctrine' => new \Cycle\ORM\Collection\DoctrineCollectionFactory(),
                // 'illuminate' => new \Cycle\ORM\Collection\IlluminateCollectionFactory(),
            ],
        ],

        /**
         * Генераторы схем (Опционально)
         * null (по умолчанию) - Будут использоваться генераторы схем, определенные в загрузчиках (Bootloader)
         */
        'generators' => null,

        // 'generators' => [
        //        \Cycle\Schema\Generator\ResetTables::class,
        //        \Cycle\Annotated\Embeddings::class,
        //        \Cycle\Annotated\Entities::class,
        //        \Cycle\Annotated\TableInheritance::class,
        //        \Cycle\Annotated\MergeColumns::class,
        //        \Cycle\Schema\Generator\GenerateRelations::class,
        //        \Cycle\Schema\Generator\GenerateModifiers::class,
        //        \Cycle\Schema\Generator\ValidateEntities::class,
        //        \Cycle\Schema\Generator\RenderTables::class,
        //        \Cycle\Schema\Generator\RenderRelations::class,
        //        \Cycle\Schema\Generator\RenderModifiers::class,
        //        \Cycle\Annotated\MergeIndexes::class,
        //        \Cycle\Schema\Generator\GenerateTypecast::class,
        // ],
    ],

    /**
     * Подготовка всех внутренних сервисов ORM (мапперы, репозитории, приведение типов...)
     */
    'warmup' => false,
];
```

## Cycle ORM

CycleORM — это мощный и гибкий инструмент объектно-реляционного отображения (ORM) для PHP, который позволяет
разработчикам взаимодействовать с базами данных объектно-ориентированным способом. Он предоставляет множество функций,
которые упрощают работу с данными, включая гибкие опции конфигурации, мощный конструктор запросов и поддержку
динамического отображения схем.

Он поддерживает различные популярные реляционные базы данных, такие как MySQL, MariaDB, PostgresSQL, SQLServer и
SQLite.

> **Примечание**
> Полная документация доступна на официальном сайте [CycleORM](https://cycle-orm.dev/docs).

### Экземпляр ORM

Вы можете получить доступ к экземпляру ORM из контейнера, используя интерфейс `Cycle\ORM\ORMInterface`.

### Репозитории

Представим, что у нас есть сущность `User`

```php
use Cycle\Annotated\Annotation as Cycle;

#[Cycle\Entity(repository: UserRepository::class)]
class User
{
    // ...
}
```

с репозиторием `UserRepository`.

```php 
class UserRepository extends \Cycle\ORM\Select\Repository
{
    public function findByEmail(string $email): ?User
    {
        return $this->findOne(['email' => $email]);
    }
}
```

Вы можете запросить репозиторий из экземпляра ORM самостоятельно, предоставив имя сущности или роли.

```php
use Cycle\ORM\ORMInterface;
use Cycle\ORM\RepositoryInterface;

class UserService
{   
    private readonly RepositoryInterface $repository;

    public function __construct(
        Cycle\ORM\ORMInterface $orm
    ) {
        $this->repository = $orm->getRepository(User::class);
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findOne(['email' => $email]);
        // ...
    }
}
```

Вы также можете запросить репозиторий из контейнера. Фреймворк использует [IoC инъекции](../container/injectors.md)
для внедрения репозиториев в ваш код, которые реализуют `Cycle\ORM\RepositoryInterface`.

```php
class UserService
{
    public function __construct(
        private readonly UserRepository $repository
    ) {
    }
    
    public function getProfile(string $email): User
    {
        $user = $this->repository->findByEmail($email);
        // ...
    }
}
```

Когда вы запрашиваете репозиторий из контейнера, Spiral автоматически запросит репозиторий из ORM и свяжет его с
правильной сущностью.

### Транзакции

Для сохранения изменений сущностей вашим сервисам приложения и контроллерам потребуется
`Cycle\ORM\EntityManagerInterface`.

По умолчанию фреймворк автоматически создаст транзакцию по требованию из контейнера. Учитывая, что транзакции всегда
очищаются после выполнения операции `run`, вы можете запросить ее как параметр конструктора.

> **Смотрите больше**
> Вы можете прочитать больше о транзакциях в
> [документации CycleORM](https://cycle-orm.dev/docs/advanced-entity-manager).

Вот пример сервиса, который использует `EntityManagerInterface`:

```php
use Cycle\ORM\EntityManagerInterface;

class UserService
{
    public function __construct(
        private readonly EntityManagerInterface $entityManager
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        $user = new User($name, $email);
        
        $this->entityManager->persist($user);
        $this->entityManager->run();
        
        return $user;
    }
}
```

> **Примечание:**
> Убедитесь, что методы `persist/delete` и `run` всегда вызываются в рамках одной области видимости метода при
> использовании транзакций, специфичных для сервиса.

### Валидация сущностей

Мост Cycle предоставляет загрузчик (Bootloader) CycleBridge\ValidationBootloader, который регистрирует дополнительные
проверки для пакета [spiral/validator](../validation/spiral.md). Этот загрузчик (Bootloader) включает два
дополнительных правила валидации, которые расширяют функциональность валидатора и позволяют более эффективную и
действенную валидацию данных в приложении.

#### exists

Проверка существования сущности с заданной ролью и первичным ключом.

По умолчанию правило проверит существование сущности по первичному ключу.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    #[Setter(filter: 'intval')]
    public int $id;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class // Роль сущности
                ] 
            ]       
        ]);
    }
}
```

Вы также можете указать имя поля и значение, которое будет использоваться для проверки существования сущности.

```php
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class UpdateUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::exists', 
                    \App\Entity\User::class, // Роль сущности
                    'username', // Имя поля
                ], 
            ],       
        ]);
    }
}
```

#### unique

Проверка уникальности сущности с заданной ролью.

```php
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Setter;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;

final class StoreUser extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $username;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'username' => [
                [
                    'entity::unique', 
                    \App\Entity\User::class, // Роль сущности
                    'username', // Имя поля
                ] 
            ]       
        ]);
    }
}
```

### Поведения сущностей

Если вы хотите использовать пакет [cycle/entity-behavior](https://cycle-orm.dev/docs/entity-behaviors-install/) в
вашем приложении, вам нужно сначала установить его:

```bash
composer require cycle/entity-behavior
```

После этого вам нужно связать `Cycle\ORM\Transaction\CommandGeneratorInterface`
с `\Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator` в контейнере приложения:

```php app/src/Application/Bootloader/EntityBehaviorBootloader.php
namespace App\Application\Bootloader;

use Cycle\ORM\Transaction\CommandGeneratorInterface;
use Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator;
use Spiral\Boot\Bootloader\Bootloader;

final class EntityBehaviorBootloader extends Bootloader
{
    protected const BINDINGS = [
        CommandGeneratorInterface::class => \Cycle\ORM\Entity\Behavior\EventDrivenCommandGenerator::class,
    ];
}
```

И наконец, вам нужно зарегистрировать `App\Application\Bootloader\EntityBehaviorBootloader` в ядре приложения:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \App\Application\Bootloader\EntityBehaviorBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \App\Application\Bootloader\EntityBehaviorBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

Вот и все! Теперь вы можете использовать поведения сущностей в вашем приложении.

### Перехватчики (Interceptor)

#### Разрешение сущностей Cycle

Интеграция Cycle ORM предоставляет перехватчик (Interceptor) `Spiral\Cycle\Interceptor\CycleInterceptor`, который
позволяет автоматически разрешать сущности в методах контроллера по их первичному ключу.

> **Примечание:**
> Читайте больше об использовании перехватчиков (Interceptor) в разделе [HTTP — Interceptors](../http/interceptors.md).

Для активации перехватчика (Interceptor):

```php app/src/Application/Bootloader/AppBootloader.php
namespace App\Application\Bootloader;

use Spiral\Cycle\Interceptor\CycleInterceptor;
use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

class AppBootloader extends DomainBootloader
{
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];

    protected const INTERCEPTORS = [
        CycleInterceptor::class,
        // ...
    ];
}
```

После этого вы можете использовать внедрение сущности cycle в методах вашего контроллера:

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Entity\User;
use Spiral\Router\Annotation\Route;

final class HomeController
{
    #[Route('/users/<user>')]
    public function index(User $user)
    {
        dump($user);
    }
}
```

> **Примечание:**
> Если сущность не может быть найдена, будет выброшено исключение 404.

### Долгая работа

Cycle ORM стремится упростить использование библиотеки в демонизированных приложениях, таких как PHP воркеры,
работающие под RoadRunner или Swoole. ORM предоставляет множество опций для избежания утечек памяти, которые также
могут применяться к пакетным операциям. Это помогает обеспечить стабильность и эффективность приложения при выполнении
долго работающих процессов.

Пакет автоматически очистит кучу после каждого запроса. Если вам нужно очистить кучу вручную, вы можете использовать
следующие методы:

```php app/src/Domain/User/Service/UserService.php
use Cycle\ORM\ORMInterface;

class UserService
{
    public function __construct(
        private readonly ORMInterface $orm
    ) {
    }
    
    public function create(string $name, string $email): User
    {
        // Создание нового пользователя
        
        $this->orm->getHeap()->clean();
    }
}
```

### Консольные команды

Интеграция Cycle ORM предоставляет множество команд для упрощения управления. Вы можете получить справку по любой из
команд, используя

```bash
php app.php help cycle...
```

> **Примечание**
> Убедитесь, что включен `Spiral\Cycle\Bootloader\CommandBootloader` после загрузчиков (Bootloader) cycle для
> активации вспомогательных команд.

#### База данных

| Команда            | Описание                                                                                                                             |
|--------------------|--------------------------------------------------------------------------------------------------------------------------------------|
| `db:list [db]`     | Получает список доступных баз данных, их таблиц и количества записей.<br/>`db` имя базы данных.                                    |
| `db:table <table>` | Описывает схему таблицы конкретной базы данных.<br/>`table` Имя таблицы (обязательно).<br/>`--database` Исходная база данных.    |

#### ORM и схема

| Команда         | Описание                                                                                              |
|-----------------|-------------------------------------------------------------------------------------------------------|
| `cycle`         | Обновляет (инициализирует) схему cycle из базы данных и аннотированных классов.                      |
| `cycle:migrate` | Генерирует миграции схемы ORM.<br/>`--run` Автоматически выполняет сгенерированную миграцию.         |
| `cycle:render`  | Отображает доступные схемы CycleORM.<br/>`--no-color` Отображает вывод без цветов.                   |

> **Примечание**
> Вы можете запустить любую команду cycle с флагом `-vv`, чтобы увидеть список измененных таблиц.

<hr>


## База данных

### Доступ к базе данных

Вы можете получить доступ к вашим базам данных в контроллерах и сервисах несколькими способами:

#### Использование провайдера базы данных

```php app/src/Domain/User/Service/UserService.php
use Cycle\Database\DatabaseProviderInterface;

final class UserService 
{
    public function __construct(
        private readonly DatabaseProviderInterface $dbal
    ) {}
    
    public function store(): void
    {
        // База данных по умолчанию
        dump($this->dbal->database());
    
        // Используя псевдоним default, который указывает на основную базу данных
        dump($this->dbal->database('default'));
    
        // Вторичная
        dump($this->dbal->database('slave'));
    }
}
```

#### Использование методов и внедрения через конструктор

Компонент DBAL полностью поддерживает [IoC инъекции](../container/injectors.md) на основе имени базы данных и их
псевдонимов:

```php
use Cycle\Database\DatabaseInterface;

public function store(
    DatabaseInterface $database, 
    DatabaseInterface $primary,
    DatabaseInterface $slave
): void {
    // Database является псевдонимом для "primary"
    dump($database === $primary);

    dump($primary);
    dump($slave);
}
```

#### Использование прототипа

Получите доступ к `Cycle\Database\DatabaseProviderInterface` и экземпляру базы данных по умолчанию, используя `PrototypeTrait`:

```php app/src/Domain/User/Service/UserService.php
final class UserService 
{
    use PrototypeTrait;
    
    public function store(): void
    {
        dump($this->dbal);
        dump($this->db); // база данных по умолчанию
    }
}
```

### Выполнение запросов

Для выполнения запроса к базе данных используйте метод `query`:

```php
dump(
    $db->query('SELECT * FROM users WHERE id > ?', [
        1
    ])->fetchAll()
);
```

Для выполнения инструкции обновления или удаления используйте альтернативный метод `execute`:

```php
dump(
    $db->execute('DELETE FROM users WHERE id > ?', [
        1,
    ]) // количество затронутых строк 
);
```

> **Примечание**
> Прочтите, как использовать конструкторы запросов [здесь](https://cycle-orm.dev/docs/database-query-builders).

### Логирование

Spiral предоставляет возможность логирования запросов к базе данных через использование компонента `spiral/logger`.
Этот компонент использует Monolog в качестве драйвера логирования по умолчанию.

> **Смотрите больше**
> Читайте больше о логгере в разделе [Основы — Логирование](../basics/logging.md).

Драйверы логирования базы данных могут быть настроены в разделе `logger` файла конфигурации `app/config/database.php`.

```php app/config/database.php
return [
    'logger' => [
        'default' => null,
        'drivers' => [],
    ],

    // ...
];
```

Если драйвер логгера не определен, будет использован канал с именем текущего драйвера базы данных. Если драйвер SQLite
используется для выполнения запросов, имя канала monolog автоматически будет установлено
в `Cycle\Database\Driver\SQLite\SQLiteDriver`.

В этом случае вы можете настроить обработчик monolog для канала `Cycle\Database\Driver\SQLite\SQLiteDriver`, чтобы
логировать все запросы к базе данных.

Это можно сделать, добавив следующий код в файл `app/config/monolog.php`:

```php app/config/monolog.php
return [
    'handlers' => [
        // ...

        \Cycle\Database\Driver\SQLite\SQLiteDriver::class => [
            [
                'class' => 'log.rotate',
                'options' => [
                    'filename' => directory('runtime') . 'logs/db.log',
                    'level' => Logger::DEBUG,
                ],
            ],
        ],
    ],
    
    // ...
];
```

Раздел `drivers` в `logger` используется для указания, какой драйвер базы данных должен использовать канал логирования,
указанный ключом.

Рассмотрим следующую конфигурацию базы данных:

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            'runtime' => 'console'
        ],
    ],
    
    'databases' => [
        'default' => [
            'driver' => 'runtime',
        ],
    ],
    
    'drivers' => [
        'runtime' => new Config\SQLiteDriverConfig(...),
        // ...
    ],
];
```

Мы можем использовать массив конфигурации логгера для сопоставления драйвера базы данных `runtime` с каналом
логирования `console`.

И следующую конфигурацию monolog:

```php app/config/monolog.php
return [
    'handlers' => [
        //...
        'console' => [
            \Monolog\Handler\ErrorLogHandler::class,
        ],
    ],
];
```

С этими конфигурациями каждый раз, когда вы используете драйвер базы данных `runtime`, его логи будут отправляться в
канал `console`.

Вы также можете направить конкретный драйвер базы данных к конкретному каналу логирования. Например, вы можете иметь
отдельный канал логирования для вашей базы данных SQLite и другой для вашей базы данных MySQL. Таким образом, вы можете
отслеживать логи каждой базы данных индивидуально и устранять любые проблемы, специфичные для каждой базы данных. Это
помогает быстро выявлять и устранять проблемы, не просеивая большой и сложный файл логов.

```php app/config/database.php
return [
    'logger' => [
        'drivers' => [
            \Cycle\Database\Driver\MySQL\MySQLDriver::class => 'db_logs',
            \Cycle\Database\Driver\SQLite\SQLiteDriver::class => 'console'
        ],
    ],
];
```

В этом случае каждый раз, когда вы будете использовать драйвер базы данных `SQLiteDriver`, он будет отправлять логи в
канал логирования `console`.

Вы также можете установить ключ `default` на конкретный канал логирования. Это будет использоваться как канал
логирования по умолчанию для всех запросов, выполняемых драйверами базы данных, которые не указаны в разделе `drivers`.

```php app/config/database.php
return [
    'logger' => [
        'default' => 'console',
    ],
];
```

Установив канал логирования по умолчанию, это гарантирует, что даже если для конкретного драйвера базы данных не был
определен специфический канал логирования, все еще есть канал, доступный для логирования.

Например, если разработчик устанавливает канал логирования по умолчанию как `console`, любой драйвер базы данных без
назначенного ему специфического канала логирования будет направлять свои логи в канал `console`. Это обеспечивает
резервный вариант для ситуаций, когда канал логирования не был определен, гарантируя, что все логи будут захвачены.

### Консольные команды

Стандартные Web и GRPC пакеты включают набор консольных команд для просмотра схемы базы данных.

Активируйте загрузчик (Bootloader) `Spiral\Cycle\Bootloader\CommandBootloader` в вашем приложении:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Cycle\Bootloader\CommandBootloader::class,
        // ...
    ];
}
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Cycle\Bootloader\CommandBootloader::class,
    // ...
];
```

Читайте больше о загрузчиках (Bootloader) в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

#### Просмотр доступных драйверов и таблиц

Для просмотра доступных баз данных, драйверов и таблиц:

```terminal
php app.php db:list
```

Вывод:

```output
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mName (ID):[39m | [32mDatabase:[39m  | [32mDriver:[39m | [32mPrefix:[39m | [32mStatus:[39m   | [32mTables:[39m | [32mCount Records:[39m |
+------------+------------+---------+---------+-----------+---------+----------------+
| [32mdefault[39m    | runtime.db | SQLite  | ---     | connected | users   | [33m0[39m              |
|            |            |         |         |           | posts   | [33m0[39m              |
+------------+------------+---------+---------+-----------+---------+----------------+
```

#### Просмотр схемы таблицы

Для просмотра подробностей о конкретной таблице:

```terminal
php app.php db:table posts
```

Вывод:

```output
Columns of default.posts:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| id      | int            | primary        | int       | ---            |
| title   | string (255)   | text           | string    | ---            |
| user_id | int            | integer        | int       | ---            |
+---------+----------------+----------------+-----------+----------------+

Indexes of default.posts:
+-----------------------------------+-------+----------+
| Name:                             | Type: | Columns: |
+-----------------------------------+-------+----------+
| posts_index_user_id_5e32b9642a0ff | INDEX | user_id  |
+-----------------------------------+-------+----------+

Foreign Keys of default.posts:
+------------------+---------+----------------+-----------------+------------+------------+
| Name:            | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+------------------+---------+----------------+-----------------+------------+------------+
| posts_user_id_fk | user_id | users          | id              | CASCADE    | CASCADE    |
+------------------+---------+----------------+-----------------+------------+------------+
```

<hr>

## Миграции

Когда вы работаете с Cycle ORM и вам нужно адаптировать вашу базу данных к изменениям в сущностях вашего приложения,
пакет `cycle/migrations` предоставляет удобный способ генерации и управления вашими миграциями. Этот процесс сравнивает
вашу текущую схему базы данных с изменениями ваших сущностей и создает необходимые файлы миграций.

### Генерация миграций

После того как вы внесли изменения в ваши сущности или создали новые, вам нужно сгенерировать файлы миграций, которые
отражают эти изменения. Для этого выполните следующую команду в вашей консоли:

```terminal
php app.php cycle:migrate
```

Новые файлы миграций будут созданы в директории app/migrations.

> **Предупреждение**
> Перед выполнением вышеуказанной команды убедитесь, что применили любые существующие миграции, выполнив
> `php app.php migrate`. Это гарантирует, что схема вашей базы данных актуальна.

### Применение миграций

После генерации файлов миграций их нужно применить к вашей базе данных, чтобы внести фактические изменения.

```terminal
php app.php migrate
```

Эта команда применяет последние сгенерированные миграции. Когда вы запускаете ее, Cycle ORM обновляет схему вашей
базы данных в соответствии с изменениями, описанными в ваших файлах миграций.

**Cycle ORM предоставляет несколько команд для управления миграциями на разных этапах и для разных целей:**

#### Повтор

```terminal
php app.php migrate:replay
```

Эта команда полезна для повторного выполнения миграций. Она сначала откатывает миграции, а затем выполняет их снова.
Это может быть удобно, когда вы хотите быстро протестировать изменения, внесенные в ваши миграции.

**Опции:**

- `--all`: Эта опция повторит все миграции, а не только последнюю.

#### Откат

```terminal
php app.php migrate:rollback
```

Если вам нужно отменить миграции, используйте эту команду. Она отменяет изменения, внесенные миграциями. По умолчанию
она откатывает последнюю миграцию.

**Опции:**

- `--all`: Эта опция откатит все миграции, а не только последнюю.

#### Статус

```terminal
php app.php migrate:status
```

Хотите посмотреть, что было сделано, а что ожидается? Эта команда показывает вам список всех миграций вместе с их
статусами — были ли они применены или нет.

**Вот пример вывода:**

```output
+-----------------------------------------------------------+---------------------+---------------------+
| Migration                                                 | Created at          | Executed at         |
+-----------------------------------------------------------+---------------------+---------------------+
| 0_default_create_auth_tokens                              | 2023-09-25 16:45:13 | 2023-10-04 10:46:11 |
| 0_default_create_user_role_create_user_roles_create_users | 2023-09-26 21:22:16 | 2023-10-04 10:46:11 |
| 0_default_change_user_roles_add_read_only                 | 2023-09-27 13:53:19 | 2023-10-04 10:46:11 |
+-----------------------------------------------------------+---------------------+---------------------+
```

#### Инициализация

```terminal
php app.php migrate:init
```

Прежде чем вы сможете начать использовать миграции, компоненту миграций нужно место для записи информации о том, какие
миграции были выполнены. Эта команда настраивает это отслеживание, создавая таблицу миграций в вашей базе данных.

> **Примечание**
> Эта команда выполняется автоматически при первом запуске `php app.php migrate`.

### Конфигурация

Если вы хотите настроить, как Cycle ORM обрабатывает миграции, вы можете создать файл конфигурации
в `app/config/migration.php`. Этот файл позволяет вам установить различные опции, такие как расположение файлов
миграций или имя таблицы миграций в базе данных.

**Вот что вы можете установить в файле конфигурации:**

- **Directory**: Выберите, где сохранять файлы миграций.
- **Table**: Назовите таблицу, которая отслеживает статусы миграций.
- **Strategy**: Определите, как генерируются файлы миграций.
- **Name Generator**: Укажите, как создаются имена файлов миграций.
- **Safe**: Пропускайте запросы подтверждения во время выполнения миграций в производственных средах.

**Вот пример файла конфигурации**

```php app/config/migration.php
use Cycle\Schema\Generator\Migrations\Strategy\SingleFileStrategy;
use Cycle\Schema\Generator\Migrations\NameBasedOnChangesGenerator;

return [
    /**
     * Директория для хранения файлов миграций
     */
    'directory' => directory('app').'migrations/',

    /**
     * Имя таблицы для хранения информации о статусе миграций (для каждой базы данных)
     */
    'table' => 'migrations',
    
    /**
     * Стратегия генератора файлов миграций
     */
    'strategy' => SingleFileStrategy::class,
    
    /**
     * Генератор имен файлов миграций
     */
    'nameGenerator' => NameBasedOnChangesGenerator::class,

    /**
     * Когда установлено в true, подтверждение не будет запрашиваться при выполнении миграции.
     */
    'safe' => env('APP_ENV') === 'production',
];
```

### Стратегии файлов миграций

Начиная с версии **2.6.0** `spiral/cycle-bridge`, вы можете выбрать, как генерируются файлы миграций:

#### 1. Стратегия одного файла

Эта стратегия объединяет все изменения в один файл миграции каждый раз, когда вы запускаете команду. Это полезно, если
вы предпочитаете группировать все изменения в одном месте.

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305083 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('user_roles')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        // ...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();  
        
        $this->table('users')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        //...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();
    }  
  
    public function down(): void  
    {  
        $this->table('users')->drop();  
        $this->table('user_roles')->drop();
    }
}
```

#### 2. Стратегия множественных файлов

При таком подходе изменения разделяются на разные файлы на основе имени таблицы. Это означает, что если у вас есть
изменения для разных таблиц, изменения каждой таблицы попадают в свой собственный файл.

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305083 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('user_roles')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        // ...
	        ->setPrimaryKeys(['uuid'])  
	        ->create(); 
    }  
  
    public function down(): void  
    {  
        $this->table('user_roles')->drop();  
    }
}

```

```php
<?php  
  
declare(strict_types=1);  
  
namespace Migration;  
  
use Cycle\Migrations\Migration;  
  
class OrmDefaultA42b7e366d78543ca8c5a4b60d305043 extends Migration  
{  
    protected const DATABASE = 'default';  
  
    public function up(): void  
    {  
        $this->table('users')  
	        ->addColumn('created_at', 'datetime', ['nullable' => false, 'default' => 'CURRENT_TIMESTAMP'])  
	        //...
	        ->setPrimaryKeys(['uuid'])  
	        ->create();
    }  
  
    public function down(): void  
    {  
        $this->table('users')->drop();
    }
}
```

#### 3. Пользовательская стратегия

У вас также есть возможность создать пользовательскую стратегию миграции, реализовав
интерфейс `Cycle\Schema\Generator\Migrations\Strategy\GeneratorStrategyInterface`.

### Генерация имен файлов миграций

Начиная с версии **2.6.0** `spiral/cycle-bridge`, вы можете настроить стратегию именования по умолчанию для файлов
миграций.

По умолчанию Spiral использует `Cycle\Schema\Generator\Migrations\NameBasedOnChangesGenerator`, который учитывает все
изменения в файле миграции для создания уникального имени. Но вы можете создать пользовательскую стратегию генерации
имен файлов, реализовав интерфейс `Cycle\Schema\Generator\Migrations\NameGeneratorInterface`.

---

## Ограничения SQLite

При использовании нескольких воркеров в веб-приложении важно учитывать, как управляется одновременный доступ к ресурсам,
таким как базы данных. SQLite, хотя и является надежным и легковесным решением для баз данных, имеет ограничения, когда
речь идет об одновременном доступе нескольких воркеров.

### Блокировка файловой базы данных

Во-первых, базы данных SQLite основаны на файлах, что означает, что они хранятся как один файл на диске. Этот выбор
дизайна может создавать проблемы, когда несколько воркеров пытаются получить доступ к одной и той же базе данных SQLite
одновременно. SQLite использует блокировки на уровне файлов для поддержания целостности данных, позволяя только одному
записывающему процессу изменять файл базы данных в любой момент времени. В результате, если несколько воркеров пытаются
писать в базу данных одновременно, они столкнутся с конкуренцией и потенциальными проблемами блокировки, что приведет к
снижению производительности и потенциальному повреждению данных.

### Невозможность совместного использования баз данных в памяти

Кроме того, SQLite не предоставляет встроенного механизма для совместного использования базы данных в памяти между
несколькими процессами или воркерами. Базы данных в памяти часто используются для оптимизации производительности, так
как они исключают операции ввода-вывода на диск. Однако, поскольку SQLite не может совместно использовать базу данных в
памяти между процессами, каждый воркер будет иметь свою отдельную копию базы данных в памяти. Это означает, что любые
обновления, сделанные одним воркером, не будут видны другим воркерам, что приведет к несогласованности и неправильным
результатам.

Учитывая эти ограничения, при использовании Spiral Framework или любого другого фреймворка, который использует
несколько воркеров, рекомендуется исследовать альтернативные решения баз данных, которые лучше поддерживают
одновременный доступ. Популярные варианты включают клиент-серверные базы данных, такие как MySQL или PostgreSQL,
которые разработаны для обработки одновременных соединений и обеспечивают лучшую масштабируемость в средах с
несколькими воркерами.
