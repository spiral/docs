# Контейнер — Конфигурация

Контейнер Spiral позволяет настраивать его путем создания привязок между интерфейсами или псевдонимами к конкретным реализациям. Вы можете использовать [Bootloaders](../framework/bootloaders.md) для определения этих привязок.

## Обзор

Есть два способа настройки контейнера: либо с помощью `Spiral\Core\BinderInterface`, либо с помощью `Spiral\Core\Container`.

### Привязка интерфейса к реализации

Чтобы привязать интерфейс к конкретной реализации, вы можете использовать следующий фрагмент кода:

:::: tabs

::: tab BinderInterface

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::

::: tab Container

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->bind(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

:::
::::

### Привязка интерфейса к синглтону

Чтобы привязать синглтон, вы можете использовать следующий фрагмент кода:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        CycleUserRepository::class
    );
}
```

Вы также можете привязать с определенными параметрами, используя класс `Autowire` следующим образом:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        new Autowire(CycleUserRepository::class, ['table' => 'users'])
    );
}
```

Наконец, вы можете использовать замыкания для автоматической настройки вашего класса, передав замыкание в метод bind или `bindSingleton` следующим образом:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn() => new CycleUserRepository(table: users)
    );
}
```

Замыкания также поддерживают зависимости:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\Autowire;

public function boot(BinderInterface $binder): void
{
    $binder->bindSingleton(
        UserRepositoryInterface::class, 
        static fn(UserConfig $config) => new CycleUserRepository(table: $config->getTable())
    );
}
```

Когда это замыкание выполняется, контейнер автоматически разрешит экземпляр `UserConfig` и передаст его в качестве аргумента замыканию. Это позволяет вам легко настраивать ваши классы с зависимостями без необходимости вручную создавать и управлять ими.

### Проверка наличия привязки в контейнере

Чтобы проверить, есть ли в контейнере привязка, используйте:

```php
use Spiral\Core\Container;

public function boot(Container $container): void
{
    $container->has(UserRepositoryInterface::class)
}
```

### Удаление привязки

Чтобы удалить привязку контейнера:

```php
use Spiral\Core\BinderInterface;

public function boot(BinderInterface $binder): void
{
    $binder->removeBinding(UserRepositoryInterface::class)
}
```

Контейнер поддерживает привязку `WeakReference`:

:::: tabs

::: tab Строковый псевдоним

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind('test-alias', $reference);
    
    dump($hash === \spl_object_hash($container->get('test-alias'))); // true
    
    unset($object);
    // Новый объект не может быть создан, поскольку имя класса не было сохранено
    dump($container->get('test-alias')); // null
}
```

:::

::: tab Псевдоним имени класса

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Core\Container;
use WeakReference;

public function boot(Container $container): void
{
    $object = new stdClass();
    $hash = \spl_object_hash($object);
    $reference = WeakReference::create($object);

    $container->bind(stdClass::class, $reference);
    
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // true
    
    unset($object);
    // новый экземпляр создан с использованием псевдонима класса
    dump($hash === \spl_object_hash($container->get(stdClass::class))); // false
}
```

:::
::::

## Расширенные привязки

Начиная с версии **3.8.0 Spiral Framework** мы заменили предыдущую структуру на основе массивов для хранения информации о привязках внутри контейнера. Новый подход использует объекты передачи данных (DTO), предоставляя более структурированный и организованный способ хранения информации о привязках. С этим изменением разработчики теперь могут легко настраивать привязки контейнера, используя эти объекты.

Чтобы продемонстрировать улучшенную функциональность контейнера, вот пример, демонстрирующий новую конфигурацию привязки:

```php
use Spiral\Core\Config\Factory;

$container->bind(LoggerInterface::class, new Factory(
    callable: static function() {
        return new Logger(....);
    }, 
    singleton: true
))
```

Вот доступные DTO привязок:

### Alias

Этот упрощенный DTO позволяет создать ссылку на другую привязку внутри контейнера.

