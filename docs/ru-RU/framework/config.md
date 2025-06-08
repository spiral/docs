# Framework — Объекты конфигурации

Spiral использует объекты конфигурации для разделения фаз загрузки и выполнения приложения, а также для предоставления легко доступного источника конфигурации.

Основное преимущество использования объектов конфигурации заключается в том, что они позволяют легко изменять конфигурацию во время процесса загрузки, при этом гарантируя, что конфигурация остается неизменной во время выполнения. Это помогает предотвратить неожиданные изменения конфигурации и обеспечивает более предсказуемое поведение приложения.

![Application Control Phases](https://user-images.githubusercontent.com/67324318/186413037-e60f89fd-9313-44c5-b4f8-eb2585a77230.png)

**Вот некоторые преимущества использования объектов конфигурации:**

- Объекты конфигурации позволяют легко изменять значения конфигурации во время процесса загрузки, делая простым изменение поведения приложения без модификации кода.
- После запуска приложения объекты конфигурации замораживают значения, предотвращая неожиданные изменения конфигурации во время выполнения. Это помогает обеспечить ожидаемое поведение приложения.
- Объекты конфигурации централизуют информацию о конфигурации, делая легким её поиск и изменение по необходимости. Это улучшает общую организацию и поддерживаемость кодовой базы.
- Держа информацию о конфигурации отдельно от кода приложения, объекты конфигурации помогают улучшить разделение ответственности и делают код более понятным и поддерживаемым.

## Провайдер конфигурации

Spiral предоставляет `Spiral\Config\ConfiguratorInterface`, который позволяет легко получать доступ к значениям конфигурации, хранящимся в файлах конфигурации. Он предоставляет методы для получения и проверки существования значений конфигурации.

Чтобы продемонстрировать это, мы можем создать файл конфигурации `app/config/github.php`:

```php app/config/github.php
<?php

return [
    'access_token' => 'xxx-xxxx',
    // ...
];
```

Вы можете использовать любое имя, но рекомендуется использовать имя сервиса, который будет использовать эту конфигурацию.

> **Примечание**
> Вы также можете использовать формат `json` или расширить `ConfiguratorInterface` для добавления пользовательских читателей конфигурации.

### Использование

Для доступа к значениям конфигурации в вашем сервисе вы можете использовать `ConfiguratorInterface` следующим образом:

```php
use Spiral\Config\ConfiguratorInterface;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(ConfiguratorInterface $configurator)
    {
        if (!$configurator->exists('github')) {
            throw new \RuntimeException('Github configuration is missing');
        }
        
        $config = $configurator->get('github');
        $this->accessToken = $config['access_token'] ?? throw new \RuntimeException('Missing access token');
    }

    // ...
}
```

## Объект конфигурации

Читать конфигурацию в виде массивов не очень удобно. Фреймворк предоставляет OOP абстракцию для чтения ваших значений. Мы можем создать этот класс вручную или автоматически сгенерировать его через `spiral/scaffolder`:

```terminal
php app.php create:config github -r
``` 

> **Примечание**
> Опция `-r` используется для обратного инжиниринга структуры конфигурации из файла конфигурации `app/config/github.php`.

Полученный класс конфигурации расположен в `app/src/Config/GithubConfig.php`:

```php app/src/Config/GithubConfig.php
namespace App\Config;

use Spiral\Core\InjectableConfig;

class GithubConfig extends InjectableConfig
{
    public const CONFIG = 'github';

    protected array $config = [
        'access_token' => '',
        // ...
    ];

    public function getAccessToken(): string
    {
        return $this->config['access_token'];
    }
}
``` 

Базовый класс `Spiral\Core\InjectableConfig` позволяет вам запрашивать этот объект непосредственно в вашем коде без какой-либо конфигурации IoC контейнера. Константа `CONFIG` содержит имя файла конфигурации.

> **Примечание**
> Также возможно настроить сгенерированный класс и добавить дополнительные методы по необходимости.

```php
use App\Config\GithubConfig;

final class GithubClient
{
    private readonly string $accessToken;
    
    public function __construct(GithubConfig $config)
    {
        $this->accessToken = $config->getAccessToken();
    }

    // ...
}
```

> **Примечание**
> Объект конфигурации предоставляет API только для чтения, изменение значений во время выполнения невозможно для предотвращения нежелательных побочных эффектов в долго работающих приложениях.

Каждый компонент Spiral предоставляет объект конфигурации, который вы можете использовать в вашем приложении.

Используя класс конфигурации, легко получать доступ и управлять значениями конфигурации в вашем приложении Spiral, при этом наслаждаясь преимуществами использования OOP абстракции и улучшенной организации кода.

## Конфигурация по умолчанию в загрузчике

Используйте пользовательский загрузчик для определения значений конфигурации по умолчанию, чтобы избежать необходимости создания ненужных файлов. Переменные окружения могут использоваться как значения по умолчанию.

Это может быть полезно в случаях, когда конфигурации по умолчанию достаточно для большинства приложений, и чтобы избежать необходимости создания ненужных файлов конфигурации.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Boot\EnvironmentInterface;
use Spiral\Config\ConfiguratorInterface;

final class GithubBootloader extends Bootloader
{
    public function init(ConfiguratorInterface $configurator, EnvironmentInterface $env): void
    {
        $configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }
}
```

Этот подход может быть полезен для установки значений конфигурации по умолчанию, которые являются общими для всех окружений, или для предоставления резервного значения, если конкретная переменная окружения не установлена.

Если вы хотите использовать конфигурацию по умолчанию, вы можете просто удалить файл `app/config/github.php`.

> **Примечание**
> Эта функция позволяет устанавливать значения конфигурации по умолчанию, которые могут быть легко переопределены при необходимости, при этом все еще предоставляя резервный вариант, если файл конфигурации отсутствует.

## Автоконфигурация

Некоторые компоненты предоставляют API автоконфигурации для изменения своих настроек во время загрузки приложения. Обычно такой API доступен через загрузчик компонента.

> **Примечание**
> Например `HttpBootloader`->`addMiddleware`.

Мы можем предоставить наш API автоконфигурации в нашем загрузчике. Используйте `ConfiguratorInterface`->`modify` для этой цели. Наш загрузчик будет объявлен как Singleton для ускорения обработки.

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Config\ConfiguratorInterface;
use Spiral\Config\Patch\Set;
use Spiral\Core\Container\SingletonInterface;

class GithubBootloader extends Bootloader implements SingletonInterface
{
    public function __construct(
        private readonly ConfiguratorInterface $configurator
    ) {
    }

    public function init(EnvironmentInterface $env): void
    {
        $this->configurator->setDefaults(GithubConfig::CONFIG, [
            'access_token' => $env->get('GITHUB_ACCESS_TOKEN'),
            'authentication_type' => $env->get('GITHUB_AUTHENTICATION_TYPE', 'token')
        ]);
    }

    public function setAccessToken(string $token): void
    {
        $this->configurator->modify(
          GithubConfig::CONFIG, 
          new Set('access_token', $token)
        );
    }
}
```

Теперь мы можем изменять конфигурацию через строгий API из другого загрузчика:

```php
namespace App\Application\Bootloader;

use Spiral\Boot\Bootloader\Bootloader;

class SomeBootloader extends Bootloader
{
    public function init(GithubBootloader $github): void
    {
        $github->setAccessToken('xxx-xxxx');
    }
}
```

## Жизненный цикл конфигурации

Фреймворк предоставляет механизм безопасности, чтобы убедиться, что вы не изменяете значения конфигурации после того, как объект конфигурации был запрошен любым из компонентов (т.е. конфигурация заморожена).

Чтобы продемонстрировать это, внедрите `GithubConfig` в `SomeBootloader`:

```php
namespace App\Application\Bootloader;

use App\Config\GithubConfig;
use Spiral\Boot\Bootloader\Bootloader;

final class SomeBootloader extends Bootloader
{
    public function boot(GithubBootloader $github, GithubConfig $config): void
    {
        // запрещено
        $github->setAccessToken(800);
    }
}
```

Вы получите это исключение `Spiral\Config\Exception\ConfigDeliveredException`: *Unable to patch config `github`, config object has already been delivered.*

> **Предупреждение**
> Рекомендуется устанавливать значения по умолчанию и изменять конфигурации в методе `init`. И запрашивать файл конфигурации только в методе `boot`. Метод `init` вызывается перед `boot`. Это гарантирует, что когда вызывается метод `boot`, конфигурации будут в правильном состоянии.
