# Компонент — Статический анализ

Tokenizer является ключевым компонентом, предлагающим широкий спектр функциональных возможностей, которые значительно
улучшают опыт разработки в приложениях Spiral. Его основная роль заключается в сканировании указанных директорий,
позволяя разработчикам легко управлять и организовывать свою кодовую базу. Этот инструмент особенно хорошо справляется
с идентификацией и использованием классов на основе определенных интерфейсов или атрибутов, обеспечивая разнообразные
практические применения.

#### Основные случаи использования

1. **Автоматическая регистрация маршрутов**: Одно из классических применений — идентификация атрибутов маршрутов в
   действиях контроллеров. Таким образом, он автоматизирует процесс регистрации маршрутов в вашем приложении, используя
   атрибуты, определенные в контроллерах. Эта функция значительно снижает ручные накладные расходы и упрощает механизм
   маршрутизации.

2. **Поддержка модульной структуры**: В проектах, где модульная архитектура является важной, Tokenizer превосходен. Он
   может автоматически обнаруживать и регистрировать модули (например, добавленные как composer-пакеты), которые
   реализуют определенный интерфейс. Эта возможность обеспечивает бесшовную интеграцию и управление различными модулями
   в вашем приложении.

3. **Конфигурация на основе атрибутов**: Tokenizer может сканировать определенные атрибуты в вашей кодовой базе,
   обеспечивая конфигурацию и определение поведения на основе атрибутов. Этот подход соответствует современным практикам
   кодирования, где атрибуты используются для определения аспектов, таких как внедрение зависимостей, настройки
   конфигурации и многое другое.

## Локаторы

Локаторы — это как прожектор для вашего кода. Они помогают найти определенные части кода.

### Локатор классов

Если вы стремитесь найти классы, обратитесь к `Spiral\Tokenizer\ClassesInterface`. Используя его, вы можете искать
классы по их имени, интерфейсам, которые они реализуют, или трейтам, которые они включают.

Вот быстрый пример. Допустим, вы хотите найти все классы, которые реализуют интерфейс 
`\Psr\Http\Server\MiddlewareInterface`:

```php
use Spiral\Tokenizer\ClassesInterface;

public function findClasses(ClassesInterface $classes): void
{
    foreach ($classes->getClasses(\Psr\Http\Server\MiddlewareInterface::class) as $middleware) {
        dump($middleware->getFileName());
    }
}
```

Метод `getClasses` затем вернет массив объектов `ReflectionClass`, представляющих найденные классы.

### Локатор перечислений

Если вы ищете перечисления, вам нужен `Spiral\Tokenizer\EnumsInterface`. Он поставляется с методом `getEnums`, чтобы
помочь вам в поиске:

```php
use Spiral\Tokenizer\EnumsInterface;

public function findEnums(EnumsInterface $enums): void
{
    foreach ($enums->getEnums() as $enum) {
        dump($enum->getFileName());
    }
}
```

### Локатор интерфейсов

Если вы хотите найти определенные интерфейсы, `Spiral\Tokenizer\InterfacesInterface` поможет вам. Используйте его
метод `getInterfaces` следующим образом:

```php
use Spiral\Tokenizer\InterfacesInterface;

public function findEnums(InterfacesInterface $interfaces): void
{
    foreach ($interfaces->getInterfaces() as $interface) {
        dump($interface->getFileName());
    }
}
```

> **Предупреждение**
> Tokenizer будет игнорировать все файлы, которые содержат операторы `include` или `require`. Это потому, что небезопасно
> требовать такие рефлексии. Пожалуйста, не используйте их в вашем коде.

### Настройка директорий поиска

Tokenizer по умолчанию проводит поиск в директории приложения. Однако вы часто можете захотеть, чтобы он рассматривал
и другие директории. К счастью, настройка этого проста с использованием `Spiral\Bootloader\TokenizerBootloader`.

:::: tabs

::: tab Bootloader

