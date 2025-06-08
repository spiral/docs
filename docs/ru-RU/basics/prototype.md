# Основы — Прототипирование

Прототипирование — это фаза разработки, где вы набрасываете структуру вашего приложения, быстро итерируя по базовому 
дизайну ваших классов и их взаимодействий. Spiral предлагает мощное расширение, которое улучшает этот процесс прототипирования, 
ускоряющее разработку сервисов приложения, контроллеров, посредников и других классов через модификацию AST 
(проще говоря, оно пишет код за вас). Расширение включает IDE-дружественные подсказки для большинства обычных компонентов фреймворка 
и репозиториев Cycle.

## Установка

Убедитесь, что добавили `Spiral\Prototype\Bootloader\PrototypeBootloader` в ваш класс App:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
        // ...
    ];
}
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Prototype\Bootloader\PrototypeBootloader::class,
    // ...
];
```

Прочитайте больше о bootloaders в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::::

Теперь вы можете запустить `php app.php configure` для генерации помощников автодополнения IDE для вашего приложения.

## Использование prototype-свойств

### Прототипирование

Чтобы начать использовать возможности прототипирования, вы добавляете трейт `Spiral\Prototype\Traits\PrototypeTrait` к любому классу, который хотите 
прототипировать. Этот трейт подобен магическому помощнику, который помогает вашей IDE находить и использовать другие части вашего приложения без 
необходимости настраивать множество деталей.

![IDE Tooltips](https://user-images.githubusercontent.com/796136/67619538-8f0c8c80-f805-11e9-9cd8-0597133bf33a.gif)

**Вот более подробный взгляд на то, как добавить его к классу контроллера:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index()
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

Когда вы включаете этот трейт в ваш класс, ваша IDE начнет предлагать, какие сервисы или компоненты вы можете использовать. Например,
если ваше приложение имеет систему управления пользователями, набор `$this->users` может побудить вашу IDE предложить 
методы из вашего пользовательского сервиса или репозитория.

Это отлично подходит для быстрого тестирования идей, потому что вам не нужно настраивать все формальные соединения (как внедрение зависимостей) 
между частями вашего приложения.

### От прототипирования к реальному коду

Магические свойства удобны, но они не лучший вариант для производительности и ясности в долгосрочной перспективе. Поэтому, как только вы
удовлетворены тем, как все работает, вы можете сказать Spiral заменить магию настоящим кодом.

Просто запустите следующую команду:

```terminal
php app.php prototype:inject -r
```

Это говорит Spiral: *"Пройдись по моим классам, найди эти магические свойства и преврати их в настоящий, твердый код."* Он \
автоматически добавит необходимый код для внедрения зависимостей, делая ваше приложение готовым для реального использования.

> **Примечание**
> Используйте флаг `-r` для удаления `PrototypeTrait` из класса.

**Вот как будет выглядеть код после выполнения команды:**

```php app/src/Endpoint/Web/HomeController.php
namespace App\Endpoint\Web;

use App\Database\Repository\UserRepository;
use Spiral\Views\ViewsInterface;

class HomeController
{
    public function __construct(
        private readonly ViewsInterface $views, 
        private readonly UserRepository $users
    ) {
    }

    public function index(): string
    {
        return $this->views->render('profile', [
            'user' => $this->users->findByName('Antony')
        ]);
    }
}
```

### Обнаружение классов с прототипированием

В некоторых случаях вы можете захотеть найти все классы, которые используют prototype-свойства. Чтобы просмотреть все классы, просто выполните
следующую команду:

```terminal
php app.php prototype:usage
```

Она выведет список классов, как в примере ниже:

```terminal
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| Class:                                                             | Property:            | Target:                                                      |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
| App\Endpoint\Web\Controller\User\SetupPasswordAction               | userService          | App\Service\UserServiceInterface                             |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
| App\Endpoint\Web\Controller\User\SetupPasswordFormAction           | users                | App\Repository\UserRepositoryInterface                       |
|                                                                    | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
| App\Endpoint\Web\Controller\Auth\LoginFormAction                   | response             | Spiral\Http\ResponseWrapper                                  |
|                                                                    | views                | Spiral\Views\ViewsInterface                                  |
|                                                                    | request              | Spiral\Http\Request\InputManager                             |
|                                                                    | sessionErrors        | App\Application\HTTP\SessionErrorsInterface                  |
+--------------------------------------------------------------------+----------------------+--------------------------------------------------------------+
```

> **Примечание**
> что вы можете удалить расширение `spiral/prototype` после завершения всех инъекций.

### Проверка изменений

После предыдущего шага хорошей идеей будет проверить изменения, чтобы убедиться, что все было настроено так, как вы ожидали. 
Инструмент прототипирования умен, но всегда полезно перепроверить. Вам может потребоваться внести корректировки или 
оптимизации в код.

Как только вы довольны настройкой, вы можете продолжать создавать ваше приложение с прочным фундаментом, который инструмент прототипирования 
помог вам создать.

## Пользовательские свойства

Вы можете зарегистрировать любое количество prototype-свойств, используя `Spiral\Prototype\Bootloader\PrototypeBootloader` в вашем
bootloader:

```php
use Spiral\Prototype\Bootloader\PrototypeBootloader;

