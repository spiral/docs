# Framework — Перехватчики (Interceptor)

Одной из ключевых особенностей Spiral является поддержка перехватчиков, которые могут использоваться для добавления функциональности к приложению без изменения основного кода приложения. Это может помочь сохранить вашу кодовую базу более модульной и поддерживаемой.

**Преимущества использования перехватчиков:**

- **Разделение ответственности:** Использование перехватчиков позволяет вам держать различные части вашего приложения отдельными и организованными. Например, вы можете использовать перехватчик для обработки аутентификации без необходимости добавлять этот код в каждую часть вашего приложения, которая требует аутентификации.
- **Повторное использование:** С перехватчиками вы можете написать код один раз и использовать его в нескольких частях вашего приложения, уменьшая дублирование кода.
- **Модульность:** Возможность добавлять, удалять или заменять перехватчики без влияния на остальную часть приложения делает его более гибким и легким для обновления.
- **Производительность:** Перехватчики могут использоваться для оптимизации производительности приложения путем кэширования ответов, уменьшения количества запросов к базе данных и многого другого.
- **Простота использования:** Добавление перехватчиков в ваше приложение относительно легко и просто.

Вы можете использовать перехватчики с различными компонентами, такими как:

- [HTTP](../http/interceptors.md)
- [События](../advanced/events.md#interceptors)
- [gRPC](../grpc/interceptors.md)
- [Websockets](../websockets/interceptors.md)
- [Очереди](../queue/interceptors.md)
- [Temporal](../temporal/interceptors.md)

## Интерфейс перехватчика

Перехватчики реализуют `Spiral\Interceptors\InterceptorInterface`:

```php
namespace Spiral\Interceptors;

use Spiral\Interceptors\Context\CallContextInterface;

interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

Интерфейс предоставляет гибкий способ перехвата вызовов к любой цели, будь то метод, функция или пользовательский обработчик.

## Контекст вызова

`CallContextInterface` содержит всю информацию о перехваченном вызове:

- **Target** — определение цели вызова (метод, функция, замыкание и т.д.)
- **Arguments** — список аргументов для вызова
- **Attributes** — дополнительный контекст, который может использоваться для передачи данных между перехватчиками

```php
namespace Spiral\Interceptors\Context;

interface CallContextInterface extends AttributedInterface
{
    public function getTarget(): TargetInterface;
    public function getArguments(): array;
    public function withTarget(TargetInterface $target): static;
    public function withArguments(array $arguments): static;
    
    // Методы из AttributedInterface:
    public function getAttributes(): array;
    public function getAttribute(string $name, mixed $default = null): mixed;
    public function withAttribute(string $name, mixed $value): static;
    public function withoutAttribute(string $name): static;
}
```

> **Примечание**
> `CallContextInterface` является неизменяемым, поэтому вызовы `withTarget()` и `withArguments()` возвращают новый экземпляр с обновленными значениями.

## Интерфейс цели

`TargetInterface` определяет цель, чей вызов вы хотите перехватить. Он может представлять метод, функцию, замыкание или даже строку пути для RPC или конечных точек очереди сообщений.

```php
namespace Spiral\Interceptors\Context;

interface TargetInterface extends \Stringable
{
    public function getPath(): array;
    public function withPath(array $path, ?string $delimiter = null): static;
    public function getReflection(): ?\ReflectionFunctionAbstract;
    public function getObject(): ?object;
    public function getCallable(): callable|array|null;
}
```

### Создание целей

Статические фабричные методы в классе `Target` упрощают создание различных типов целей:

```php
use Spiral\Interceptors\Context\Target;

// Из рефлексии метода
$target = Target::fromReflectionMethod(new \ReflectionMethod(UserController::class, 'show'), UserController::class);

// Из рефлексии функции
$target = Target::fromReflectionFunction(new \ReflectionFunction('array_map'));

// Из замыкания
$target = Target::fromClosure(fn() => 'Hello, World!');

// Из строки пути (для RPC конечных точек или обработчиков очереди сообщений)
$target = Target::fromPathString('user.show');

// Из пары контроллер-действие
$target = Target::fromPair(UserController::class, 'show');
```

## Создание перехватчика

Давайте создадим простой перехватчик, который регистрирует время выполнения вызова:

```php
namespace App\Interceptor;

use Psr\Log\LoggerInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

class ExecutionTimeInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly LoggerInterface $logger
    ) {
    }

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        $target = $context->getTarget();
        $startTime = \microtime(true);

        try {
            return $handler->handle($context);
        } finally {
            $executionTime = \microtime(true) - $startTime;

            $this->logger->debug(
                'Target executed',
                [
                    'target' => (string)$target,
                    'execution_time' => $executionTime,
                ]
            );
        }
    }
}
```

Этот перехватчик:
1. Записывает время начала
2. Передает вызов следующему обработчику в цепочке
3. Вычисляет и регистрирует время выполнения после завершения обработчика

## Построение конвейера перехватчиков

Для использования перехватчиков необходимо построить конвейер перехватчиков, используя `PipelineBuilderInterface`:

```php
use Spiral\Interceptors\PipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\CallableHandler;
use App\Interceptor\ExecutionTimeInterceptor;
use App\Interceptor\AuthorizationInterceptor;

