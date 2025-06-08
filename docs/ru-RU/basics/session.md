# Основы — Сессии

Если вам нужно включить сессии в альтернативном пакете, установите composer пакет `spiral/session` и добавьте
bootloader `Spiral\Bootloader\Http\SessionBootloader` в ваше приложение.

## SessionInterface

Доступ к пользовательской сессии можно получить, используя контекстно-зависимый объект `Spiral\Session\SessionInterface`:

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Session\SessionInterface;

// ...

public function index(SessionInterface $session): void
{
    $session->resume();
    dump($session->getID());
}
```

> **Примечание**
> Вам не разрешается хранить ссылку на сессию в singleton объектах. См. обходное решение ниже.

## Секция сессии

По умолчанию вам не разрешается работать с сессией напрямую, вместо этого следует выделить изолированную и именованную секцию,
которая предоставляет классические методы `set`, `get`, `delete` или любую другую функциональность. Используйте `getSection` объекта сессии для
этих целей:

```php app/src/Endpoint/Web/HomeController.php
public function index(SessionInterface $session): void
{
    $cart = $session->getSection('cart');

    $cart->set('items', ['my-items']);

    dump($cart->getAll());
}
```

## Session Scope

Для упрощения использования сессии в singleton сервисах и контроллерах используйте `Spiral\Session\SessionScope`. Этот
компонент также доступен через свойство prototype `session`. Компонент может использоваться в singleton сервисах и
всегда указывает на активный контекст сессии:

```php app/src/Endpoint/Web/HomeController.php
use Spiral\Prototype\Traits\PrototypeTrait;

class HomeController
{
    use PrototypeTrait;

    public function index(): void
    {
        dump($this->session->getSection('cart')->getAll());
    }
}
```

## Жизненный цикл сессии

Сессия будет автоматически запущена при первом доступе к данным и зафиксирована когда запрос
покинет `SessionMiddleware`. Для ручного управления сессией используйте методы объекта `Spiral\Session\SessionInterface`.

> **Примечание**
> SessionScope полностью реализует SessionInterface.

### Возобновить сессию

Для ручного возобновления/создания сессии:

```php
$this->session->resume();
```

### Зафиксировать

Для ручной фиксации и закрытия сессии:

```php
$this->session->commit();
```

### Прервать

Для отмены всех изменений и закрытия сессии:

```php
$this->session->abort();
```

### Получить ID сессии

Для получения ID сессии (только когда сессия возобновлена):

```php
dump($this->session->getID());
```

Для проверки запущена ли сессия:

```php
dump($this->session->isStarted());
```

### Уничтожить

Для уничтожения сессии и всего содержимого:

```php
$this->session->destroy();
``` 

### Регенерировать ID

Для создания нового ID сессии без влияния на содержимое сессии:

```php
$this->session->regenerateID();
```

## Пользовательская конфигурация

Для изменения конфигурации сессии создайте файл `app/config/session.php` для изменения необходимых значений.

Компонент сессии основан на нативной реализации сессий PHP. По умолчанию содержимое сессии хранится в
файловой системе в директории `runtime/session`. Если ваше приложение будет балансировать нагрузку между несколькими веб-серверами,
вы должны выбрать централизованное хранилище, к которому все серверы смогут получить доступ, например Redis.

Параметр конфигурации `handler` сессии определяет где данные сессии будут храниться для каждого запроса.
Spiral поставляется с несколькими драйверами из коробки:

### Конфигурация **FileHandler**

Сессии хранятся в папке `runtime/session`.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\FileHandler;

return [
    'lifetime' => 86400,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        FileHandler::class,
        [
            'directory' => directory('runtime') . 'session',
            'lifetime'  => 86400
        ]
    )
];
```

### Конфигурация **CacheHandler**

Сессии хранятся в одном из хранилищ на основе кэша, настроенных в компоненте Cache.

```php app/config/session.php
use Spiral\Core\Container\Autowire;
use Spiral\Session\Handler\CacheHandler;

$ttl = 86400;

return [
    'lifetime' => $ttl,
    'cookie' => 'sid',
    'secure' => false,
    'handler' => new Autowire(
        CacheHandler::class,
        [
            'storage' => 'my-storage', // (Опционально) Название хранилища кэша. По умолчанию - текущее хранилище кэша
            'ttl' => $ttl,
            'prefix' => 'foo:' // (Опционально) По умолчанию, session:
        ]
    )
];
```

### Пользовательский обработчик сессий

Если ни один из существующих драйверов сессий не подходит потребностям вашего приложения, Spiral позволяет написать свой собственный обработчик сессий. Ваш пользовательский драйвер сессий должен реализовывать встроенный в PHP
интерфейс [`SessionHandlerInterface`](https://www.php.net/manual/en/class.sessionhandlerinterface.php).

```php app/config/session.php
return [
    'handler' => new Autowire(
        MemoryHandler::class,
        [
            'driver' => 'redis',
            'database' => 1,
            'lifetime' => 86400
        ]
    )
];
```

> **Примечание**
> Вы можете использовать Autowire вместо имени класса для настройки дополнительных параметров.

### Настройка инициализации сессии

Сессия инициализируется с помощью специальной фабрики `Spiral\Session\SessionFactoryInterface`.

```php
namespace Spiral\Session;

interface SessionFactoryInterface
{
    /**
     * @param string $clientSignature Специфичный для пользователя токен, не обеспечивает полную безопасность, но
     *                                     усложняет передачу сессии.
     * @param string|null $id Когда null - ожидается что php создаст сессию автоматически.
     */
    public function initSession(string $clientSignature, string $id = null): SessionInterface;
}
```

Вы можете заменить реализацию `Spiral\Session\SessionFactoryInterface` по умолчанию в контейнере на свою собственную.

```php
$container->bindSingleton(\Spiral\Session\SessionFactoryInterface::class, CustomSessionFactory::class);
```