Вот как вы можете указать дополнительные директории:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addDirectory($directories->get('vendor') . 'spiral/validator/src');
    }
}
```

:::

::: tab Config

Альтернативно, вы можете добавить директории непосредственно в файл конфигурации `app/config/tokenizer.php`:

```php app/config/tokenizer.php
return [
    'directories' => [
        directory('app'),
        directory('vendor') . 'spiral/validator/src',
    ],
];
```

:::

::::

### Исключение определенных директорий

Вы также можете исключить определенные директории из поиска Tokenizer. Вот как:

```php app/config/tokenizer.php
return [
    'directories' => [
        //...
    ],
    'exclude' => [
        directory('resources'),
        directory('config'),
        'tests',
        'migrations',
    ],
];
```

> **Примечание**
> Помните, расширение директорий для поиска классов может замедлить процесс. Рекомендуется добавлять только те
> директории, которые необходимы для ваших потребностей.

### Локатор классов с областью видимости

При работе с обширными директориями Tokenizer может немного замедлиться. Однако вы можете ускорить его, используя
локатор классов с областью видимости. Этот инструмент позволяет разделить и покорить, настраивая определенные зоны
поиска, которые мы называем `scopes` (области видимости).

Если вам нужно искать классы в большом количестве директорий, компонент tokenizer может страдать от плохой
производительности. В этом случае вы можете использовать локатор классов с областью видимости для улучшения
производительности.

С локатором классов с областью видимости вы можете определить директории для поиска в именованных областях видимости.
Это позволяет селективно искать только те директории, которые релевантны для вашей текущей задачи.

#### Настройка областей видимости

Чтобы использовать локатор классов с областью видимости, вам нужно определить ваши области видимости:

:::: tabs

::: tab Config

Области видимости могут быть определены в файле конфигурации `app\config\tokenizer.php`.

Вот пример того, как определить область видимости с именем `scopeName`, которая ищет в директории `app/Directory`:

```php app/config/tokenizer.php
return [
    'scopes' => [
        'scopeName' => [
            'directories' => [
                directory('app') . 'Directory',
            ],
            'exclude' => [
                directory('app') . 'Directory/Excluded',
            ]
        ],
    ]
];
```

> **Примечание**
> Параметр `exclude` здесь не случайно. Если есть части директории, которые, как вы знаете, вам не понадобятся, просто
> скажите Tokenizer пропустить их. Это сделает процесс еще быстрее!

:::

::: tab Bootloader

Области видимости могут быть определены с использованием загрузчика `Spiral\Bootloader\TokenizerBootloader`:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Bootloader\TokenizerBootloader;
use Spiral\Boot\DirectoriesInterface;

class AppBootloader extends Bootloader
{
    public function init(DirectoriesInterface $directories, TokenizerBootloader $tokenizer): void
    {
        $tokenizer->addScopedDirectory('scopeName', $directories->get('app') . 'Directory');
    }
}
```

:::

::::

Как только у вас готовы области видимости, вы можете затем проинструктировать Tokenizer искать только в выбранной
области видимости. Это как сказать ему, в какой отдел идти!

:::: tabs

::: tab Classes

`Spiral\Tokenizer\ScopedClassesInterface` позволяет вам делать это с его методом `getScopedClasses`. Просто передайте
ему имя области видимости, и он вернет все классы, которые найдет в этой зоне.

Чтобы использовать метод, вам нужно передать имя `scope` в качестве аргумента. Метод затем вернет массив объектов
`ReflectionClass`, представляющих классы, найденные в этой области видимости.

```php
use Spiral\Tokenizer\ScopedClassesInterface;

final class SomeLocator
{
    public function __construct(
        private readonly ScopedClassesInterface $locator
    ) {
    }

    public function findDeclarations(): array
    {
        foreach ($this->locator->getScopedClasses('scopeName') as $class) {
            // ...
        }
    }
}
```

:::

::: tab Enums

