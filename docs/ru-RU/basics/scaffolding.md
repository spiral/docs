# Основы — Генерация кода (Scaffolding)

Spiral предоставляет компонент `spiral/scaffolder`. Этот мощный инструмент позволяет разработчикам быстро и легко генерировать код приложения для различных классов, используя набор консольных команд:

- Bootloader'ы приложения,
- консольные команды,
- конфигурации приложения,
- HTTP-контроллеры, посредники (middleware), фильтры запросов,
- обработчики задач очереди.

и многое другое...

## Установка

Чтобы активировать компонент, достаточно добавить класс `Spiral\Scaffolder\Bootloader\ScaffolderBootloader` в список Bootloader'ов:

:::: tabs

::: tab Используя метод

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
        // ...
    ];
}
```

Подробнее о Bootloader'ах читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Используя константу

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\Scaffolder\Bootloader\ScaffolderBootloader::class,
    // ...
];
```

Подробнее о Bootloader'ах читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

## Конфигурация

После добавления Bootloader'а вы можете настроить компонент под свои потребности, заменив генераторы объявлений и их параметры, используя конфигурационный файл `scaffolder`.

Вот конфигурация по умолчанию для доступных объявлений:

```php 
use Spiral\Scaffolder\Declaration;

return [
    'declarations' => [
        Declaration\BootloaderDeclaration::TYPE => [
            'namespace' => 'Bootloader',
            'postfix' => 'Bootloader',
            'class' => Declaration\BootloaderDeclaration::class,
        ],
        Declaration\ConfigDeclaration::TYPE => [
            'namespace' => 'Config',
            'postfix' => 'Config',
            'class' => Declaration\ConfigDeclaration::class,
            'options' => [
                'directory' => directory('config'),
            ],
        ],
        Declaration\ControllerDeclaration::TYPE => [
            'namespace' => 'Controller',
            'postfix' => 'Controller',
            'class' => Declaration\ControllerDeclaration::class,
        ],
        Declaration\FilterDeclaration::TYPE => [
            'namespace' => 'Filter',
            'postfix' => 'Filter',
            'class' => Declaration\FilterDeclaration::class,
        ],
        Declaration\MiddlewareDeclaration::TYPE => [
            'namespace' => 'Middleware',
            'postfix' => '',
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Command',
            'postfix' => 'Command',
            'class' => Declaration\CommandDeclaration::class,
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Job',
            'postfix' => 'Job',
            'class' => Declaration\JobHandlerDeclaration::class,
        ],
    ],
];
```

Вы можете настроить пространство имен класса, постфикс и тип объявления для каждого доступного типа объявления класса. Нет необходимости переопределять всю конфигурацию объявлений. Вместо этого вы можете настроить только те типы объявлений, которые вам нужны.

Вот пример того, как вы можете настроить конфигурацию:

```php app/config/scaffolder.php
use Spiral\Scaffolder\Declaration;

return [
    // ...
    'declarations' => [
        Declaration\MiddlewareDeclaration::TYPE => [
            'class' => Declaration\MiddlewareDeclaration::class,
        ],
        Declaration\CommandDeclaration::TYPE => [
            'namespace' => 'Endpoint\Console',
        ],
        Declaration\JobHandlerDeclaration::TYPE => [
            'namespace' => 'Endpoint\Queue',
            'postfix' => 'Job',
        ],
    ],
];
```

> **Примечание**
> Этот подход особенно полезен в больших приложениях, где может использоваться множество различных типов объявлений. Настраивая только то, что необходимо, процесс конфигурации может быть упрощен, а ошибки сведены к минимуму.

### Изменение директории для генерируемых классов

По умолчанию генератор создает классы в директории `app/src`. В некоторых случаях вы можете захотеть изменить директорию, где генератор создает классы.

Вы можете сделать это, используя опцию `directory`.

```php app/config/scaffolder.php
return [
    'directory' => directory('app') . '/Generated' // <=============
];
```

Вы также можете изменить директорию для конкретных объявлений. Например, вы можете захотеть генерировать консольные команды в директории `app/src/Endpoint/Console`. Для этого можно использовать опцию `directory`:

```php app/config/scaffolder.php
return [
    'declarations' => [
        Declaration\CommandDeclaration::TYPE => [
            // ...
            'directory' => directory('app') . '/Endpoint/Console' // <=============
        ],
    ],
];
```

## Добавление пользовательских объявлений через ScaffolderBootloader

Возможно зарегистрировать пользовательские объявления. Вы можете сделать это, зарегистрировав свои пользовательские объявления с помощью `ScaffolderBootloader`:

