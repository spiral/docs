# Компонент — Data Grid

Используя мощь Spiral, компонент `spiral/data-grid` предоставляет разработчикам простое решение для автоматического
создания Cycle и DBAL select-запросов на основе пользовательских требований.

> **Внимание:**
> Spiral не предоставляет Cycle ORM из коробки. Чтобы использовать этот компонент, вам необходимо установить компонент
> `spiral/cycle-bridge`. Дополнительную информацию о Cycle ORM Bridge можно найти в
> разделе [Основы — База данных и ORM](../basics/orm.md)

#### Когда стоит рассматривать компонент Data Grid?

1. **Разработка RESTful API:** Представьте себе возможность создания динамических API, которые могут фильтровать,
   сортировать и управлять реляционными данными на основе пользовательских запросов, и все это без написания утомительного
   и повторяющегося кода запросов.

2. **Платформы визуализации данных:** Если вашему приложению нужно отображать огромные наборы данных в табличном
   формате с возможностями сортировки по столбцам, фильтрации записей и разбивки результатов на страницы, компонент
   Data Grid — ваш билет к достижению этого с минимальными усилиями.

3. **Системы управления контентом (CMS):** Для платформ, которые часто получают и отображают данные на основе ролей
   пользователей, предпочтений или критериев поиска, этот компонент может повысить эффективность, делая извлечение
   данных быстрым и простым.

4. **E-commerce платформы:** Подумайте о сценариях, когда пользователи фильтруют товары по категориям, сортируют по
   цене или просматривают страницы с тысячами товаров. Компонент Data Grid может сделать эти операции плавными и
   удобными для пользователя.

**Используя компонент Data Grid, разработчики могут достичь четкого разделения между слоями приложения:**

- **Автоматизированная обработка запросов:** Вместо засорения слоя интерфейса подробными инструкциями по запросам,
  определите пользовательские спецификации (такие как фильтрация или сортировка) и позвольте компоненту автоматически
  создавать необходимые доменные запросы.

- **Унифицированное представление данных:** После того как доменный слой извлекает или обрабатывает данные, компонент
  может стандартизировать способ представления этих данных, независимо от их источника или сложности.

- **Отвязанные источники ввода:** Универсальность компонента в обработке множественных источников ввода гарантирует, что
  доменный слой остается независимым от происхождения запросов, будь то HTTP, консоль или gRPC.

- **Расширяемые схемы grid:** Разрабатывайте схемы grid в соответствии с вашими доменными структурами и позвольте слою
  интерфейса динамически адаптировать их на основе пользовательского ввода, сохраняя четкую границу между структурой
  данных и логикой представления.

## Установка

Для установки компонента:

```terminal
composer require spiral/data-grid-bridge spiral/cycle-bridge
```

Активируйте bootloader `Spiral\DataGrid\Bootloader\GridBootloader` в вашем приложении после bootloaders Cycle:

:::: tabs

::: tab Использование метода

```php app/src/Application/Kernel.php
public function defineBootloaders(): array
{
    return [
        // ...
        \Spiral\DataGrid\Bootloader\GridBootloader::class,
        // ...
    ];
}
```

Подробнее о bootloaders читайте в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::: tab Использование константы

```php app/src/Application/Kernel.php
protected const LOAD = [
    // ...
    \Spiral\DataGrid\Bootloader\GridBootloader::class,
    // ...
];
```

Подробнее о bootloaders читайте в разделе [Фреймворк — Bootloaders](../framework/bootloaders.md).
:::

::::

## Быстрый старт

После установки настройте writers для ваших источников данных в конфигурации `app/config/dataGrid.php`.

**Вот базовая настройка для Cycle ORM Bridge:**

```php app/config/dataGrid.php
return [
    'writers' => [
        \Spiral\Cycle\DataGrid\Writer\QueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\PostgresQueryWriter::class,
        \Spiral\Cycle\DataGrid\Writer\BetweenWriter::class,
    ],
];
```

> **Примечание**
> Как видите, мы можем зарегистрировать несколько writers для разных спецификаций.

## Использование

Компонент предоставляет две базовые абстракции — **Grid Factory** и **Grid Schema**.

### Grid Schema

Представьте себе Schema как набор правил о том, как получать данные на основе того, что просит пользователь.

**Вот простой пример схемы grid:**

```php
use Spiral\DataGrid\GridSchema; 
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Value\StringValue;

$schema = new GridSchema();

// Пользовательская пагинация: ограничить результаты до 10 на страницу
$schema->setPaginator(new PagePaginator(10));

// Опция сортировки: по id
$schema->addSorter('id', new Sorter('id'));

// Опция фильтрации: найти по имени, соответствующему пользовательскому вводу
$schema->addFilter('name', new Like('name', new StringValue()));
```

Короче говоря, с помощью Schema разработчики могут настраивать фильтры и опции сортировки при получении данных.

Для более чистой настройки вы можете расширить `GridSchema` и настроить все внутри его конструктора, как в примере
ниже:

```php
use Spiral\DataGrid\GridSchema; 
use Spiral\DataGrid\Specification\Pagination\PagePaginator;
use Spiral\DataGrid\Specification\Sorter\Sorter;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Value\StringValue;

class UserSchema extends GridSchema
{
    public function __construct()
    {
        // Пользовательская пагинация: ограничить результаты до 10 на страницу
        $this->setPaginator(new PagePaginator(10));
        
        // Опция сортировки: по id
        $this->addSorter('id', new Sorter('id'));
        
        // Опция фильтрации: найти по имени, соответствующему пользовательскому вводу
        $this->addFilter('name', new Like('name', new StringValue()));
    }
}
```

### Grid Factory

Grid Factory — это связь между вашей схемой grid и фактическими данными, которые вы хотите получить. Следующий пример
кода демонстрирует, как соединить схему с данными, используя Cycle ORM Repository:

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\GridFactoryInterface;

$schema = new UserSchema();

$factory = $container->get(GridFactoryInterface::class);
$users = $container->get(\App\Database\UserRepository::class);
  
/** @var Spiral\DataGrid\GridInterface $result */
$result = $factory->create($users->select(), $schema);  

// Получить обработанные данные
print_r(iterator_to_array($result));  
```

Вы также можете установить спецификации по умолчанию:

```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withDefaults([
    GridFactory::KEY_SORT     => ['id' => 'desc'],
    GridFactory::KEY_FILTER   => ['name' => 'Antony'],
    GridFactory::KEY_PAGINATE => ['page' => 3, 'limit' => 100]
]);
```

> **Примечание**
> Поскольку метод `withDefaults` неизменяемый, его вызов не изменяет исходную Grid Factory. Вместо этого он дает
> вам новый экземпляр с указанными значениями по умолчанию.

Как применить спецификации:

- чтобы выбрать пользователей со второй страницы, откройте страницу с `POST` или `QUERY` данными, например: `?paginate[page]=2`
- чтобы активировать фильтр `like`: `?filter[name]=antony`
- чтобы сортировать по id в `ASC` или `DESC`: `?sort[id]=desc`
- чтобы получить общее количество значений: `?fetchCount=1`

Наконец, последний пример кода показывает, как может выглядеть пример контроллера, когда все собрано вместе:

```php app/src/Endpoint/Web/Controller/UserController.php
use Spiral\DataGrid\GridInterface;
use Spiral\DataGrid\GridFactoryInterface;
use App\Database\UserRepository;

class UserController
{
    #[Route('/users')]
    public function index(UserSchema $schema, GridFactoryInterface $factory, UserRepository $users): array
    {
        /** @var GridInterface $result */
        $result = $factory->create($users->select(), $schema);
        
        $values = [];

        foreach ([
            GridInterface::FILTERS, 
            GridInterface::SORTERS, 
            GridInterface::COUNT, 
            GridInterface::PAGINATOR
        ] as $key) {
             $values[$key] = $result->getOption($key);
        }
        
        return [
            'users' => iterator_to_array($result),
            'grid' => [
                'values' => $values,
            ],
        ];
    }
}
```

## Источники ввода

Grid Factory получает свои данные из `Spiral\DataGrid\InputInterface`. По умолчанию он берет эти данные из HTTP
запроса через `Spiral\Http\Request\InputManager`.

Однако великолепной особенностью компонента является его гибкость. Он работает не только с HTTP запросами. Вы можете
заставить его работать с такими вещами, как командные строки, gRPC запросы или другие источники. Просто используйте
`InputInterface` для выбранного источника данных и настройте DataGrids для нужд вашего приложения.

**Есть два основных способа изменить источник данных:**

1. Локальные изменения с `GridFactory::withInput`: Идеально подходит для временных изменений, например, во время
   тестирования. Вот пример использования массива в качестве входных данных:

```php
use Spiral\DataGrid\Input\ArrayInput;

/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withInput(new ArrayInput([
    'name' => 'antony',
    'id' => 'desc'
]));

/** @var Spiral\DataGrid\GridInterface $result */
$result = $factory->create($users->select(), $schema);  
```

> **Примечание**
> Поскольку метод `withInput` неизменяемый, его вызов не изменяет исходную Grid Factory. Вместо этого он дает
> вам новый экземпляр с указанным вводом.

2. **Установка глобального источника ввода через контейнер:** Здесь вы изменяете источник ввода для всего приложения.

Для иллюстрации, вот как вы можете изменить Grid Factory для получения ввода из консоли:

```php app/src/Application/Bootloader/AppBootloader.php
use Spiral\Boot\Bootloader\Bootloader;