Точно так же, как и с классами, вы можете настроить определенные области видимости для поиска перечислений. Как только у
вас настроены области видимости в файле `app\config\tokenizer.php`, вы можете использовать
`Spiral\Tokenizer\ScopedEnumsInterface`.

Вот пример того, как вы можете искать перечисления в определенной области видимости:

```php
use Spiral\Tokenizer\ScopedEnumsInterface;

final class EnumSearcher
{
    public function __construct(
        private readonly ScopedEnumsInterface $locator
    ) {
    }

    public function pinpointEnums(): array
    {
        $foundEnums = [];

        foreach ($this->locator->getScopedEnums('scopeName') as $enum) {
            $foundEnums[] = $enum;
            // или любые другие операции, которые вы хотите...
        }

        return $foundEnums;
    }
}
```

:::

::: tab Interfaces

Аналогично вышесказанному, как только ваши области видимости готовы в файле конфигурации,
`Spiral\Tokenizer\ScopedInterfacesInterface` поможет вам искать интерфейсы в этих указанных зонах.

Вот как искать интерфейсы в назначенной области видимости:

```php
use Spiral\Tokenizer\ScopedInterfacesInterface;

final class InterfaceSearcher
{
    public function __construct(
        private readonly ScopedInterfacesInterface $locator
    ) {
    }

    public function findInterfaces(): array
    {
        $identifiedInterfaces = [];

        foreach ($this->locator->getScopedInterfaces('scopeName') as $interface) {
            $identifiedInterfaces[] = $interface;
            // или добавьте ваши желаемые действия...
        }

        return $identifiedInterfaces;
    }
}
```

:::

::::

## Эффективное сканирование с прослушивателями классов

Для больших кодовых баз регулярные сканирования с использованием локаторов Tokenizer могут замедлить процесс,
особенно когда вы регулярно ищете классы, перечисления или интерфейсы. Прослушиватели классов предоставляют более
умный подход, позволяя вам прослушивать и реагировать на обнаружения классов без повторного сканирования директорий.

### Зачем использовать прослушиватели классов?

Подумайте о том, что вам нужно искать в действительно большой библиотеке каждый раз, когда вы хотите найти книгу. Это
много работы. Теперь подумайте о том, что кто-то говорит вам всякий раз, когда новая книга появляется в библиотеке.
Это похоже на то, как работают прослушиватели классов. Вместо поиска в библиотеке каждый раз, вы получаете уведомление,
когда прибывает новая книга (класс). Это особенно полезно во время начальной загрузки приложения, где происходит
первоначальное сканирование, после чего прослушиватели остаются в курсе.

### Настройка прослушивателей

По умолчанию прослушиватели фокусируются на классах, но вы можете легко настроить их, чтобы расширить сеть и включить
перечисления и интерфейсы тоже. Для этого вам нужно добавить следующую конфигурацию в файл `app\config\tokenizer.php`:

```php app/config/tokenizer.php
return [
    'load' => [
        'classes' => true, // Поиск классов
        'enums' => true, // Поиск перечислений
        'interfaces' => true, // Поиск интерфейсов
    ],
];
```

> **Примечание**
> Помните, вам не нужно включать все три. Настройте это под потребности вашего проекта. Чем больше вы включаете, тем
> медленнее будет процесс.

### Использование

Чтобы использовать эту функцию, вам нужно включить загрузчик `Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader`
в ваш проект в верхней части списка загрузчиков:

:::: tabs

::: tab Using method

```php app/src/Application/Kernel.php
public function defineSystemBootloaders(): array
{
    return [
        \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
        // ...
    ];
}
```

Подробнее о загрузчиках читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::: tab Using constant

```php app/src/Application/Kernel.php
protected const SYSTEM = [
    \Spiral\Tokenizer\Bootloader\TokenizerListenerBootloader::class,
    // ...
];
```

Подробнее о загрузчиках читайте в разделе [Framework — Bootloaders](../framework/bootloaders.md).
:::