// Создание перехватчиков
$interceptors = [
    new ExecutionTimeInterceptor($logger),
    new AuthorizationInterceptor($auth),
];

// Построение конвейера
$pipeline = (new PipelineBuilder())
    ->withInterceptors(...$interceptors)
    ->build(new CallableHandler());

// Создание контекста вызова
$context = new CallContext(
    target: Target::fromPair(UserController::class, 'show'),
    arguments: [42],
    attributes: ['request' => $request]
);

// Выполнение конвейера
$result = $pipeline->handle($context);
```

## Обработчики

Конвейер должен заканчиваться обработчиком, который выполняет цель. Spiral предоставляет несколько встроенных обработчиков:

### CallableHandler

`CallableHandler` просто вызывает цель без какой-либо дополнительной обработки:

```php
use Spiral\Interceptors\Handler\CallableHandler;

$handler = new CallableHandler();
```

### AutowireHandler

`AutowireHandler` разрешает недостающие аргументы, используя контейнер:

```php
use Spiral\Interceptors\Handler\AutowireHandler;

$handler = new AutowireHandler($container);
```

Этот обработчик полезен при работе с контроллерами, где вы хотите автоматически внедрять зависимости сервисов.

## Расширенный пример

Вот более полный пример использования перехватчиков:

```php
namespace App\Controller;

use App\Interceptor\AuthorizationInterceptor;
use App\Interceptor\CacheInterceptor;
use App\Interceptor\ExecutionTimeInterceptor;
use App\User\UserService;
use Psr\Container\ContainerInterface;
use Spiral\Core\Attribute\Proxy;
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Interceptors\Context\CallContext;
use Spiral\Interceptors\Context\Target;
use Spiral\Interceptors\Handler\AutowireHandler;

class UserController
{
    private $pipeline;

    public function __construct(
        private readonly UserService $userService,
        #[Proxy] ContainerInterface $container
    ) {
        // Построение конвейера с перехватчиками
        $this->pipeline = (new CompatiblePipelineBuilder())
            ->withInterceptors(
                new ExecutionTimeInterceptor($container->get(LoggerInterface::class)),
                new AuthorizationInterceptor($container->get(AuthInterface::class)),
                new CacheInterceptor($container->get(CacheInterface::class))
            )
            ->build(new AutowireHandler($container));
    }

    public function show(int $id)
    {
        // Создание контекста для целевого метода
        $context = new CallContext(
            target: Target::fromReflectionMethod(
                new \ReflectionMethod($this->userService, 'findUser'),
                $this->userService
            ),
            arguments: ['id' => $id]
        );

        // Выполнение конвейера
        return $this->pipeline->handle($context);
    }
}
```

В этом примере:
1. Мы строим конвейер с логированием времени выполнения, проверками авторизации и кэшированием результатов
2. Мы используем `AutowireHandler` для разрешения недостающих аргументов из контейнера
3. Метод контроллера создает контекст, нацеленный на метод `findUser` сервиса `UserService`
4. Конвейер выполняет все перехватчики, а затем вызывает целевой метод

> **Примечание**
> Чтобы узнать об областях видимости контейнера и прокси-объектах, см. раздел [Области видимости IoC](../container/scopes.md) в нашей документации.

## Сравнение с устаревшими перехватчиками

> **Примечание**
> Старая реализация перехватчиков на основе `spiral/hmvc` больше не рекомендуется.
> Вы можете найти старую документацию на [https://spiral.dev/docs/framework-interceptors/3.13](https://spiral.dev/docs/framework-interceptors/3.13).

В Spiral 3.14.0 была введена новая реализация перехватчиков в пакете `spiral/interceptors`.
Вот как новая реализация отличается от устаревшей:

:::: tabs

::: tab Устаревшие перехватчики
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Core\CoreInterface;
use Spiral\Core\CoreInterceptorInterface;

class CacheInterceptor implements CoreInterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed
    {
        // Шаг 1: Генерация ключа кэша на основе контроллера, действия и параметров
        $cacheKey = $this->generateCacheKey($controller, $action, $parameters);

        // Шаг 2: Проверка, есть ли результат уже в кэше
        if ($this->cache->has($cacheKey)) {
            // Возврат кэшированного результата, если доступен
            return $this->cache->get($cacheKey);
        }

        // Шаг 3: Выполнение действия контроллера, если нет кэшированного результата
        $result = $core->callAction($controller, $action, $parameters);

        // Шаг 4: Кэширование результата для будущих запросов
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(string $controller, string $action, array $parameters): string
    {
        // Создание детерминистического ключа кэша из контроллера, действия и параметров
        return \md5($controller . '::' . $action . '::' . \serialize($parameters));
    }

    private function isCacheable(mixed $result): bool
    {
        // Кэширование только сериализуемых результатов
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::: tab Новые перехватчики
```php
namespace App\Interceptor;