class AppBootloader extends Bootloader
{
    protected const SINGLETONS = [
        \Spiral\DataGrid\InputInterface::class => ConsoleInput::class,
    ];
}
```

## Grid writers

В Data Grid у нас есть **Schemas** для описания того, как должны управляться данные, и **Factories** для определения
источника этих данных. Поверх них у нас также есть **Writers**.

Их работа? Изменять данные на основе пользовательских вводов.

**Представьте их вот так:**

Если у вас есть данные в книге и вы используете карандаш (writer) для добавления, изменения или стирания содержимого,
то этот карандаш и есть grid writer. Spiral имеет writers для Cycle ORM. Но крутая часть в том, что вы можете сделать
свой собственный карандаш для других систем, таких как Doctrine Collections.

### Как создать собственный Writer

Хотите сделать свой собственный карандаш (или writer)? Следуйте этим шагам:

1. Используйте `Spiral\DataGrid\WriterInterface` в качестве руководства.

**Вот пример:**

```php app/src/Application/Schema/DoctrineCollectionWriter.php
<?php

declare(strict_types=1);

namespace App\Application\Schema;

use Doctrine\Common\Collections\Collection;
use Doctrine\Common\Collections\Criteria;
use Doctrine\Common\Collections\Expr\Comparison;
use Spiral\DataGrid\Compiler;
use Spiral\DataGrid\Specification\Filter\Equals;
use Spiral\DataGrid\Specification\Filter\Like;
use Spiral\DataGrid\Specification\Sorter\AbstractSorter;
use Spiral\DataGrid\SpecificationInterface;
use Spiral\DataGrid\WriterInterface;

final class DoctrineCollectionWriter implements WriterInterface
{
    public function write(mixed $source, SpecificationInterface $specification, Compiler $compiler): mixed
    {
        // Если данные не в коллекции, просто верните их как есть.
        if (!$source instanceof Collection) {
            return $source;
        }

        // Подготовьте набор правил для изменения данных.
        $criteria = null;

        // Если изменение касается сортировки.
        if ($specification instanceof AbstractSorter) {
            $orders = [];
            foreach ($specification->getExpressions() as $field) {
                $orders[$field] = ($specification->getValue() === AbstractSorter::ASC)
                    ? Criteria::ASC
                    : Criteria::DESC;
            }

            if ($orders !== []) {
                $criteria = (new Criteria())->orderBy($orders);
            }
        // Если изменение касается точного соответствия значений.
        } elseif ($specification instanceof Equals) {
            $expr = new Comparison($specification->getExpression(), Comparison::EQ, $specification->getValue());
            $criteria = (new Criteria())->where($expr);
        // Если изменение касается проверки того, содержат ли данные определенный текст.
        } elseif ($specification instanceof Like) {
            $criteria = new Criteria(
                Criteria::expr()->contains($specification->getExpression(), $specification->getValue())
            );
        } else {
            // ... и так далее.
            return null;
        }

        // Примените изменения, если они есть.
        if ($criteria !== null) {
            $source = $source->matching($criteria)->getValues();
        }

        return $source;
    }
}
```

Если writer возвращает `null`, компилятор будет считать, что writer не знает, как обработать спецификацию, и в случае,
если все writers возвращают `null`, компилятор выбросит исключение `Spiral\DataGrid\Exception\CompilerException`. По
сути, это способ системы сказать: "Эй, что-то здесь не так. Ни один из writers не выполнил свою работу."

**Почему это важно**

Представьте, что вы пытаетесь обновить запись в базе данных. Вы дали системе набор правил о том, как сделать это
обновление. Вы ожидаете одного из двух результатов: либо запись успешно обновлена, либо есть проблема с параметрами
обновления.

`CompilerException` служит механизмом обратной связи. Вместо тихого сбоя и оставления разработчиков в недоумении,
Spiral явно предупреждает их о том, что ни один из writers не внес никаких изменений. Эта обратная связь может быть
бесценной для отладки и обеспечения целостности данных.

2. Зарегистрируйте ваш writer в конфигурации `app/config/dataGrid.php`.

Нам нужно зарегистрировать writer в секции `writers`. Порядок writers важен, поскольку компилятор будет использовать
их в том же порядке, в котором они зарегистрированы.

```php app/config/dataGrid.php
return [
    'writers' => [
        \App\Application\Schema\DoctrineCollectionWriter::class,
    ],
];
```

Вот и все! Теперь вы можете попробовать передать Doctrine Collection в grid factory и посмотреть, как это работает.

## Подсчет элементов

Если вам нужно подсчитать элементы, используя сложную функцию, вы можете передать вызываемую функцию через метод
`withCounter`:

```php
/** @var Spiral\DataGrid\GridFactory $factory */
$factory = $factory->withCounter(static function ($select): int {
    return count($select) * 2;
});
```

> **Примечание**
> Это простой пример, но эта функция может быть очень полезной в случае сложных SQL запросов с соединениями.

## Спецификации пагинации

### Спецификация Page Paginator

Это простая пагинация страница+лимит:

```php
use Spiral\DataGrid\GridSchema;
use Spiral\DataGrid\Specification\Pagination\PagePaginator;

$schema = new GridSchema();
$schema->setPaginator(new PagePaginator(10, [25, 50, 100, 500]));
// ...
```

Из пользовательского ввода такой paginator принимает массив с 2 ключами, `limit` и `page`.
Если лимит установлен, он должен присутствовать в параметре конструктора `allowedLimits`.

```php
use Spiral\DataGrid\Specification\Pagination\PagePagination;

