# Контейнер — Инжекторы

Spiral предоставляет способ контролировать процесс создания любого интерфейса или наследников абстрактного класса с помощью интерфейса инжектора.

**Есть несколько преимуществ его использования:**

- **Гибкость:** Интерфейс инжектора позволяет создавать определенный класс на основе определенного контекста, обеспечивая высокую степень гибкости в создании экземпляров классов.
- **Разделение:** Интерфейс инжектора позволяет разделить классы от их зависимостей, делая код более модульным и легким в обслуживании.
- **Контроль над созданием экземпляров:** Интерфейс инжектора предоставляет способ контролировать создание экземпляров классов, упрощая управление жизненным циклом объектов и обеспечивая их правильную инициализацию.

Это руководство демонстрирует, как создать экземпляр класса и присвоить ему уникальное значение, независимо от того, какие наследники его реализуют.

## Инжекторы классов

Давайте представим, что у нас есть интерфейс `Psr\SimpleCache\CacheInterface`, который предоставляет простой интерфейс кэша.

### Создание инжектора

Класс инжектора должен реализовывать `Spiral\Core\Container\InjectorInterface`, который предоставляет метод под названием `createInjection`. Этот метод используется каждый раз, когда определенный класс запрашивается из контейнера.

В нашем примере мы можем объединить загрузчик (bootloader) и инжектор в один экземпляр:

```php app/src/Application/Bootloader/CacheBootloader.php
namespace App\Application\Bootloader;

use Psr\Container\ContainerInterface;
use Psr\SimpleCache\CacheInterface;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Core\BinderInterface;
use Spiral\Core\Container\InjectorInterface;

class CacheBootloader extends Bootloader implements InjectorInterface
{
    public function __construct(
        private readonly ContainerInterface $container,
    ) {}

    public function boot(BinderInterface $binder): void
    {
        // Зарегистрировать внедряемый класс
        $binder->bindInjector(CacheInterface::class, self::class);
    }

    public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
    {
        return match ($context) {
            'redis' => new RedisCache(...),
            'memcached' => new MemcachedCache(...),
            default => new ArrayCache(...),
        };
    }
}
```

> **Примечание**
> Не забудьте активировать загрузчик (bootloader).

### Использование инжектора

Теперь мы можем использовать инжектор для получения определенной реализации кэша на основе контекста.

Когда контейнер разрешает `CacheInterface`, он запросит его у инжектора, используя метод `createInjection`. Метод принимает два аргумента: `$class` и `$context`. Аргумент `$class` возвращает объект `ReflectionClass` для запрашиваемого класса, а аргумент `$context` возвращает имя параметра или псевдонима (например, имя аргумента метода или функции, которая запросила внедряемый класс).

Вот пример того, как использовать инжектор:

```php app/src/Endpoint/Web/BlogController.php
namespace App\Endpoint\Web;

use Psr\SimpleCache\CacheInterface;

class BlogController
{
    public function __construct(
        private readonly CacheInterface $redis,
        private readonly CacheInterface $memcached,
        private readonly CacheInterface $cache,
    } {
        
        \assert($redis instanceof RedisCache);

        \assert($memcached instanceof MemcachedCache);
        
        \assert($cache instanceof ArrayCache);
    }
}
```

В этом примере класс `BlogController` принимает три свойства: `$redis`, `$memcached` и `$cache`. Все эти свойства имеют тип `CacheInterface`, но они соответствуют разным реализациям интерфейса на основе контекста, который был передан методу `createInjection` инжектора.

### Наследование классов

Наследование классов возможно с инжектором.

> **Примечание**
> В настоящее время инжектор поддерживает только классы (не интерфейсы), которые наследуют базовый класс, но будущие релизы Spiral также будут поддерживать наследование интерфейсов.

```php
abstract class RedisCacheInterface implements CacheInterface
{

}
```

В этом примере `RedisCacheInterface` является абстрактным классом, который реализует `CacheInterface`.

Например, в методе `createInjection` мы можем проверить, является ли запрашиваемый класс подклассом `RedisCacheInterface` и вернуть экземпляр `RedisCache`, или проверить, является ли запрашиваемый класс подклассом `MemcachedCacheInterface` и вернуть экземпляр `MemcachedCache`.

```php app/src/Application/Bootloader/CacheBootloader.php
public function createInjection(\ReflectionClass $class, string $context = null): CacheInterface
{
    if ($class->isSubclassOf(RedisCacheInterface::class)) {
        return new RedisCache(...);
    }
    
    return match ($context) {
        'redis' => new RedisCache(...),
        'memcached' => new MemcachedCache(...),
        default => new ArrayCache(...),
    };
}
```

## Инжекторы Enum

Компонент `spiral/boot` предоставляет удобный способ разрешения классов enum с использованием интерфейса `Spiral\Boot\Injector\InjectableEnumInterface`. Когда контейнер запрашивает Enum, он вызовет указанный метод для определения текущего значения Enum и внедрит любые необходимые зависимости.

**Есть несколько преимуществ использования внедрений Enum:**

- **Типобезопасность:** Внедрения обеспечивают типобезопасность, гарантируя, что правильный тип переменной передается методу или классу.
- **Динамическое разрешение:** Внедрения позволяют динамически разрешать значения Enum на основе текущего состояния приложения, это упрощает изменение значения Enum без необходимости вручную обновлять его в нескольких местах по всему приложению.
- **Переиспользование:** Внедрения могут быть переиспользованы в разных частях приложения, делая более эффективным управление созданием экземпляров Enum.
- **Улучшенная читаемость:** Внедрения делают код более читаемым и самообъясняющим, предоставляя четкое значение экземпляров Enum, используемых в коде.
- **Улучшенная поддерживаемость:** Внедрения делают код более поддерживаемым, поскольку переменные централизованы в одном месте и могут быть легко управляемы.

Давайте создадим `Enum`, с которым мы можем легко определить окружение нашего приложения.

```php
use Spiral\Boot\EnvironmentInterface;
use Spiral\Boot\Injector\ProvideFrom;
use Spiral\Boot\Injector\InjectableEnumInterface;

#[ProvideFrom(method: 'detect')]
enum AppEnvironment: string implements InjectableEnumInterface
{
    case Production = 'prod';
    case Stage = 'stage';
    case Testing = 'testing';
    case Local = 'local';

    public function isProduction(): bool
    {
        return $this === self::Production;
    }

    public function isTesting(): bool
    {
        return $this === self::Testing;
    }

    public function isLocal(): bool
    {
        return $this === self::Local;
    }

    public function isStage(): bool
    {
        return $this === self::Stage;
    }

    public static function detect(EnvironmentInterface $environment): self
    {
        $value = $environment->get('APP_ENV');

        return \is_string($value)
            ? (self::tryFrom($value) ?? self::Local)
            : self::Local;
    }
}
```

Атрибут `ProvideFrom` используется для указания метода detect, который используется для определения текущего значения Enum. Метод будет вызван, когда контейнер запросит Enum, и любые необходимые зависимости из контейнера будут внедрены в него.

### Использование

Контейнер автоматически внедрит Enum с правильным значением.

```php app/src/Endpoint/Console/MigrateCommand.php
final class MigrateCommand extends Command 
{
     const NAME = '...';

     public function perform(AppEnvironment $appEnv): int
     {
           if ($appEnv->isProduction()) {
                 // Запретить
           }
           
           // Выполнить миграцию
     }
}
```