```php app/src/Application/Bootloader/ScaffolderBootloader.php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Scaffolder\Bootloader\ScaffolderBootloader as BaseScaffolderBootloader;

final class ScaffolderBootloader extends Bootloader
{
    public function init(BaseScaffolderBootloader $scaffolder): void
    {
        $scaffolder->addDeclaration('Repository', [
            'namespace' => 'App\\Repository',
            'postfix'   => 'Repository',
            'class'     => RepositoryDeclaration::class,
            'options'   => [
                'orm' => 'cycle',
                // некоторые пользовательские параметры
            ],
        ]);
    }
}
```

Регистрируя пользовательские объявления, вы можете расширить функциональность компонента для удовлетворения конкретных потребностей вашего приложения.

## Доступные команды

| Команда           | Описание                                    |
|-------------------|---------------------------------------------|
| create:bootloader | Создать объявление Bootloader'а            |
| create:command    | Создать объявление команды                  |
| create:config     | Создать объявление конфигурации             |
| create:controller | Создать объявление контроллера              |
| create:middleware | Создать объявление промежуточного ПО        |
| create:filter     | Создать объявление фильтра запроса          |
| create:jobHandler | Создать объявление обработчика задач        |

Некоторые пакеты могут предоставлять свои собственные команды. Например, пакет `Cycle Bridge` (если установлен) предоставляет следующие команды:

| Команда           | Описание                           |
|-------------------|------------------------------------|
| create:migration  | Создать объявление миграции        |
| create:repository | Создать объявление репозитория     | 
| create:entity     | Создать объявление сущности        | 