```php
use Spiral\Core\Config\Alias;

$container->bind(
    \DatetimeImmutable::class, 
    static fn() => new \DatetimeImmutable()
);

$container->bind(
    'now', 
    new Alias(alias: \DatetimeImmutable::class)
);
```

В этом примере мы сначала привязываем класс `\DatetimeImmutable` к фабричной функции, которая создает новый экземпляр `\DatetimeImmutable` каждый раз, когда он запрашивается. Затем мы создаем привязку Alias с именем now и связываем её с привязкой класса `\DatetimeImmutable`.

Теперь вы можете запросить класс `\DatetimeImmutable` из контейнера по псевдониму $container->get('now')

Alias также предоставляет второй аргумент - `singleton`. Установив singleton в `true`, класс с псевдонимом становится синглтоном. Это означает, что всякий раз, когда вы запрашиваете `$container->get('now')` из контейнера, он будет возвращать один и тот же экземпляр каждый раз. С другой стороны, вызов `$container->get(\DatetimeImmutable::class)` будет возвращать новый экземпляр `\DatetimeImmutable` с текущим временем при каждом запросе.

### Autowire

Привязка `Spiral\Core\Config\Autowire` служит оберткой для класса `Spiral\Core\Container\Autowire`, предоставляя автоматизированный способ разрешения и создания экземпляров классов путем внедрения их зависимостей.

```php
use Spiral\Core\Config\Autowire;
use Spiral\Core\Container\Autowire as AutowireAlias;

$container->bind(MyClass::class, new Autowire(
    autowire: new AutowireAlias(MyClass::class, ['foo' => 'bar']),
    singleton: true
));
```

Аргумент `singleton`, установленный в true в этом примере, указывает, что контейнер должен рассматривать `MyClass` как синглтон. Когда вы запрашиваете экземпляр MyClass из контейнера `$container->get(MyClass::class)`, он будет возвращать один и тот же экземпляр каждый раз.

### Factory

Привязка `Spiral\Core\Config\Factory` служит простой фабрикой для создания смешанных типов с использованием замыкания.

```php
use Spiral\Core\Config\Factory;

$container->bind('time', new Factory(
    callable: static fn() => time(),
));
```

Каждый раз, когда вы запрашиваете текущее время из контейнера `$container->get('time')`, он будет возвращать текущую временную метку.

Дополнительно, привязка `Factory` может быть настроена как синглтон:

```php
$container->bind('time', new Factory(
    callable: static fn() => time(),
    singleton: true,
));
```

Установив аргумент `singleton` в `true`. Это означает, что всякий раз, когда вы запрашиваете значение времени из контейнера `$container->get('time')`, он будет возвращать одно и то же значение при множественных вызовах.

### DeferredFactory

Привязка `Spiral\Core\Config\DeferredFactory` позволяет привязать отложенную фабрику к контейнеру, используя специальный массив callable.

```php
use Spiral\Core\Config\DeferredFactory;

$container->bind('some-binding', new DeferredFactory(
    factory: [MyClass::class, 'handle'],
));
```

В приведенном выше примере мы привязываем ключ some-binding к экземпляру `DeferredFactory`. Свойство factory установлено в `[MyClass::class, 'handle']`. Когда ключ some-binding запрашивается из контейнера `$container->get('some-binding')`, `DeferredFactory` разрешит экземпляр `MyClass` и затем вызовет метод handle на этом экземпляре для получения требуемого значения.

Дополнительно, привязка может быть настроена как синглтон:

```php
$container->bind('some-binding', new DeferredFactory(
    factory: ...,
    singleton: true
));
```

### Scalar

Привязка Scalar — это новая функциональность, введенная для контейнера. Она предоставляет удобный способ хранения и получения статических скалярных значений внутри контейнера. Это может быть полезно для настройки путей, констант или любых других скалярных значений, необходимых вашему приложению.

```php
use Spiral\Core\Config\Scalar;

$container->bind('app-path', new Scalar(value: '/var/www/my-app'));
```

### Shared

Привязка Shared позволяет привязать постоянный объект к ключу в контейнере. После создания объекта он будет переиспользоваться каждый раз, когда ключ запрашивается, и пользовательские аргументы не могут быть предоставлены при последующих запросах.