$paginator = new PagePaginator(10, [25, 50, 100, 500]);

$paginator->withValue(['limit' => 123]); // не применится
$paginator->withValue(['limit' => 50]);  // применится
$paginator->withValue(['limit' => 100]); // применится

$paginator->withValue(['limit' => 100, 'page' => 2]);
```

Под капотом этот paginator преобразует `limit` и `page` в спецификации `Limit` и `Offset`. Вы можете написать свой
собственный paginator, например, основанный на курсоре (например: `lastID`+`limit`).

## Спецификации сортировщика

Сортировщики — это спецификации, которые несут направление сортировки. Для сортировщиков, которые могут применять
направление, вы можете передать одно из следующих значений:

- `1`, `'1'`, `'asc'`, `SORT_ASC` для восходящего порядка
- `-1`, `'-1'`, `'desc'`, `SORT_DESC` для нисходящего порядка

Следующие спецификации доступны для grid на данный момент:

* [упорядоченные сортировщики](#спецификации-сортировщика-упорядоченные-сортировщики)
* [направленный сортировщик](#спецификации-сортировщика-направленный-сортировщик)
* [сортировщик](#спецификации-сортировщика-сортировщик)
* [набор сортировщиков](#спецификации-сортировщика-набор-сортировщиков)

### Упорядоченные сортировщики

`AscSorter` и `DescSorter` содержат выражения, которые должны применяться с восходящим (или нисходящим) порядком
сортировки:

```php
use Spiral\DataGrid\Specification\Sorter;

$ascSorter = new Sorter\AscSorter('first_name', 'last_name');
$descSorter = new Sorter\DescSorter('first_name', 'last_name');
```

### Направленный сортировщик

Этот сортировщик содержит 2 независимых сортировщика, каждый для восходящего и нисходящего порядка. Получая порядок
через `withValue`, мы получим один из сортировщиков:

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\DirectionalSorter(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name')
);

// будет сортировать по first_name asc
$ascSorter = $sorter->withDirection('asc');

// будет сортировать по last_name desc
$descSorter = $sorter->withDirection('desc');
```