use Psr\SimpleCache\CacheInterface;
use Spiral\Interceptors\Context\CallContextInterface;
use Spiral\Interceptors\Context\TargetInterface;
use Spiral\Interceptors\HandlerInterface;
use Spiral\Interceptors\InterceptorInterface;

final class CacheInterceptor implements InterceptorInterface
{
    public function __construct(
        private readonly CacheInterface $cache,
        private readonly int $ttl = 3600,
    ) {}

    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed
    {
        // Шаг 1: Генерация ключа кэша, используя путь цели и аргументы
        $cacheKey = $this->generateCacheKey($context->getTarget(), $context->getArguments());

        // Шаг 2: Проверка, есть ли результат уже в кэше
        if ($this->cache->has($cacheKey)) {
            // Возврат кэшированного результата, если доступен
            return $this->cache->get($cacheKey);
        }

        // Шаг 3: Выполнение цели, если нет кэшированного результата
        $result = $handler->handle($context);

        // Шаг 4: Кэширование результата для будущих запросов
        if ($this->isCacheable($result)) {
            $this->cache->set($cacheKey, $result, $this->ttl);
        }

        return $result;
    }

    private function generateCacheKey(TargetInterface $target, array $args): string
    {
        // Создание детерминистического ключа кэша из строки цели и аргументов
        return \md5((string) $target . '::' . serialize($args));
    }

    private function isCacheable(mixed $result): bool
    {
        // Кэширование только сериализуемых результатов
        return !\is_resource($result) && (
                \is_scalar($result) ||
                \is_array($result) ||
                $result instanceof \Serializable ||
                $result instanceof \stdClass
            );
    }
}
```
:::

::::

### Изменения интерфейса

**Устаревший интерфейс:**
```php
interface CoreInterceptorInterface
{
    public function process(string $controller, string $action, array $parameters, CoreInterface $core): mixed;
}
```

**Новый интерфейс:**
```php
interface InterceptorInterface
{
    public function intercept(CallContextInterface $context, HandlerInterface $handler): mixed;
}
```

Основные различия:
- Параметры `$controller` и `$action` были заменены более гибкой концепцией `Target`
- Массив `$parameters` теперь является частью `CallContext`
- Вместо `CoreInterface` есть `HandlerInterface`, который предлагает больше гибкости

### Совместимость

В Spiral 3.x поддерживаются как устаревшие перехватчики (`CoreInterceptorInterface`), так и новые перехватчики (`InterceptorInterface`).
Однако устаревшая реализация не рекомендуется и будет исключена в Spiral 4.x.

Если вам нужно использовать обе реализации вместе, используйте `CompatiblePipelineBuilder`:

```php
use Spiral\Core\CompatiblePipelineBuilder;
use Spiral\Core\CoreInterceptorInterface; // Устаревший перехватчик
use Spiral\Interceptors\InterceptorInterface; // Новый перехватчик

$pipeline = (new CompatiblePipelineBuilder())
    ->withInterceptors(
        new LegacyStyleInterceptor(), // Реализует CoreInterceptorInterface
        new NewStyleInterceptor()     // Реализует InterceptorInterface
    )
    ->build(new CallableHandler());
```

### Рекомендации по миграции

При миграции с устаревшей реализации:
- Замените `CoreInterceptorInterface` на `InterceptorInterface`
- Используйте `Target` вместо параметров `$controller` и `$action`
- Переместите `$parameters` в `CallContext`
- Замените `$core->callAction()` на `$handler->handle()`

## События

| Событие                                      | Описание                                        |
|----------------------------------------------|-------------------------------------------------|
| Spiral\Interceptors\Event\InterceptorCalling | Срабатывает перед вызовом перехватчика.         |

> **Предупреждение**
> Событие `Spiral\Core\Event\InterceptorCalling` отправляется только устаревшим `\Spiral\Core\InterceptorPipeline` из устаревшей реализации. Новая реализация фреймворка не использует это событие.

> **Примечание**
> Чтобы узнать больше о диспетчеризации событий, см. раздел [События](../advanced/events.md) в нашей документации.
