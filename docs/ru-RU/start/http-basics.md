# Начало работы — Первый HTTP-контроллер

Если вы используете Spiral для PHP-проекта и готовы приступить к настройке вашего первого контроллера, вот основные шаги, которые нужно выполнить:

## Создание контроллера

Прежде всего, вам нужно создать контроллер. В Spiral контроллер — это класс, который определяет поведение вашего приложения для определенного набора маршрутов. Он отвечает за обработку входящих запросов, совместимых с [PSR-7](https://www.php-fig.org/psr/psr-7/), обработку данных и возврат ответа клиенту.

Для создания вашего первого контроллера без усилий используйте команду генерации кода:

```terminal
php app.php create:controller CurrentDate
```

> **Примечание**
> Подробнее о генерации кода читайте в разделе [Основы — Генерация кода](../basics/scaffolding.md#http-controller).

После выполнения этой команды следующий вывод подтвердит успешное создание:

```output
Declaration of '[32mCurrentDateController[39m' has been successfully written into '[33mapp/src/Endpoint/Web/CurrentDateController.php[39m'.
```

Теперь давайте добавим логику в наш только что созданный контроллер.

Вот пример контроллера, который возвращает текущую дату и время:

```php app/src/Endpoint/Web/CurrentDateController.php
namespace App\Endpoint\Web;

final class CurrentDateController 
{
    public function show(): string
    {
        return \date('Y-m-d H:i:s');
    }
}
```

Следующий шаг включает связывание маршрута с вашим контроллером.

## Создание маршрута

:::: tabs

::: tab Используя атрибуты

Spiral упрощает определение маршрутов в вашем приложении, используя атрибуты PHP. Вам просто нужно добавить атрибут `#[Route]` к методу контроллера, как показано ниже:

```php app/src/Endpoint/Web/CurrentDateController.php
use Spiral\Router\Annotation\Route;

// ...

#[Route(route: '/date', name: 'current-date', methods: 'GET')]
public function show(): string
{
    return \date('Y-m-d H:i:s');
}
```

:::

::: tab Используя RoutingConfigurator

Для разработчиков, ищущих удобный и организованный подход к определению маршрутов приложения, Spiral предлагает метод `defineRoutes` в классе `App\Application\Bootloader\RoutesBootloader`.

Вот пример того, как определить маршрут, который будет обрабатываться нашим контроллером:

```php app/src/Application/Bootloader/RoutesBootloader.php
final class RoutesBootloader extends BaseRoutesBootloader
{
    // ...

    protected function defineRoutes(RoutingConfigurator $routes): void
    {
        $routes->add(name: 'current-date', pattern: '/date')
            ->action(controller: CurrentDateController::class, action: 'show');
    }
}
```

:::

::::

Для просмотра списка маршрутов используйте следующую команду:

```terminal
php app.php route:list
```

Вы должны увидеть ваш маршрут `current-date` в отображаемом списке:

```output
+--------------+--------+----------+------------------------------------------------+--------+
|[32m Name:        [39m|[32m Verbs: [39m|[32m Pattern: [39m|[32m Target:                                        [39m|[32m Group: [39m|
+--------------+--------+----------+------------------------------------------------+--------+
| current-date | [32mGET[39m    | /date    | App\Endpoint\Web\CurrentDateController->show | web    |
+--------------+--------+----------+------------------------------------------------+--------+
```

## Тестирование вашего контроллера

После того как ваш контроллер настроен, пришло время протестировать его, запустив сервер RoadRunner с помощью команды:

```terminal
./rr serve
```

Теперь вы можете протестировать ваш контроллер, перейдя по маршруту в браузере. Просто откройте следующий URL в вашем браузере: http://127.0.0.1/date

<br><br>

**Вот и всё! Вы успешно настроили ваш первый контроллер в Spiral.**

<hr>

## Что дальше?

Теперь углубитесь в основы, прочитав некоторые статьи:

* [Маршрутизация](../http/routing.md)
* [Аннотированная маршрутизация](../http/annotated-routes.md)
* [Промежуточное ПО (Middleware)](../http/middleware.md)
* [Страницы ошибок](../http/errors.md)
* [Пользовательский HTTP-обработчик](../cookbook/psr-15.md)
* [Генерация кода](../basics/scaffolding.md)
