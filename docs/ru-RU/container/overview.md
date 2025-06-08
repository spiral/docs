# Контейнер — Обзор

Spiral включает набор инструментов, которые упрощают управление зависимостями и создание объектов в вашем коде. Одним из основных инструментов является контейнер, который помогает обрабатывать зависимости классов и автоматически "внедряет" их в класс. Это означает, что вместо создания объектов и настройки зависимостей вручную, контейнер позаботится об этом за вас.

## PSR-11 Container

Контейнер Spiral также следует набору стандартов [PSR-11 Container](https://github.com/php-fig/container), что обеспечивает совместимость с другими библиотеками.

В вашем коде вы можете получить доступ к контейнеру, запросив `Psr\Container\ContainerInterface`.

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function __construct(
        private readonly \Psr\Container\ContainerInterface $container
    ) {}

    public function show(string $id): void
    {
       $repository = $this->container->get(UserRepository::class);
       // ...
    }
}
```

## Внедрение зависимостей

Контейнер Spiral позволяет использовать как **конструкторное**, так и **методное** внедрение зависимостей для ваших классов. Это означает, что зависимости могут автоматически **внедряться** в класс через конструктор или через определенный метод.

:::: tabs

::: tab Конструктор

Например, в классе `UserController` зависимость `UserRepository` внедряется через метод `__construct()`. Это означает, что контейнер автоматически создаст и доставит объект UserRepository при вызове метода.

```php app/src/Endpoint/Web/UserController.php
use Psr\Container\ContainerInterface;

final class UserController
{
    public function __construct(
        private readonly UserRepository $users
    ) {}

    public function show(string $id): void
    {
       $user = $this->users->findOrFail($id);
       // ...
    }
}
```

:::

::: tab Метод

Например, в классе `UserController` зависимость `UserRepository` внедряется через метод `show()`. Это означает, что контейнер автоматически создаст и доставит объект UserRepository при вызове метода.

```php app/src/Endpoint/Web/UserController.php
final class UserController
{
    public function show(UserRepository $users, string $id): void
    {
       $user = $users->findOrFail($id);
       // ...
    }
}
```

:::
::::

> **Примечание**
> Эта функция доступна для классов, таких как контроллеры, консольные команды и задачи очереди.
> Дополнительно, контейнер также поддерживает расширенные возможности, такие как объединенные типы, вариативные аргументы, ссылочные параметры и значения объектов по умолчанию для авто-связывания.

### Автоматическое разрешение зависимостей

Контейнер фреймворка может автоматически разрешать зависимости конструктора или метода, предоставляя экземпляры конкретных классов.

```php
class MyController
{
    public function __construct(
        OtherClass $class, 
        SomeInterface $some
    ) {
    }
}
```

В приведенном примере контейнер попытается предоставить экземпляр `OtherClass`, автоматически сконструировав его. Однако `SomeInterface` не будет разрешен, если у вас нет соответствующей привязки в контейнере.

```php
$container->bind(SomeInterface::class, SomeClass::class); 
```

Обратите внимание, что контейнер попытается разрешить *все* зависимости конструктора (если вы не предоставите некоторые значения вручную). Это означает, что все зависимости классов должны быть доступны, или параметр должен быть объявлен как необязательный:

```php
// завершится ошибкой, если зависимость `value` не предоставлена
__construct(OtherClass $class, $value)

// будет использовать `null` как `value`, если другое значение не предоставлено
__construct(OtherClass $class, $value = null) 

// завершится ошибкой, если SomeInterface не указывает на конкретную реализацию
__construct(OtherClass $class, SomeInterface $some) 

// будет использовать null как значение `some`, если конкретная реализация не предоставлена
__construct(OtherClass $class, SomeInterface $some = null) 
```

## Сервисы

### Binder

Фреймворк предоставляет `Spiral\Core\BinderInterface`, который вы можете использовать для привязки класса к интерфейсу или псевдониму. Подробнее об этом читайте в разделе [Контейнер — Конфигурация](../container/configuration.md).

### Factory

Фреймворк предоставляет `Spiral\Core\FactoryInterface`, который вы можете использовать для создания класса без разрешения всех его зависимостей конструктора. Это может быть полезно в ситуациях, когда вам нужен только определенный подмножество зависимостей класса, или если вы хотите передать определенные значения для некоторых зависимостей.

```php
public function makeClass(FactoryInterface $factory): MyClass
{
    return $factory->make(UserService::class, [
        'table' => 'users'
    ]); 
}
```

В приведенном выше примере метод `make()` используется для создания экземпляра `UserService` и передачи значения `table` для зависимости параметра конструктора. Другие зависимости конструктора будут разрешены автоматически контейнером.

Это позволяет вам иметь больше контроля над созданием ваших классов и может быть особенно полезно в ситуациях, когда вам нужно создать несколько экземпляров класса с разными зависимостями конструктора.

### Resolver

Фреймворк предоставляет `Spiral\Core\ResolverInterface`, который вы можете использовать для разрешения аргументов методов для динамических целей, таких как методы контроллеров. Это может быть полезно в ситуациях, когда вы хотите вызвать метод и вам нужно разрешить его зависимости во время выполнения.

```php
abstract class Handler
{
    public function __construct(
        protected readonly ResolverInterface $resolver
    ) {
    }