> **Подробнее**
> Узнайте больше о пакете `Cycle Bridge` и доступных консольных
> командах [здесь](https://spiral.dev/docs/packages-cycle-bridge).

### Bootloader

Эта команда создает класс Bootloader. Bootloader'ы отвечают за инициализацию и конфигурацию компонентов во время запуска приложения. С помощью этой команды вы можете быстро сгенерировать код для нового Bootloader'а, который затем можете настроить под свои конкретные потребности.

> **Подробнее**
> Узнайте больше о Bootloader'ах в разделе [Framework — Bootloaders](../framework/bootloaders.md).

```terminal
php app.php create:bootloader <n>
```

Будет создан класс `<n>Bootloader`.

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\BootloaderDeclaration::TYPE => [
    'namespace' => 'Application\Bootloader',
],
```

#### Пример

```terminal
php app.php create:bootloader App
```

Результат:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

final class AppBootloader extends Bootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [];
    protected const DEPENDENCIES = [];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

Используя опцию `-d`, вы можете сгенерировать доменный Bootloader.

#### Пример

```terminal
php app.php create:bootloader App -d
```

Результат:

```php app/src/Application/Bootloader/AppBootloader.php
declare(strict_types=1);

namespace App\Application\Bootloader;

use Spiral\Bootloader\DomainBootloader;
use Spiral\Core\CoreInterface;

final class AppBootloader extends DomainBootloader
{
    protected const BINDINGS = [];
    protected const SINGLETONS = [
        CoreInterface::class => [self::class, 'domainCore']
    ];
    protected const DEPENDENCIES = [];
    protected const INTERCEPTORS = [
        // Поместите ваши Interceptor'ы здесь
    ];

    public function init(): void
    {
    }

    public function boot(): void
    {
    }
}
```

### Консольная команда

Эта команда создает класс консольной команды. Команды предоставляют способ выполнения функциональности приложения через консоль. С помощью этой команды вы можете сгенерировать код для нового объявления команды, который затем можете настроить для реализации желаемой консольной функциональности.

> **Подробнее**
> Узнайте больше о консольных командах в разделе [Console — Getting started](../console/configuration.md).

```terminal
php app.php create:command <n> [alias]
```

Будет сгенерирован класс `<n>Command`. Имя команды будет равно `name` или `alias` (если это значение задано).

Если псевдоним не предоставлен, он будет сгенерирован автоматически из имени. Например, если имя `CreateUser`, псевдоним будет `create:user`.

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\CommandDeclaration::TYPE => [
    'namespace' => 'Endpoint\Console',
],
```

#### Пример без `alias`

```terminal
php app.php create:command UserRegister
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    public function __invoke(): int
    {
        // Поместите логику вашей команды здесь
        $this->info('Логика команды еще не реализована');

        return self::SUCCESS;
    }
}
```

#### Пример с псевдонимом

```terminal
php app.php create:command UserRegister create:user
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user')]
final class UserRegisterCommand extends Command
```

#### Опции и аргументы

Вы также можете использовать опции `-a` и `-o` для добавления аргументов и опций к команде.

```terminal
php app.php create:command UserRegister -a username -a password -o isAdmin
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
declare(strict_types=1);

namespace App\Endpoint\Console;

use Spiral\Console\Attribute\Argument;
use Spiral\Console\Attribute\AsCommand;
use Spiral\Console\Attribute\Option;
use Spiral\Console\Attribute\Question;
use Spiral\Console\Command;

#[AsCommand(name: 'user:register')]
final class UserRegisterCommand extends Command
{
    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the username argument?')]
    private string $username;

    #[Argument(description: 'Argument description')]
    #[Question(question: 'What would you like to name the password argument?')]
    private string $password;

    #[Option(description: 'Argument description')]
    private bool $isAdmin;

    public function __invoke(): int
    {
        // Поместите логику вашей команды здесь
        $this->info('Логика команды еще не реализована');

        return self::SUCCESS;
    }
}
```

#### Описание команды

Вы также можете использовать опцию `-d` для добавления описания к команде.

```terminal
php app.php create:command UserRegister -d "Register a new user"
```

Результат:

```php app/src/Endpoint/Console/UserRegisterCommand.php
#[AsCommand(name: 'create:user', description: 'Register a new user')]
final class UserRegisterCommand extends Command
```

### Конфигурация приложения

Эта команда создает класс конфигурации. Конфигурации предоставляют способ управления настройками конфигурации приложения. С помощью этой команды вы можете сгенерировать код для новой конфигурации, который затем можете настроить для управления настройками конфигурации вашего приложения.

> **Подробнее**
> Узнайте больше о конфигурациях приложения в разделе [Framework — Config Objects](../framework/config.md).

```terminal
php app.php create:config <n>
```

Будет создан класс `<n>Config` и файл `<app directory>/config/<n>.php`, если он не существует.

#### Доступные опции:

`reverse (r)` - Используя этот флаг, генератор будет искать файл `<app directory>/config/<n>.php` и создаст класс конфигурации на основе его содержимого. Сгенерированный класс будет включать значения по умолчанию и геттеры, а в некоторых случаях также будет включать геттеры по ключу для значений массивов. Если значение массива состоит из более чем одного подзначения с одинаковыми типами для ключей и подзначений, генератор попытается создать метод геттера по ключу. Если сгенерированный ключ конфликтует с существующим методом, геттер по ключу будет пропущен.

#### Пример с пустым конфигурационным файлом

```terminal
php app.php create:config app
```

Выходной конфигурационный файл:

```php app/config/app.php
return [];
```

Выходной класс конфигурации:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Значения по умолчанию для конфигурации.
     * Будут объединены с конфигурацией приложения во время выполнения.
     */
    protected array $config = [];
}
```

#### Пример с обращением

```php app/config/app.php
return [
    //создаст "getParam()" геттер по ключу (успешно сингуляризованное имя)
    'params' => [
        'one' => 'param',
        'two' => 'another param',
    ],
    //создаст "getParameterBy()" геттер по ключу (неуспешно сингуляризованное имя)
    'parameter' => [
        'one' => 'parameter',
        'two' => 'another parameter',
    ],
    //создаст "getValueBy()" геттер по ключу (потому что "getValue()" конфликтует со следующим полем "value")
    'values' => [
        1 => 'value',
        2 => 'another value',
    ],
    'value' => 'third value',
    //не создаст геттер по ключу из-за только одного подзначения
    'few' => [
        'one' => 'value',
    ],
    //не создаст геттер по ключу из-за смешанных типов значений
    'mixedValues' => [
        'one' => 'value',
        'two' => 2,
    ],
    //не создаст геттер по ключу из-за смешанных типов ключей
    'mixedKeys' => [
        'one' => 'value',
        2 => 'another value',
    ],
    //не создаст геттер по ключу из-за конфликтов имен
    //(потому что "getConflict()" и "getConflictBy()" конфликтуют со следующими полями "conflict" и "conflictBy")
    'conflicts' => [
        'one' => 'conflict',
        'two' => 'another conflict',
    ],
    'conflict' => 'third conflic',
    'conflictBy' => 'fourth conflic',
];
```

```terminal
php app.php create:config my -r
```

Результат:

```php app/src/Application/Config/AppConfig.php
declare(strict_types=1);

namespace App\Application\Config;

use Spiral\Core\InjectableConfig;

final class AppConfig extends InjectableConfig
{
    public const CONFIG = 'app';