::::

### Создание прослушивателя

Следующий шаг — создать класс прослушивателя. Этот класс должен реализовывать
`Spiral\Tokenizer\TokenizationListenerInterface`, который требует два метода:

- `listen(\ReflectionClass $class)`: Этот метод вызывается каждый раз, когда обнаруживается класс. Вы можете добавить
  логику для обработки или хранения информации из этого класса.
- `finalize()`: Думайте об этом как о заключительном акте. Как только все классы сканированы и обработаны, этот метод
  вызывается. Это идеальное место для завершения дел или финализации операций на основе обнаруженных классов.

**Вот пример прослушивателя:**

```php
use Spiral\Attributes\ReaderInterface;

final class RouteAttributeListener implements TokenizationListenerInterface
{
    private array $attributes = [];

    public function __construct(
        private readonly ReaderInterface $reader,
        private readonly RouterInterface $router
    ) {
    }
    
    public function listen(\ReflectionClass $class): void
    {
        foreach ($class->getMethods() as $method) {
            $route = $this->reader->firstFunctionMetadata($method, Route::class);

            if ($route === null) {
                continue;
            }

            $this->attributes[] = [$method, $route];
        }
    }

    public function finalize(): void
    {
        foreach ($this->attributes as [$method, $route]) {
            $this->router->addRoute(...);
        }
    }
}
```

Этот прослушиватель, например, прослушивает классы с определенными атрибутами маршрутизации и добавляет их к
маршрутизатору, когда сканирование завершено.

### Кеширование целей прослушивателей

Чтобы улучшить производительность вашего приложения, вы можете использовать атрибуты
`Spiral\Tokenizer\Attribute\TargetAttribute` и `Spiral\Tokenizer\Attribute\TargetClass` для фильтрации классов и
атрибутов, которые обрабатываются прослушивателями. Это позволяет улучшить производительность вашего кода путем
фильтрации классов и атрибутов, которые обрабатываются прослушивателями.

Когда вы используете атрибуты для фильтрации классов, которые обрабатываются прослушивателями, компонент кеширует
отфильтрованные классы в директории `runtime/cache/listeners` после первой начальной загрузки вашего приложения.

Кеширование отфильтрованных классов предоставляет несколько преимуществ для вашего приложения. Оно значительно сокращает
время, необходимое для обработки вашей кодовой базы, поскольку локатор классов может загружать отфильтрованные классы из
кеша, а не повторно сканировать вашу кодовую базу каждый раз при запуске приложения. Это может помочь улучшить
производительность вашего приложения и сократить время, необходимое для начальной загрузки приложения.

По умолчанию кеширование отфильтрованных классов отключено. Если вы хотите включить кеширование, вы можете установить
переменную окружения `TOKENIZER_CACHE_TARGETS` в `true`.

```dotenv .env
TOKENIZER_CACHE_TARGETS=true
```

#### TargetAttribute

Он позволяет фильтровать классы на основе их атрибутов. Когда вы указываете целевой атрибут, локатор классов будет
обрабатывать только классы, которые имеют этот атрибут. Это может быть полезно, если у вас есть прослушиватель, который
нужно анализировать только определенный тип класса, например, класс контроллера, который имеет определенный атрибут
маршрутизации.

**Вот пример того, как его использовать:**

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;

#[TargetAttribute(Route::class, useAnnotations: true)]
final class RouteLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

В этом примере `RouteAttributeListener` будет обрабатывать только классы, которые имеют атрибут `Route`. Это означает,
что если локатор классов найдет класс без этого атрибута, он не вызовет метод `listen` этого прослушивателя.

Вы можете добавить несколько атрибутов к вашему классу прослушивателя:

```php
use Spiral\Tokenizer\Attribute\TargetAttribute;
use Spiral\Tokenizer\TokenizationListenerInterface;

#[TargetAttribute(Route::class)]
#[TargetAttribute(SymfonyRoute::class)]
class RouteLocatorListener implements TokenizationListenerInterface
{
    public function listen(\ReflectionClass $class): void
    {
        // Делайте что-то с классами, которые имеют атрибуты Route или SymfonyRoute
    }
}
```