public function boot(PrototypeBootloader $prototype): void
{
    $prototype->bindProperty('myService', MyService::class);
}
```

> **Примечание**
> что вы можете комбинировать такой подход с автоматическим обнаружением классов для достижения лучшей интеграции архитектуры доменного слоя
> в ваш процесс разработки.

## На основе атрибутов

Альтернативно, вы можете использовать атрибуты для регистрации prototype классов и сервисов. Используйте
атрибут `Spiral\Prototype\Annotation\Prototyped` в классе, который хотите внедрить:

```php app/src/Domain/User/Service/UserService.php
namespace App\Domain\User\Service;

use Spiral\Prototype\Annotation\Prototyped;

#[Prototyped(property: 'userService')]
final class UserService
{
    // ...
}
```

Убедитесь, что запустили `php app.php update` или `php app.php prototype:dump` для автоматического поиска вашего сервиса.

> **Предупреждение**
> Для использования атрибутов с интерфейсами вам нужно включить поиск интерфейсов. Для этого прочитайте раздел
> [Настройка слушателей](../advanced/tokenizer.md#configuring-listeners).

## Доступные сокращения

Чтобы просмотреть все зарегистрированные сокращения в вашем приложении, выполните:

```terminal
php app.php prototype:list
```

### Доступные сокращения

Для вас доступно множество сокращений компонентов:

| Свойство     | Компонент                                                                                    |
|--------------|----------------------------------------------------------------------------------------------|
| app          | App\App (или класс, который реализует `Spiral\Boot\Kernel`)                                  |
| classLocator | Spiral\Tokenizer\ClassesInterface                                                            |
| console      | Spiral\Console\Console                                                                       |
| container    | Psr\Container\ContainerInterface                                                             |
| db           | Cycle\Database\DatabaseInterface (`spiral/cycle-bridge` пакет должен быть установлен)        |
| dbal         | Cycle\Database\DatabaseProviderInterface (`spiral/cycle-bridge` пакет должен быть установлен) |
| encrypter    | Spiral\Encrypter\EncrypterInterface                                                          |
| env          | Spiral\Boot\EnvironmentInterface                                                             |
| files        | Spiral\Files\FilesInterface                                                                  |
| guard        | Spiral\Security\GuardInterface                                                               |
| http         | Spiral\Http\Http                                                                             |
| i18n         | Spiral\Translator\TranslatorInterface                                                        |
| input        | Spiral\Http\Request\InputManager                                                             |
| session      | Spiral\Session\SessionScope                                                                  |
| cookies      | Spiral\Cookies\CookieManager                                                                 |
| logger       | Psr\Log\LoggerInterface                                                                      |
| logs         | Spiral\Logger\LogsInterface                                                                  |
| memory       | Spiral\Boot\MemoryInterface                                                                  |
| orm          | Cycle\ORM\ORMInterface (`spiral/cycle-bridge` пакет должен быть установлен)                  |
| paginators   | Spiral\Pagination\PaginationProviderInterface                                                |
| queue        | Spiral\Queue\QueueInterface                                                                  |
| queueManager | Spiral\Queue\QueueConnectionProviderInterface                                                |
| request      | Spiral\Http\Request\InputManager                                                             |
| response     | Spiral\Http\ResponseWrapper                                                                  |
| router       | Spiral\Router\RouterInterface                                                                |
| server       | Spiral\Goridge\RPC (`spiral/roadrunner-bridge` пакет должен быть установлен)                 |
| snapshots    | Spiral\Snapshots\SnapshotterInterface                                                        |
| storage      | Spiral\Storage\StorageInterface                                                              |
| validator    | Spiral\Validation\ValidationInterface                                                        |
| views        | Spiral\Views\ViewsInterface                                                                  |
| auth         | Spiral\Auth\AuthScope                                                                        |
| authTokens   | Spiral\Auth\TokenStorageInterface                                                            |
| cache        | Psr\SimpleCache\CacheInterface                                                               |
| cacheManager | Spiral\Cache\CacheStorageProviderInterface                                                   |