    /**
     * Значения по умолчанию для конфигурации.
     * Будут объединены с конфигурацией приложения во время выполнения.
     */
    protected array $config = [
        'params' => [],
        'parameter' => [],
        'values' => [],
        'value' => '',
        'few' => [],
        'mixedValues' => [],
        'mixedKeys' => [],
        'conflicts' => [],
        'conflict' => '',
        'conflictBy' => '',
    ];

    public function getParams(): array
    {
        return $this->config['params'];
    }

    public function getParameter(): array
    {
        return $this->config['parameter'];
    }

    public function getValues(): array
    {
        return $this->config['values'];
    }

    public function getValue(): string
    {
        return $this->config['value'];
    }

    public function getFew(): array
    {
        return $this->config['few'];
    }

    public function getMixedValues(): array
    {
        return $this->config['mixedValues'];
    }

    public function getMixedKeys(): array
    {
        return $this->config['mixedKeys'];
    }

    public function getConflicts(): array
    {
        return $this->config['conflicts'];
    }

    public function getConflict(): string
    {
        return $this->config['conflict'];
    }

    public function getConflictBy(): string
    {
        return $this->config['conflictBy'];
    }

    public function getParam(string $param): string
    {
        return $this->config['params'][$param];
    }

    public function getParameterBy(string $parameter): string
    {
        return $this->config['parameter'][$parameter];
    }

    public function getValueBy(int $value): string
    {
        return $this->config['values'][$value];
    }
}
```


### HTTP-контроллер

Эта команда создает класс контроллера. Контроллеры обрабатывают HTTP-запросы и ответы для конкретных эндпоинтов в вашем приложении. С помощью этой команды вы можете сгенерировать код для нового класса контроллера, который затем можете настроить для обработки HTTP-запросов и ответов по мере необходимости.

> **Подробнее**
> Узнайте больше о HTTP-контроллерах в разделе [HTTP — Getting started](../http/configuration.md).

```terminal
php app.php create:controller <n>
```

Будет создан класс `<n>Controller`. Доступные опции:

* `action (a)` (допускается несколько значений) - вы можете добавить действия, используя эту опцию
* `prototype (p)` - если установлен, будет добавлен `PrototypeTrait`

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\ControllerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web',
],
```

#### Пример с пустым списком действий

```terminal
php app.php create:controller User
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
}

```

#### Пример с опцией `prototype`

```terminal
php app.php create:controller User -p
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Prototype\Traits\PrototypeTrait;
use Spiral\Router\Annotation\Route;

class UserController
{
    use PrototypeTrait;
}
```

#### Пример со списком действий

```bash
php app.php create:controller User \
      -a index \
      -a show \
      -a create \
      -a update \
      -a delete
```

Результат:

```php app/src/Endpoint/Web/UserController.php
declare(strict_types=1);

namespace App\Endpoint\Web;

use Psr\Http\Message\ResponseInterface;
use Spiral\Router\Annotation\Route;

class UserController
{
    /**
     * Пожалуйста, не забудьте настроить атрибут Route или удалить его и зарегистрировать маршрут вручную.
     */
    #[Route(route: 'path', name: 'name')]
    public function index(): ResponseInterface
    {
    }

    /**
     * Пожалуйста, не забудьте настроить атрибут Route или удалить его и зарегистрировать маршрут вручную.
     */
    #[Route(route: 'path', name: 'name')]
    public function show(): ResponseInterface
    {
    }

    /**
     * Пожалуйста, не забудьте настроить атрибут Route или удалить его и зарегистрировать маршрут вручную.
     */
    #[Route(route: 'path', name: 'name')]
    public function create(): ResponseInterface
    {
    }

    /**
     * Пожалуйста, не забудьте настроить атрибут Route или удалить его и зарегистрировать маршрут вручную.
     */
    #[Route(route: 'path', name: 'name')]
    public function update(): ResponseInterface
    {
    }

    /**
     * Пожалуйста, не забудьте настроить атрибут Route или удалить его и зарегистрировать маршрут вручную.
     */
    #[Route(route: 'path', name: 'name')]
    public function delete(): ResponseInterface
    {
    }
}
```

### Фильтр запроса

Эта команда создает класс фильтра запроса. Фильтры запросов предоставляют способ сопоставления и валидации HTTP-запросов до того, как они будут обработаны контроллером. С помощью этой команды вы можете сгенерировать код для нового объявления фильтра запроса, который затем можете настроить для изменения HTTP-запросов по мере необходимости.

> **Подробнее**
> Узнайте больше о фильтрах запросов в разделе [Filters — Getting started](../filters/configuration.md).

```terminal
php app.php create:filter <n>
```