> **Примечание**
> что вы можете сортировать, используя разный набор полей в обоих сортировщиках.
> Если у вас одинаковый набор полей, используйте [сортировщик](#спецификации-сортировщика-сортировщик) вместо этого.

### Сортировщик

Это обертка сортировщика для направленного сортировщика в случае, если у вас одинаковые поля для сортировки в обоих
направлениях:

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\Sorter('first_name', 'last_name');

// будет сортировать по first_name и last_name asc
$ascSorter = $sorter->withDirection('asc');

// будет сортировать по first_name и last_name desc
$descSorter = $sorter->withDirection('desc');
```

### Набор сортировщиков

Это просто способ объединения сортировщиков в один набор, передача направления применит его ко всему набору:

```php
use Spiral\DataGrid\Specification\Sorter;

$sorter = new Sorter\SorterSet(
    new Sorter\AscSorter('first_name'),
    new Sorter\DescSorter('last_name'),
    new Sorter\Sorter('email', 'username')
    // ...
);

// будет сортировать по first_name, email и username asc, также last_name desc
$ascSorter = $sorter->withDirection('asc');

// будет сортировать по last_name, email и username desc, также first_name asc
$descSorter = $sorter->withDirection('desc');
```

## Спецификации фильтров

Фильтры — это спецификации, которые несут значения. Значения могут передаваться напрямую через конструктор. В этом случае
значение фильтра фиксировано и будет применено как есть.

```php
use Spiral\DataGrid\Specification\Filter;

// name должно быть 'Antony'
$filter = new Filter\Equals('name', 'Antony');

// name по-прежнему 'Antony' 
$filter = $filter->withValue('John');   
```

Если вы передаете `ValueInterface` в конструктор, вы можете использовать метод `withValue()`. Тогда будет проверено,
соответствует ли входящее значение типу `ValueInterface`, и оно будет преобразовано.

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price еще не определен
$filter = new Filter\Equals('price', new Value\NumericValue());

// значение будет преобразовано в int и price должна быть равна 7  
$filter = $filter->withValue('7'); 

// это значение неприменимо, поскольку оно не числовое  
$filter = $filter->withValue([123]);
```

Следующие спецификации доступны для grid в настоящее время:

* [all](#спецификации-фильтров-all)
* [any](#спецификации-фильтров-any)
* [(not) equals](#спецификации-фильтров-not-equals)
* [compare gt/gte lt/lte](#спецификации-фильтров-compare)
* [(not) in array](#спецификации-фильтров-not-in-array)
* [like](#спецификации-фильтров-like)
* [map](#спецификации-фильтров-map)
* [select](#спецификации-фильтров-select)
* [between](#спецификации-фильтров-between)

> **Примечание**
> Есть более интересные вещи в разделах [значения фильтров](#значения-фильтров) и [акцессоры значений](#акцессоры-значений)
> ниже.

### All

Это объединяющий фильтр для логической операции `and`.

Несколько примеров с фиксированными значениями:

```php
use Spiral\DataGrid\Specification\Filter;

// price должна быть равна 2 и quantity должно быть больше 5
$all = new Filter\All(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

Переданное значение будет применено ко всем под-фильтрам:

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$all = new Filter\All(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// price должна быть равна 5, quantity должно быть больше 5 и option_id меньше 4
$all = $all->withValue(5);
```

### Any

Это объединяющий фильтр для логической операции `or`.

Примеры с фиксированными значениями:

```php
use Spiral\DataGrid\Specification\Filter;

// price должна быть равна 2 или quantity должно быть больше 5
$any = new Filter\Any(
    new Filter\Equals('price', 2),
    new Filter\Gt('quantity', 5)
);
```

Переданное значение будет применено ко всем под-фильтрам.

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$any = new Filter\Any(
    new Filter\Equals('price', new Value\NumericValue()),
    new Filter\Gt('quantity', new Value\IntValue()),
    new Filter\Lt('option_id', 4)
);

// price должна быть равна 5 или quantity должно быть больше 5 или option_id меньше 4
$any = $any->withValue(5);
```

### (Not) equals

Это простые фильтры выражений для логических операций `=`, `!=`.

Примеры с фиксированным значением:

```php
use Spiral\DataGrid\Specification\Filter;

$equals = new Filter\Equals('price', 2);       // price должна быть равна 2
$notEquals = new Filter\NotEquals('price', 2); // price не должна быть равна 2
```

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price должна быть равна 2
$equals = new Filter\Equals('price', new Value\NumericValue());
$equals = $equals->withValue('2');

// price не должна быть равна 2
$notEquals = new Filter\NotEquals('price', new Value\NumericValue());
$notEquals = $notEquals->withValue('2');
```

### Compare

Это простые фильтры выражений для логических операций `>`, `>=`, `<`, `<=`.

Примеры с фиксированным значением:

```php
use Spiral\DataGrid\Specification\Filter;

$gt = new Filter\Gt('price', 2);   // price должна быть больше 2
$gte = new Filter\Gte('price', 2); // price должна быть больше 2 или равна
$lt = new Filter\Lt('price', 2);   // price должна быть меньше 2
$lte = new Filter\Lte('price', 2); // price должна быть меньше 2 или равна
```

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price должна быть больше 2
$gt = new Filter\Gt('price', new Value\NumericValue());
$gt = $gt->withValue('2');

// price должна быть больше 2 или равна
$gte = new Filter\Gte('price', new Value\NumericValue());
$gte = $gte->withValue('2');

// price должна быть меньше 2
$lt = new Filter\Lt('price', new Value\NumericValue());
$lt = $lt->withValue('2');

// price должна быть меньше 2 или равна
$lte = new Filter\Lte('price', new Value\NumericValue());
$lte = $lte->withValue('2');
```

### (Not) in array

Это простые фильтры выражений для логических операций `in`, `not in`.

Примеры с фиксированным значением:

```php
use Spiral\DataGrid\Specification\Filter;

$inArray = new Filter\InArray('price', [2, 5]);       // price должна быть в массиве из 2 и 5
$notInArray = new Filter\NotInArray('price', [2, 5]); // price не должна быть в массиве из 2 и 5
```

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price должна быть в массиве из 2 и 5
$inArray = new Filter\InArray('price', new Value\NumericValue());
$inArray = $inArray->withValue(['2', '5']);

// price не должна быть в массиве из 2 и 5
$notInArray = new Filter\NotInArray('price', new Value\NumericValue());
$notInArray = $notInArray->withValue(['2', '5']);
```

Третий параметр позволяет автоматически обертывать `ValueInterface` в `ArrayValue` (включено по умолчанию). В случае
если у вас нетривиальное значение (или обернутое в акцессор значение), передайте `false` как 3-й аргумент для
контроля обертывания фильтра:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor\Split;
use Spiral\DataGrid\SpecificationInterface;

$inArray = new Filter\InArray('field', new Split(new Value\ArrayValue(new Value\IntValue()), '|'), false);
$inArray->withValue('1|2|3')->getValue(); // [1, 2, 3]
```

### Like

Это простой фильтр выражения для операции `like`.

Примеры с фиксированным значением:

```php
use Spiral\DataGrid\Specification\Filter;

$likeFull = new Filter\Like('name', 'Tony', '%%%s%%'); // name должно быть как '%Tony%'
$likeEnding = new Filter\Like('name', 'Tony', '%s%%'); // name должно быть как 'Tony%'
```

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// name должно быть как '%Tony%'
$like = new Filter\Like('name', new Value\StringValue());
$like = $like->withValue('Tony');
```

### Map

Map — это сложный фильтр, представляющий карту фильтров с их собственными значениями.

```php
use Spiral\DataGrid\Specification\Filter;

// price должна быть больше 2 и quantity должно быть меньше 5
$map = new Filter\Map([
    'from' => new Filter\Gt('price', 2),
    'to'   => new Filter\Lt('quantity', 5)
]);
```

Переданные значения будут применены ко всем под-фильтрам, все значения обязательны:

Примеры с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

$map = new Filter\Map([
    'from' => new Filter\Gt('price', new Value\NumericValue()),
    'to'   => new Filter\Lt('quantity', new Value\NumericValue())
]);

// price должна быть больше 2 и quantity должно быть меньше 5
$map = $map->withValue(['from' => 2, 'to' => 5]);

// неверный ввод, map будет установлен в null
$map = $map->withValue(['to' => 5]);
```

### Select

Эта спецификация представляет набор доступных выражений.
Передача значения из ввода выберет одну или несколько спецификаций из этого набора.

> **Примечание**
> Вам нужно только передать ключ или массив ключей. Обратите внимание, что никакой `ValueInterface` не должен быть объявлен.

Пример с одним значением:

```php
use Spiral\DataGrid\Specification\Filter;

// обратите внимание, что у нас здесь целочисленные ключи
$select = new Filter\Select([
    new Filter\Equals('name', 'value'),
    new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    new Filter\Equals('email', 'email@example.com'),
]);

// выбрать второй фильтр, будет равен спецификации 'any'.
$filter = $select->withValue(1);
```

Пример с несколькими значениями:

```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// фильтр будет содержать оба под-фильтра, обернутые в спецификацию 'all'
$filter = $select->withValue(['one', 'two']);
```

Пример с неизвестным значением:

```php
use Spiral\DataGrid\Specification\Filter;

$select = new Filter\Select([
    'one'  => new Filter\Equals('name', 'value'),
    'two'  => new Filter\Any(
        new Filter\Equals('price', 2),
        new Filter\Gt('quantity', 5)
    ),
    'three' => new Filter\Equals('email', 'email@example.com'),
]);

// фильтр будет равен null
$filter = $select->withValue('four');
```

### Between

Этот фильтр представляет SQL-операцию `between`, но может быть представлен как два фильтра `gt/gte` и `lt/lte`.

У вас есть возможность определить, должны ли граничные значения включаться или нет. Если граничные значения не
включены, этот фильтр будет преобразован в фильтры `gt`+`lt`, иначе при получении фильтров через метод `getFilters()`
вы можете указать либо использование оригинального оператора `between`, либо фильтров `gte`+`lte`.

> **Примечание**
> Не все базы данных поддерживают операцию `between`, поэтому преобразование в `gt/gte`+`lt/lte` используется по умолчанию.

Фильтр Between имеет две модификации: основанную на поле и основанную на значении.

```php
use Spiral\DataGrid\Specification\Filter;

$fieldBetween  = new Filter\Between('field', [10, 20]);
$valueBetween  = new Filter\ValueBetween('2020 Apr, 10th', ['start_date', 'end_date']);
```

Примеры выше аналогичны следующим SQL-запросам:

```sql
#
основанный на поле
select *
from table_name
where field between 10 and 20;
#
или используя преобразование gte/lte
select *
from table_name
where field >= 10
  and field <= 20;

#
основанный на значении
select *
from table_name
where '2020 Apr, 10th' between start_date and end_date;
#
или используя преобразование gte/lte
select *
from table_name
where start_date <= '2020 Apr, 10th'
  and end_date >= '2020 Apr, 10th';
```

Пример с использованием `ValueInterface`:

```php
use Spiral\DataGrid\Specification\Filter;
use Spiral\DataGrid\Specification\Value;

// price должна быть между 10 и 20
$fieldBetween  = new Filter\Between('price', new Value\NumericValue());
$fieldBetween = $fieldBetween->withValue([10, '20']);

// '2020 Apr, 10th' должно быть между start_date и end_date
$valueBetween  = new Filter\ValueBetween(new Value\DatetimeValue(), ['start_date', 'end_date']);
$valueBetween = $valueBetween->withValue('2020 Apr, 10th');
```

Выберите тип рендеринга:

```php
use Spiral\DataGrid\Specification\Filter;

$between  = new Filter\Between('price', [10, 20]);

$between->getFilters();     // будет преобразован в gte+lte
$between->getFilters(true); // будет представлен как есть

$notIncludingBetween  = new Filter\Between('price', [10, 20], false, false);

// будет преобразован в gte+lte в любом случае
$notIncludingBetween->getFilters();
$notIncludingBetween->getFilters(true);
```

> **Примечание**
> То же самое касается фильтра `ValueBetween`.

## Смешанные спецификации

`Spiral\DataGrid\Specification\Filter\SortedFilter` и `Spiral\DataGrid\Specification\Sorter\FilteredSorter` —
это специальные последовательные спецификации, которые позволяют использовать как фильтры, так и сортировщики под
одним именем.

Использование:

```php

$schema->addFilter(
    'filter',
    new Filter\Select(
        [
            'upcoming'      => new Sorter\SortedFilter(
                'upcoming',
                new Filter\Gt('date', new DateTimeImmutable('now')),
                new Sorter\AscSorter('date')
            ),
            'mostReviewed'  => new Sorter\SortedFilter(
                'mostReviewed',
                new Filter\Lte('date', new DateTimeImmutable('now')),
                new Sorter\DescSorter('count_reviews')
            )
        ]
    )
);
```

> **Примечание**
> Мы применяем и сортировку, и фильтрацию, используя фильтр `upcoming`.

## Значения фильтров

Мы используем значения фильтров для преобразования типа ввода и его валидации. Пожалуйста, не используйте метод
`convert()` без валидации ввода с помощью метода `accepts()`. Они могут сказать вам, является ли ввод приемлемым, и
преобразовать его в желаемый тип. Следующие значения доступны для grid в настоящее время:

* [any](#значения-фильтров-any)
* [array](#значения-фильтров-array)
* [bool](#значения-фильтров-bool)
* [zero compare](#значения-фильтров-zero-compare)
* [numbers](#значения-фильтров-numbers)
* [datetime](#значения-фильтров-datetime)
* [datetime format](#значения-фильтров-datetime-format)
* [enum](#значения-фильтров-enum)
* [intersect](#значения-фильтров-intersect)
* [subset](#значения-фильтров-subset)
* [string](#значения-фильтров-string)
* [scalar](#значения-фильтров-scalar)
* [regex](#значения-фильтров-regex)
* [uuid](#значения-фильтров-uuid)
* [range](#значения-фильтров-range)
* [not empty](#значения-фильтров-not-empty)

### Any

Это значение принимает любой ввод и не преобразует его:

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\AnyValue();
 
$value->accepts('123'); // всегда true
$value->convert('123'); // всегда равно вводу
```

### Array

Это значение ожидает массив и преобразует все его элементы согласно базовому типу значения. Ввод не должен быть пустым:

```php
use Spiral\DataGrid\Specification\Value;

// ожидает массив int значений
$value = new Value\ArrayValue(new Value\IntValue());
 
$value->accepts('123');   // false
$value->accepts([]);      // false
$value->accepts(['123']); // true
$value->convert(['123']); // [123]
```

### Bool

Это значение ожидает bool ввод, 1/0 (как int или строки), и строки `true`/`false`:

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\BoolValue();
 
$value->accepts('123');   // false
$value->accepts('0');     // true
$value->accepts(['123']); // false
$value->convert('1');     // true
$value->convert('false'); // false
```

### Zero-compare

Эти значения предназначены для проверки вашего ввода на положительность/отрицательность/неположительность/неотрицательность
согласно базовому типу значения:

```php
use Spiral\DataGrid\Specification\Value;

$positive = new Value\PositiveValue(new Value\IntValue());       // как int должно быть > 0
$negative = new Value\NegativeValue(new Value\IntValue());       // как int должно быть < 0
$nonPositive = new Value\NonPositiveValue(new Value\IntValue()); // как int должно быть >= 0
$nonNegative = new Value\NonNegativeValue(new Value\IntValue()); // как int должно быть <= 0
```

### Numbers

Применяет числовые значения, также пустые строки (ноль тоже является значением):

```php
use Spiral\DataGrid\Specification\Value;

$int = new Value\IntValue();         // преобразует в int
$float = new Value\FloatValue();     // преобразует в float
$numeric = new Value\NumericValue(); // преобразует в int/float
```

### Datetime

Это значение ожидает строку, представляющую timestamp или datetime, и преобразует её в `\DateTimeImmutable`:

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\DatetimeValue();
 
$value->accepts('abc');     // false
$value->accepts('123');     // true
$value->accepts('-1 year'); // true
$value->convert('-1 year'); // объект DateTimeImmutable
```

### Datetime Format

Это значение ожидает строку, представляющую datetime, отформатированную согласно заданному формату. Datetime
преобразуется в `\DateTimeImmutable`, datetime будет дополнительно отформатирована, используя выходной формат, если
передан 2-й аргумент:

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\DatetimeFormatValue('Y-m-d');
 
$value->accepts('2020 Jan 21st');  // false
$value->accepts('2020-01-21');     // true

$value->convert('2020-01-21'); // объект DateTimeImmutable

$value = new Value\DatetimeFormatValue('Y-m-d', 'F dS, y');
$value->convert('2020-01-21'); // January 21st, 20
```

### Enum

Это значение ожидает, что ввод будет частью заданного массива enum, и преобразует его согласно базовому типу значения.
Все значения enum также преобразуются:

```php
use Spiral\DataGrid\Specification\Value;

// ожидает массив int значений
$value = new Value\EnumValue(new Value\IntValue(), 1, '2', 3);
 
$value->accepts('3'); // true
$value->accepts(4);   // false
$value->convert('3'); // 3
```

### Intersect

Это значение основано на enum значении, разница в том, что хотя бы один из элементов массива ввода должен
соответствовать заданному массиву enum:

```php
use Spiral\DataGrid\Specification\Value;

// ожидает массив int значений
$value = new Value\IntersectValue(new Value\IntValue(), 1, '2', 3);
 
$value->accepts('3');    // true
$value->accepts(4);      // false
$value->accepts([3, 4]); // true
$value->convert('3');    // [3]
```

### Subset

Это значение основано на enum значении, разница в том, что все элементы массива ввода должны соответствовать заданному
массиву enum:

```php
use Spiral\DataGrid\Specification\Value;

// ожидает массив int значений
$value = new Value\SubsetValue(new Value\IntValue(), 1, '2', 3);
 
$value->accepts('3');    // true
$value->accepts(4);      // false
$value->accepts([3, 4]); // false
$value->accepts([2, 3]); // true
$value->convert('3');    // [3]
```

### String

Применяет строковый ввод, также пустые строки (если передан соответствующий параметр конструктора):

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\StringValue();
$allowEmpty = new Value\StringValue(true);

$value->accepts('');      // false
$value->accepts(false);   // false
$value->accepts('3');     // true
$value->accepts(4);       // true
$value->convert(3);       // '3'
$allowEmpty->accepts(''); // true
```

### Scalar

Применяет скалярные значения, также пустые строки (если передан соответствующий параметр конструктора):

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\ScalarValue();
$allowEmpty = new Value\ScalarValue(true);

$value->accepts('');       // false
$value->accepts(false);    // true
$value->accepts('3');      // true
$value->accepts(4);        // true
$value->convert(3);        // '3'
$allowEmpty->accepts('');  // true
```

### Regex

Применяет строковый ввод и проверяет, соответствует ли он заданному паттерну regex, преобразует в строку:

```php
use Spiral\DataGrid\Specification\Value;

$value = new Value\RegexValue('/\d+/');

$value->accepts('');  // false
$value->accepts(3);   // true
$value->accepts('4'); // true
$value->convert(3);   // '3'
```

### Uuid

Применяет строки в формате UUID, пользователь может выбрать, какой паттерн валидации использовать:

- any (просто проверить формат строки)
- nil (специальное null значение uuid)
- одна из версий [1-5]

Вывод преобразуется в строку.

```php
use Spiral\DataGrid\Specification\Value;

$v4 = new Value\UuidValue('v4');
$valid = new Value\UuidValue();

$v4->accepts('');                                        // false
$v4->accepts('00000000-0000-0000-0000-000000000000');    // false
$valid->accepts('');                                     // false
$valid->accepts('00000000-0000-0000-0000-000000000000'); // true
```

### Range

Это значение ожидает, что ввод будет внутри заданного диапазона, и преобразует его согласно базовому типу значения.
Граничные значения диапазона также преобразуются. Вы также можете указать, может ли ввод быть равен граничным значениям
или нет:

```php
use Spiral\DataGrid\Specification\Value;

// как есть, ожидает, что значение будет >=1 и <3
$value = new Value\RangeValue(
    new Value\IntValue(),
    Value\RangeValue\Boundary::including(1),
    Value\RangeValue\Boundary::excluding(3)
);
 
$value->accepts('3'); // false
$value->accepts(1);   // false
```

### Not-Empty

Если вам нужно сначала проверить, что значение не пустое, оберните значение в `NotEmpty()`:

```php
use Spiral\DataGrid\Specification\Value;

$int = new Value\IntValue();
$notEmpty = new Value\NotEmpty($int);

$int->accepts(0);      // true
$notEmpty->accepts(0); // false
```

## Акцессоры значений

Акцессоры действуют как значения из раздела выше, но имеют другую цель — вы можете использовать их для выполнения
нетиповых преобразований, например, используя строки, вы можете захотеть обрезать значение или преобразовать его в
верхний регистр. Они могут применяться только если значение применимо заданным `ValueInterface`. Примеры ниже:

```php
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor;

(new Accessor\ToUpper(new Value\StringValue()))->convert('abc'); // 'ABC'
(new Accessor\ToUpper(new Value\StringValue()))->convert('ABC'); // 'ABC'
(new Accessor\ToUpper(new Value\StringValue()))->convert(123);   // '123'
(new Accessor\ToUpper(new Value\ScalarValue()))->convert(123);   // 123
```

Все поддерживаемые акцессоры имеют следующий порядок обработки: сначала выполняют свои собственные операции, затем
передают их на более низкий уровень. Например, у нас есть акцессоры `add` и `multiply`:

```php
use Spiral\DataGrid\Specification\Value;
use Spiral\DataGrid\Specification\Value\Accessor;

$multiply = new Accessor\Multiply(new Accessor\Add(new Value\IntValue(), 2), 2);
$add = new Accessor\Add(new Accessor\Multiply(new Value\IntValue(), 2), 2);

$multiply->convert(2); // 2*2+2=6
$add->convert(2);      // (2+2)*2=8
```

Следующие акцессоры доступны для grid в настоящее время:

- `trim` обрезает строку
- `toUpper` преобразует строку в верхний регистр
- `toLower` преобразует строку в нижний регистр
- `split` разделяет строку, используя заданный символ, в массив
