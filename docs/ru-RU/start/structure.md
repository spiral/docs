# Начало работы — Структура каталогов

Spiral не навязывает какую-либо структуру каталогов для вашего приложения, поэтому у вас есть гибкость в организации ваших файлов и каталогов. Однако он предоставляет рекомендуемую структуру, которая может быть использована в качестве отправной точки. Эта структура также может быть легко изменена.

## Каталоги

Структура каталогов по умолчанию может контролироваться через метод `mapDirectories` класса Kernel. По умолчанию все каталоги приложения будут рассчитаны на основе `root` с использованием следующего шаблона:

| Каталог   | Значение               |
|-----------|------------------------|
| root      | **задается пользователем** |
| app       | **root**/app           |
| config    | **app**/config         |
| resources | **app**/resources      |
| runtime   | **root**/runtime       |
| cache     | **root**/runtime/cache |
| public    | **root**/public        |
| vendor    | **root**/vendor        |

Некоторые компоненты объявят свои собственные каталоги, такие как:

| Компонент         | Каталог    | Значение           |
|-------------------|------------|--------------------|
| spiral/views      | views      | **app**/views      |
| spiral/translator | locale     | **app**/locale     |
| spiral/migrations | migrations | **app**/migrations |

## Инициализация каталогов

Вы можете установить значение каталога `root` или любого другого каталога в вашем файле `app.php`.

```php
$app = \App\Application\Kernel::create(
    directories: ['root' => __DIR__]
)->run();
```

Например, если вы хотели изменить расположение каталога `runtime` на временный каталог системы, вы бы сделали это:

```php
$app = \App\Application\Kernel::create(
    directories: [
        'root' => __DIR__, 
        'runtime' => \sys_get_temp_dir()
    ]
)->run();
```

Вы можете получить доступ к путям различных каталогов в вашем приложении, используя класс `Spiral\Boot\DirectoriesInterface`. Этот интерфейс предоставляет методы для доступа к путям различных каталогов, которые определены через метод `mapDirectories`.

Вот пример того, как вы можете использовать класс `DirectoriesInterface` для доступа к пути каталога `runtime`:

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

Вы также можете использовать функцию `directory` внутри глобальной области видимости (Scope) IoC (конфигурационные файлы, контроллеры, код сервисов).

```php app/config/cache.php
return [
    'storages' => [
        'file' => [
            'path' => directory('runtime') . 'cache',
        ],   
    ],
];
```

## Пространства имен

По умолчанию все каркасные приложения используют корневое пространство имен `App`, указывающее на каталог `app/src`. Вы можете изменить базовое пространство имен на любое желаемое в `composer.json`:

```json composer.json
{
  "autoload": {
    "psr-4": {
      "App\\": "app/src/"
    }
  }
}
```

## Структура приложения

Структура, которую мы представили ниже, является общей структурой, используемой во многих PHP-приложениях, и она может служить отправной точкой для ваших проектов. Следуя этой структуре, вы можете организовать свой код логичным и поддерживаемым способом, что облегчит создание и масштабирование ваших приложений со временем. Конечно, вам может потребоваться внести изменения, чтобы соответствовать конкретным потребностям вашего проекта, но эта структура обеспечивает прочную основу для большинства приложений.

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

#### Вот краткое объяснение каталогов и файлов в этой структуре:

- **Endpoint**: Этот каталог содержит точки входа для вашего приложения, включая HTTP-конечные точки (в подкаталоге Web), интерфейсы командной строки (в подкаталоге Console) и gRPC-сервисы (в подкаталоге RPC).

- **Application**: Этот каталог содержит ядро вашего приложения, включая класс Kernel, который загружает ваше приложение, классы загрузчиков (Bootloader), которые регистрируют сервисы в контейнере, и каталог Exception, который содержит логику обработки исключений.

- **Domain**: Этот каталог содержит логику вашего домена, организованную по поддоменам. Например, Entity для модели User, Service для сохранения новых пользователей, Repository для получения пользователей из базы данных и Exception для обработки ошибок, связанных с пользователями.

- **Infrastructure**: Этот каталог содержит инфраструктурный код для вашего приложения, включая каталог Persistence для кода, связанного с базой данных, каталог CycleORM для кода, связанного с ORM, и каталог Interceptor для глобальных перехватчиков (Interceptor).

<hr>

## Что дальше?

Теперь изучите основы более глубоко, прочитав некоторые статьи:

* [Ядро и окружение](../framework/kernel.md)
* [Файлы и каталоги](../basics/files.md)