Будет сгенерирован класс `<n>Filter`.

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\FilterDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Filter',
],
```

#### Пример

```terminal
php app.php create:filter CreateUser
```

> **Предупреждение**
> Убедитесь, что в вашем приложении включен Bootloader `Spiral\Validation\Bootloader\ValidationBootloader`.

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
}
```

#### Создание фильтра со свойствами

Опция `property (p)` используется для определения свойств класса фильтра. Каждое свойство определяется с использованием формата `<n>:<source>:<type>`, где:

- `<n>` - имя свойства,
- `<source>` - источник входных данных (например, `post`, `get`, `header`, `cookie`, `server` и т.д.),
- и `<type>` - тип свойства (например, `string`, `int`, `bool` или `array`).

```terminal
php app.php create:filter CreateUser -p username:post -p tags:post:array -p ip:ip -p token:header -p status:query:int
```

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;

final class CreateUserFilter extends Filter
{
    #[Post(key: 'username')]
    public string $username;

    #[Post(key: 'tags')]
    public array $tags;

    #[RemoteAddress(key: 'ip')]
    public string $ip;

    #[Header(key: 'token')]
    public string $token;

    #[Query(key: 'status')]
    public int $status;
}
```

> **Примечание**
> Узнайте больше о доступных атрибутах [здесь](../filters/filter.md).

#### Создание фильтра с правилами валидации

Чтобы сгенерировать фильтр с правилами валидации, просто добавьте опцию `-s` к команде:

```terminal
php app.php create:filter CreateUser -p ... -s
```

Результат:

```php app/src/Endpoint/Web/Filter/CreateUserFilter.php
declare(strict_types=1);

namespace App\Endpoint\Web\Filter;

use Spiral\Filters\Attribute\Input\Header;
use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Attribute\Input\Query;
use Spiral\Filters\Attribute\Input\RemoteAddress;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

final class CreateUserFilter extends Filter implements HasFilterDefinition
{
    // ...

    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition(validationRules: [
            // Поместите ваши правила валидации здесь
        ]);
    }
}
```

> **Предупреждение**
> В вашем приложении должна быть установлена библиотека валидации. Узнайте больше о доступных библиотеках валидации [здесь](../validation/factory.md).

### HTTP-посредник (Middleware)

Эта команда создает класс промежуточного ПО (Middleware). Промежуточное ПО предоставляет способ изменения HTTP-запросов и ответов при их прохождении через стек промежуточного ПО приложения. С помощью этой команды вы можете сгенерировать код для нового объявления промежуточного ПО, который затем можете настроить для изменения HTTP-запросов и ответов по мере необходимости.

> **Подробнее**
> Узнайте больше о промежуточном ПО в разделе [HTTP — Middleware](../http/middleware.md).

```terminal
php app.php create:middleware <n>
```

Будет сгенерирован класс `<n>`.

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\MiddlewareDeclaration::TYPE => [
    'namespace' => 'Endpoint\Web\Middleware',
    'postfix' => 'Middleware',
],
```

#### Пример

```terminal
php app.php create:middleware Logger
```

Результат:

```php app/src/Endpoint/Web/Middleware/LoggerMiddleware.php
declare(strict_types=1);

namespace App\Endpoint\Web\Middleware;

class LoggerMiddleware implements \Psr\Http\Server\MiddlewareInterface
{
    public function process(
        \Psr\Http\Message\ServerRequestInterface $request,
        \Psr\Http\Server\RequestHandlerInterface $handler,
    ): \Psr\Http\Message\ResponseInterface
    {
        return $handler->handle($request);
    }
}
```

### Обработчик задач

Эта команда создает класс обработчика задач. Обработчики задач обрабатывают задания, которые используются для обработки задач в очереди. С помощью этой команды вы можете сгенерировать код для нового объявления обработчика задач, который затем можете настроить для обработки заданий по мере необходимости.

> **Подробнее**
> Узнайте больше о заданиях и очередях в разделе [Queue — Getting started](../queue/configuration.md).

```terminal
php app.php create:jobHandler <n>
```

Будет создан класс `<n>Job`.

#### Конфигурация

В нашем примере мы используем следующую конфигурацию объявления:

```php
Spiral\Scaffolder\Declaration\JobHandlerDeclaration::TYPE => [
    'namespace' => 'Endpoint\Job',
],
```

#### Пример

```terminal
php app.php create:jobHandler UserRegisteredNotification
```

Результат:

```php app/src/Endpoint/Job/UserRegisteredNotificationJob.php
declare(strict_types=1);

namespace App\Endpoint\Job;

use Spiral\Queue\JobHandler;

final class UserRegisteredNotificationJob extends JobHandler
{
    public function invoke(string $id, array $payload, array $headers): void
    {
    }
}
```