    public function run(array $params): bool
    {
        $method = new \ReflectionMethod($this, 'do');

        return $method->invokeArgs(
            $this, 
            $this->resolver->resolveArguments($method, $params) // разрешить недостающие аргументы
        );
    }
}
```

Метод `run()` использует `ResolverInterface` для вызова метода `do` с внедрением метода. Теперь метод `do` может запросить внедрение метода:

```php
class UserStoreHandler extends Handler
{
    public function do(UserService $service): bool
    {
        // Сохранить пользователя
    }
}
```

Таким образом, вы можете легко разрешить зависимости метода во время выполнения и вызвать его с необходимыми аргументами, независимо от того, передаются ли зависимости в качестве параметров или их нужно разрешить контейнером.

#### Поддерживаемые типы

##### Объединенные типы

Реализация `ResolverInterface` по умолчанию поддерживает объединенные типы. Будет передана одна из доступных зависимостей нужного типа:

```php
use Doctrine\Common\Annotations\Reader;
use Spiral\Attributes\ReaderInterface;

final class Entities
{
    public function __construct(
        private Reader|ReaderInterface $reader
    ) {
    }
}
```

##### Вариативные аргументы

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int ...$bar) => $bar;

// массив, переданный по имени параметра
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => [1, 2]]
);

dump($args); // [1, 2]

// массив, переданный по имени параметра с именованными аргументами внутри
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => ['ab' => 1, 'bc' => 2]]
);

dump($args); // ['ab' => 1 'bc' => 2]

// значение, переданное по имени параметра
$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => 1]
);

dump($args); // [1]

```

##### Ссылочные аргументы

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$bar = 1;

$args = $resolver->resolveArguments(
    new \ReflectionFunction($function),
    ['bar' => &$bar]
);

$bar = 42;
dump($args); // [42]
```

##### Значение объекта по умолчанию

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(stdClass $std = new \stdClass()) => $std;

$args = $resolver->resolveArguments(new \ReflectionFunction($function));

dump($args); 

// array(1) {
//   [0] =>
//   class stdClass#369 (0) {
//   }
// }
```

#### Валидация аргументов

В некоторых случаях вы можете захотеть валидировать аргументы функции или метода. Для этого вы можете использовать публичный метод `validateArguments`, в который нужно передать `ReflectionMethod` или `ReflectionFunction` и `массив аргументов`. Если вы получили аргументы с помощью метода `resolveArguments` и не передали `false` в параметре `$validate`, они не нуждаются в дополнительной валидации. Они будут проверены автоматически. Если аргументы недействительны, будет выброшено исключение `Spiral\Core\Exception\Resolver\InvalidArgumentException`.

```php
$resolver = $this->container->get(ResolverInterface::class);
$function = static fn(int $bar) => $bar;

$resolver->validateArguments(new \ReflectionFunction($function), [42]);
```

### Invoker

Фреймворк предоставляет `Spiral\Core\InvokerInterface`, который вы можете использовать для вызова нужного метода с автоматическим разрешением его зависимостей. Это может быть полезно в ситуациях, когда вы хотите вызвать метод объекта и вам нужно разрешить его зависимости во время выполнения.

#### Вызов метода класса

InvokerInterface имеет метод `invoke()`, который вы можете использовать для вызова метода объекта и передачи определенных значений для его зависимостей.

Вот пример того, как вы можете его использовать:

```php
use Spiral\Core\InvokerInterface;

abstract class Handler
{
    public function __construct(
        protected readonly InvokerInterface $invoker
    ) {
    }

    public function run(array $params): bool
    {
        return $this->invoker->invoke([$this, 'do'], $params)
    }
}
```

Если вы передадите в качестве первого значения callable массива строку `['user-service', 'store']`, то она (`user-service`) будет запрошена из контейнера.

```php
$container->bind('user-service', UserService::class);
// ...
$invoker->invoke(
    ['user-service', 'store'], 
    $params
);
```

Контейнер разрешит класс `user-service` и вызовет метод `store` на нем.

Это позволяет вам легко вызывать методы классов, которые управляются контейнером, без необходимости вручную создавать их экземпляры. Это особенно полезно в ситуациях, когда вы хотите использовать класс как сервис и вызывать его методы из множественных частей вашего кода.

> **Примечание**
> Метод может иметь любую видимость (public, protected или private) и все еще быть вызванным.

#### Вызов callable

`InvokerInterface` также может использоваться для вызова замыканий и автоматического разрешения их зависимостей.

```php
$invoker->invoke(
    function (MyClass $class, string $parameter) {
        return new MyClassService($class);
    },
    [
        'parameter' => 'value',
    ]
); 
```

Это позволяет вам легко вызывать замыкания с необходимыми зависимостями, независимо от того, передаются ли зависимости в качестве параметров или их нужно разрешить контейнером.

### Области видимости (Scopes)

Важным аспектом разработки долгоживущих приложений является правильное управление контекстом. В демонизированных приложениях вам больше не разрешается обращаться с пользовательскими запросами как с глобальным объектом singleton и хранить ссылки на его экземпляр в ваших сервисах.

Практически это означает, что вы должны явно запрашивать контекст при обработке пользовательского ввода. Spiral предлагает простой способ управления этим, используя глобальный контейнер IoC (Инверсия управления). Он действует как носитель контекста, который позволяет вам запрашивать определенные экземпляры, ограниченные определенным контекстом, как если бы они были глобальными объектами.

Подробнее об **областях видимости** читайте в разделе [Контейнер — Области видимости IoC](../container/scopes.md).

## Замена контейнера приложения

В некоторых случаях вы можете захотеть заменить экземпляр контейнера в приложении. Вы можете сделать это при создании экземпляра приложения.

```php app.php
use Spiral\Core\Container;
use App\Application\Kernel;

$container = new Container();
$container->bind(...);

$app = Kernel::create(
    directories: ['root' => __DIR__],
    container: $container
)
```