```php
use Spiral\Core\Config\Shared;

$container->bind(MyClass::class, new Shared(value: new MyClass(...)));
```

Важно отметить, что с привязкой `Shared` пользовательские аргументы не могут быть предоставлены при последующих запросах. Объект будет создан с исходными аргументами и переиспользован как есть.

Это особенно полезно, когда вы хотите обеспечить использование одного и того же экземпляра объекта во всем приложении. Это обеспечивает постоянство и предотвращает создание множественных экземпляров с разными аргументами.

### Inflector

Инфлектор позволяет манипулировать объектом после его создания в контейнере. Это особенно полезно для применения общих модификаций или внедрений к объектам определенного типа.

```php
use Spiral\Core\Config\Inflector;

$container->bind(LoggerAwareInterface::class, new Inflector(
    inflector: static function (LoggerAwareInterface $obj, LoggerInterface $logger): LoggerAwareInterface {
        $obj->setLogger($logger);

        return $obj;
    }
));
```

В этом случае любой объект, реализующий `LoggerAwareInterface`, будет иметь свой логгер установленным на основе указанной конфигурации.

Привязка Inflector — это мощный инструмент для применения общих модификаций или внедрений последовательно к объектам в вашем приложении. Это упрощает процесс настройки и кастомизации объектов, получаемых из контейнера.

### WeakReference

Функция WeakReference позволяет работать с объектами слабых ссылок внутри контейнера. Слабые ссылки — это ссылки на объект, которые не препятствуют сборке мусора объекта, когда на него нет сильных ссылок.

```php
use Spiral\Core\Config\WeakReference;

$obj = new MyClass();

$container->bind(MyClass::class, new WeakReference(
    reference: new \WeakReference($obj)
));

$obj === $container->get(MyClass::class); // true

unset($obj);

$obj1 = $container->get(MyClass::class); // Будет создан новый объект
$obj1 === $container->get(MyClass::class); // true
```

Когда вы получаете MyClass из контейнера `$container->get(MyClass::class)`, контейнер возвращает исходный объект, поскольку он все еще существует. Однако, когда вы удаляете переменную `$obj`, убирая сильную ссылку на объект, он становится пригодным для сборки мусора. Последующие вызовы `$container->get(MyClass::class)` будут создавать новый экземпляр `MyClass`, поскольку исходный объект был собран мусорщиком.

Использование слабых ссылок может быть полезным в определенных сценариях, где вы хотите иметь контроль над жизненным циклом объекта и позволить ему быть собранным мусорщиком, когда больше нет сильных ссылок на него.

## Ленивые синглтоны

Фреймворк также позволяет использовать "ленивые синглтоны" — классы, которые автоматически рассматриваются как синглтоны контейнером без необходимости явно привязывать их как таковые.

:::: tabs

::: tab Атрибут
`Spiral\Core\Attribute\Singleton` позволяет пометить класс как синглтон. Применяя этот атрибут к классу, вы указываете, что должен быть создан только один экземпляр класса, который разделяется по всему приложению. Этот атрибут может использоваться как альтернатива интерфейсам для указания поведения синглтона.

```php
use Spiral\Core\Attribute\Singleton;

#[Singleton]
final class UserService
{
    public function store(User $user): void
    {
        //...
    }
}
```

> **Примечание**
> Подробнее об атрибутах контейнера читайте в разделе [Контейнер - Атрибуты](./attributes.md).

:::

::: tab SingletonInterface

Чтобы использовать эту функцию, вы можете просто реализовать интерфейс `Spiral\Core\Container\SingletonInterface` в вашем классе следующим образом:

```php
use Spiral\Core\Container\SingletonInterface;

final class UserService implements SingletonInterface
{
    public function store(User $user): void
    {
        //...
    }
}
```

:::
::::

Теперь контейнер автоматически будет рассматривать этот класс как синглтон и создаст только один экземпляр для всего приложения, независимо от того, сколько раз он запрашивается.

```php
protected function index(UserService $service): void
{
    dump($this->container->get(UserService::class) === $service);
}
```