Вы также можете передать второй параметр `useAnnotations: true` атрибуту, чтобы указать, что Tokenizer должен искать
целевой атрибут в аннотациях класса тоже.

Используйте `scanParents: true` для поиска классов, которые имеют целевой атрибут в их родительских классах или
интерфейсах.

#### TargetClass

Он работает аналогично `TargetAttribute`, но вместо фильтрации классов на основе их атрибутов, он фильтрует их на
основе их типа. Это полезно, если у вас есть прослушиватель, который нужно анализировать только определенный тип класса,
например, контроллер, классы, которые реализуют определенный интерфейс или расширяют определенный класс.

**Вот пример того, как использовать**

```php
use Spiral\Tokenizer\Attribute\TargetClass;

#[TargetClass(SymfonyCommand::class)]
final class CommandLocatorListener implements TokenizationListenerInterface
{
    // ...
}
```

В этом примере прослушиватель будет обрабатывать все классы, которые расширяют `SymfonyCommand`. Это означает, что если
локатор классов найдет класс, который его расширяет, он вызовет метод `listen` этого прослушивателя.

> **Примечание**
> Вы можете добавить несколько атрибутов к вашему классу прослушивателя.

### Регистрация прослушивателя

Чтобы зарегистрировать ваш прослушиватель, вам нужно использовать `Spiral\Tokenizer\TokenizerListenerRegistryInterface`.

Вот пример того, как зарегистрировать прослушиватель:

```php
use Spiral\Tokenizer\TokenizerListenerRegistryInterface;

class AppBootloader extends Bootloader
{
    public function init(
        TokenizerListenerRegistryInterface $listenerRegistry,
        RouteAttributeListener $listener
    ): void {
        $listenerRegistry->addListener($listener);
    }
}
```

> **Предупреждение**
> Чтобы убедиться, что ваши прослушиватели вызываются правильно, убедитесь, что регистрируете их в загрузчиках из
> раздела `LOAD` ядра приложения. Прослушиватели не будут вызваны, если вы зарегистрируете их в разделе `APP` ядра.

## Консольные команды

### Info

Хотите знать, как настроен tokenizer? Используйте команду `tokenizer:info`.

**Просто выполните следующую команду:**

```terminal
php app.php tokenizer:info
```

#### Что вы увидите

1. **Включенные директории:** Показывает, в каких папках tokenizer ищет.
2. **Исключенные директории:** Папки, которые tokenizer игнорирует.
3. **Загрузчики:** Говорит вам, какие виды PHP-объектов (как классы или интерфейсы) tokenizer ищет. Он также покажет,
   как включить или выключить их.
4. **Кеш tokenizer:** Показывает, есть ли ярлык (кеш), используемый для ускорения процесса. Вы можете включить или
   выключить это тоже.

#### Пример вывода

```terminal
Included directories:
+------------------------------------+-------+
| Directory                          | Scope |
+------------------------------------+-------+
| /vendor/intruforce/grpc-shared/src |       |
| app/                               |       |
+------------------------------------+-------+

Excluded directories:
+-----------+-------+
| Directory | Scope |
+-----------+-------+

Loaders:
+------------+------------------------------------------------------------------------------+
| Loader     | Status                                                                       |
+------------+------------------------------------------------------------------------------+
| Classes    | enabled                                                                      |
| Enums      | disabled. To enable, add "TOKENIZER_LOAD_ENUMS=true" to your .env file.      |
| Interfaces | disabled. To enable, add "TOKENIZER_LOAD_INTERFACES=true" to your .env file. |
+------------+------------------------------------------------------------------------------+

Tokenizer cache: disabled
To enable cache, add "TOKENIZER_CACHE_TARGETS=true" to your .env file.
```
