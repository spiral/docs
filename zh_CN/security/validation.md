# 验证接口

开发者可以使用 `spiral/validation` 组件来进行数据验证。该组件提供了基于数组的 DSL 来构建复杂验证链的功能。

该组件包含检查器、条件判断和验证对象。Web 和 GRPC 应用程序模板已经模板包含了这个组件。

> 对于深层数据验证，请参考 [请求验证](/zh_CN/filters/configuration.md) 文档。

## 安装和配置

要自行安装该组件，可以执行：

```bash
$ composer install spiral/validation
```

要在 Spiral 框架的应用程序中使用，需注册 `Spiral\Bootloader\Security\ValidationBootloader` 引导程序。

需要对组件的配置选项进行修改，可以创建和编辑 `app/config/validation.php` 文件。默认配置如下：

```php
<?php

declare(strict_types=1);

use Spiral\Validation;

return [
    // 检查器（checkers）会由 container 进行解析并提供在通用名称和类之下的
    // 彼此独立的验证规则。你可以根据需要注册新的检查器，注册检查器的数量不会带来性能问题。
    'checkers'   => [
        'type'    => Validation\Checker\TypeChecker::class,
        'number'  => Validation\Checker\NumberChecker::class,
        'mixed'   => Validation\Checker\MixedChecker::class,
        'address' => Validation\Checker\AddressChecker::class,
        'string'  => Validation\Checker\StringChecker::class,
        'file'    => Validation\Checker\FileChecker::class,
        'image'   => Validation\Checker\ImageChecker::class,
    ],

    // 启用/禁用 验证判断条件
    'conditions' => [
        'withAny'    => Validation\Condition\WithAnyCondition::class,
        'withoutAny' => Validation\Condition\WithoutAnyCondition::class,
        'withAll'    => Validation\Condition\WithAllCondition::class,
        'withoutAll' => Validation\Condition\WithoutAllCondition::class,
    ],

    // 别名只是为了让开发更简单
    'aliases'    => [
        'notEmpty'   => 'type::notEmpty',
        'required'   => 'type::notEmpty',
        'datetime'   => 'datetime::valid',
        'timezone'   => 'datetime::timezone',
        'bool'       => 'type::boolean',
        'boolean'    => 'type::boolean',
        'cardNumber' => 'mixed::cardNumber',
        'regexp'     => 'string::regexp',
        'email'      => 'address::email',
        'url'        => 'address::url',
        'file'       => 'file::exists',
        'uploaded'   => 'file::uploaded',
        'filesize'   => 'file::size',
        'image'      => 'image::valid',
        'array'      => 'is_array',
        'callable'   => 'is_callable',
        'double'     => 'is_double',
        'float'      => 'is_float',
        'int'        => 'is_int',
        'integer'    => 'is_integer',
        'numeric'    => 'is_numeric',
        'long'       => 'is_long',
        'null'       => 'is_null',
        'object'     => 'is_object',
        'real'       => 'is_real',
        'resource'   => 'is_resource',
        'scalar'     => 'is_scalar',
        'string'     => 'is_string',
        'match'      => 'mixed::match',
    ]
];
```

在开发中（通常在控制器或服务）可以通过提供者工厂使用此组件：

```php
namespace App\Controller;

use Spiral\Validation;

class HomeController
{
    public function index(Validation\ValidationInterface $validation)
    {
        $validator = $validation->validate(
            // 要验证的数据
            [
                'key' => null
            ],
            // 验证规则
            [
                'key' => [
                    'notEmpty'
                ]
            ]
        );

        dump($validator instanceof Validation\Validator);

        dump($validator->isValid());
        dump($validator->withData(['key' => 'value'])->isValid());
    }
}
```

> 也可以直接使用开发辅助原型的 `validator` 属性来使用验证组件。

## 验证接口

`ValidationInterface`->`validate` 方法返回的结果是 `ValidatorInterface` 接口的实现。该接口提供了基本的 API 来获取验证结果和错误，并允许将其附加的新的数据或者不可变上下文：

```php
interface ValidatorInterface
{
    public function withData($data): ValidatorInterface;
    public function getValue(string $field, $default = null);
    public function withContext($context): ValidatorInterface;
    public function getContext();
    public function isValid(): bool;
    public function getErrors(): array;
}
```

正常的流程是使用 `isValid`：

```php
public function index(Validation\ValidationInterface $validation)
{
    $validator = $validation->validate(
        ['key' => null],
        ['key' => ['notEmpty']]
    );

    if (!$validator->isValid()) {
        dump($validator->getErrors());
    }
}
```

### 有效数据

验证器接受任何数组数据源，但在内部它会把数据源转换为数组形式（除了 ArrayAccess）。

### 错误格式

验证组件总会返回一个 error 对象，包含每个键触发的第一个验证错误。

```php
[
    'name' => 'This field is required.',
    'key'  => 'Another error'
]
```

> 错误信息可以使用 `spiral/translator` 组件进行本地化。

## 验证 DSL

默认的 Spiral 验证器接受以嵌套数组形式提供的验证规则。数组的键是要验证的属性名称，值是要依次应用于该属性值的验证规则数组：

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty', // 'key' 属性的值不能为空
            'string'    // 'key' 属性的值必须为字符串
        ]
    ]
);

if (!$validator->isValid()) {
    dump($validator->getErrors());
}
```

在这种情况下，规则是检查器（checker）方法或者任何可用的 PHP 函数，该方法或函数接受 `value` 作为第一个参数。

例如我们可以在验证规则中直接使用 `is_numeric`

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty',  // 'key' 的值不能为空
            'is_numeric' // 'key' 的值必须是数字
        ]
    ]
);
```

### 扩展声明

在很多情况下，需要声明额外的规则参数、条件或者自定义错误信息。要实现这个目标，需要把规则声明封装为一个数组：

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty'],  // 'key' 的值不能为空
            ['is_numeric'] // 'key' 的值必须是数组
        ]
    ]
);
```

> 如果不需要额外的参数，可以省略用于包裹规则的 `[]`。

### 检查器规则

规则名称可以用 `::` 前缀分隔，这种情况下第一部分是检查器的名称，第二部分是检查器的方法名称。例如：

```php
$validator = $validation->validate(
    ['file' => null],
    [
        'file' => [
            'file::uploaded'
        ]
    ]
);
```

### 参数

在规则数组中的所有值都会作为额外参数传递给检查方法，例如使用 `in_array` 方法做为规则的定义：

```php
$validator = $validation->validate(
    ['name' => 'f'],
    [
        'name' => [
            'notEmpty',
            ['in_array', ['a', 'b', 'c'], true] // in_array($value, ['a', 'b', 'c'], true)
        ]
    ]
);
```

如果要指定正则表达式作为规则：

```php
$validator = $validation->validate(
    ['name' => 'b'],
    [
        'name' => [
            'notEmpty',
            ['regexp', '/^a+$/'] // 'name' 的值必须是 a 开头的字符串
        ]
    ]
);
```

### 错误信息

Validator will render default error message for any custom rule, to set custom error message set the rule
attribute:
验证器会为任何自定义规则提供默认的错误信息，但也可以自行设定错误信息，方法如下：

```php
$validator = $validation->validate(
    ['file' => 'b'],
    [
        'file' => [
            'notEmpty',
            ['regexp', '/^a+$/', 'error' => '格式错误，应该为 "a+" 的格式。'] // aaa...
        ]
    ]
);
```

> 你可以给任何规则制定自定义错误信息，不局限于自定义规则。

### 判断条件

有时，某条规则只有在另外的条件符合时才使用，这种情况下，可以为规则指定 `if` 属性：

```php
$validator = $validation->validate(
    [
        'password'        => '',
        'confirmPassword' => ''
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]]
        ]
    ]
);
```

> 在上面的示例中，`confirmPassword` 属性的 `notEmpty` 规则只在 `password` 不为空时才触发。

当然也可以使用多个条件，或者把它们组合成一个复杂的规则：

```php
 $validator = $validation->validate(
    [
        'password'        => 'abc',
        'confirmPassword' => 'cde'
    ],
    [
        'password'        => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]],
            ['match', 'password', 'error' => '两次输入的密码不一致']
        ]
    ]
);
```

### 可用的条件

以下是默认可用的判断条件：

| 名称       | 选项    | 描述                           |
| ---------- | ------- | ------------------------------ |
| withAny    | _array_ | 当指定的任意一个属性不为空时。 |
| withoutAny | _array_ | 当指定的任意一个属性为空时。   |
| withAll    | _array_ | 当指定的所有属性不为空时。     |
| withoutAll | _array_ | 当指定的所有属性都为空时。     |

> 你可以用 `Spiral\Validation\ConditionInterface` 来创建自己的条件。

## 可用的规则

以下是默认可用的验证规则。

> 你可以用 `Spiral\Validation\AbstractChecker` 或者 `Spiral\Validation\CheckerInterface` 来创建自己的验证规则。

### 规则别名

最常用的规则集及其别名如下：

| 别名       | 规则               |
| ---------- | ------------------ |
| notEmpty   | type::notEmpty     |
| required   | type::notEmpty     |
| datetime   | datetime::valid    |
| timezone   | datetime::timezone |
| bool       | type::boolean      |
| boolean    | type::boolean      |
| cardNumber | mixed::cardNumber  |
| regexp     | string::regexp     |
| email      | address::email     |
| url        | address::url       |
| file       | file::exists       |
| uploaded   | file::uploaded     |
| filesize   | file::size         |
| image      | image::valid       |
| array      | is_array           |
| callable   | is_callable        |
| double     | is_double          |
| float      | is_float           |
| int        | is_int             |
| integer    | is_integer         |
| numeric    | is_numeric         |
| long       | is_long            |
| null       | is_null            |
| object     | is_object          |
| real       | is_real            |
| resource   | is_resource        |
| scalar     | is_scalar          |
| string     | is_string          |
| match      | mixed::match       |

### 类型

> 前缀 `type::`

| 规则     | 参数                   | 描述                      |
| -------- | ---------------------- | ------------------------- |
| notEmpty | asString:_bool_ - true | 值不能为空（同 `!empty`） |
| notNull  | ---                    | 值不能为 null             |
| boolean  | ---                    | 值必须为布尔值或数字[0,1] |

> 以上的所有规则都可以省略前缀

### 必填项

| 规则     | 参数                   | 描述       |
| -------- | ---------------------- | ---------- |
| notEmpty | asString:_bool_ - true | 值不能为空 |

例如：

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'name'  => [
            ['notEmpty'],
            ['my::abc']
        ]
    ];
}
```

### 混合

> 前缀 `mixed::`

| 规则       | 参数                                  | 描述                           |
| ---------- | ------------------------------------- | ------------------------------ |
| cardNumber | ---                                   | 通过 Luhn 算法检查信用卡卡号   |
| match      | field:_string_, strict:_bool_ - false | 检查值是否与另一个属性的值一致 |

> 上述的规则都可以用省略前缀。

### 地址

> 前缀 `address::`

| 规则  | 参数                        | 描述               |
| ----- | --------------------------- | ------------------ |
| email | ---                         | 检查是否合法 Email |
| url   | requireScheme:_bool_ - true | 检查是否合法 URL   |

> 上述的规则都可以用省略前缀。

## 数字

> 前缀 `number::`

| 规则   | 参数                       | 描述                         |
| ------ | -------------------------- | ---------------------------- |
| range  | begin:_float_, end:_float_ | 检查数字是否在指定范围       |
| higher | limit:_float_              | 检查数字是否大于或等于指定值 |
| lower  | limit:_float_              | 检查数字是否小于或等于指定值 |

### 字符串

> 前缀 `string::`

| 规则    | 参数                    | 描述                               |
| ------- | ----------------------- | ---------------------------------- |
| regexp  | expression:_string_     | 检查字符串是否匹配指定正则表达式   |
| shorter | length:_int_            | 检查字符串长度是否小于或等于指定值 |
| longer  | length:_int_            | 检查字符串长度是否大于或等于指定值 |
| length  | length:_int_            | 检查字符串长度是否等于指定值       |
| range   | left:_int_, right:_int_ | 检查字符串长度是否在指定范围内     |

示例：

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'name' => [
            ['notEmpty'],
            ['string::length', 5]
        ]
    ];
}
```

### 文件

> 前缀 `file::`

文件检查器完全支持以字符串形式提供的文件名或者 PSR-7 的 `UploadedFileInterface` 接口：

| 规则      | 参数               | 描述                                                   |
| --------- | ------------------ | ------------------------------------------------------ |
| exists    | ---                | 检查文件是否存在                                       |
| uploaded  | ---                | 检查文件是否已上传                                     |
| size      | size:_int_         | 检查文件尺寸是否小于指定值（单位为 KB）                |
| extension | extensions:_array_ | 检查文件扩展名是否在白名单内（使用客户端提供的文件名） |

### 图片

> 前缀 `image::`

图片检查器扩展了文件检查器，因此完全支持文件检查器的功能。

| 规则    | 参数                      | 描述                                               |
| ------- | ------------------------- | -------------------------------------------------- |
| type    | types:_array_             | 检查图片类型是否在白名单内                         |
| valid   | ---                       | 图片类型规则的别名（允许 JPEG, PNG, and GIF 格式） |
| smaller | width:_int_, height:_int_ | 检查图片的像素尺寸是否小于指定值（高度可选）       |
| bigger  | width:_int_, height:_int_ | 检查图片的像素尺寸是否大于指定值（高度可选）       |

### 日期时间

> 前缀 `datetime::`

| 规则     | 参数                                                                            | 描述                                       |
| -------- | ------------------------------------------------------------------------------- | ------------------------------------------ |
| future   | orNow:_bool_ - false,<br/>useMicroSeconds:_bool_ - false                        | 必须为未来的时间                           |
| past     | orNow:_bool_ - false,<br/>useMicroSeconds:_bool_ - false                        | 必须为过去的时间                           |
| format   | format:_string_                                                                 | 值必须符合指定的时间格式                   |
| before   | field:_string_,<br/>orEquals:_bool_ - false,<br/>useMicroSeconds:_bool_ - false | 值必须为指定值之前的时间                   |
| after    | field:_string_,<br/>orEquals:_bool_ - false,<br/>useMicroSeconds:_bool_ - false | 值必须为指定值之后的时间                   |
| valid    | ---                                                                             | 值必须为有效的时间日期定义，包括数字时间戳 |
| timezone | ---                                                                             | 值必须为有效的时区定义                     |

> 设置 `useMicroSeconds` 为 true 允许检查含有微妙的时间。  
> 要注意的是，两个 `new \DateTime('now')` 创建的对象 99% 会包含不同的微秒值，因此它们总是不相等。

### 实体

> 前缀 `entity::`

Cycle ORM 专属的检查器。

| 规则   | 参数                                                  | 描述                                                                                                                   |
| ------ | ----------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| exists | class:_string_, field:_string_ - null                 | 实体在数据库中是否存在。`class` 是一个实体的类名。`field` 是属性对应的列。                                             |
| unique | class:_string_, field:_string_, withFields:_string[]_ | 值在数据库中必须是唯一的。`class` 和 `field` 定义同上，`withFields` 可以指定其它的字段，与输入值共同用于检查是否唯一。 |

示例：

#### exists:

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'id' => [
            ['entity::exists', \App\Database\User::class]
        ]
    ];
}
```

> 检查指定 id 的用户在数据库中是否存在

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'email' => [
            ['entity::exists', \App\Database\User::class, 'email']
        ]
    ];
}
```

> 检查指定 email 的用户在数据库中是否存在

#### unique

使用该规则时，可以传递一个实体对象作为上下文对象。如果上下文中存在该对象，它会做为不可变值来计算，检查器会返回 true。否则检查器会在数据库中进行查找。

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'email' => [
            ['entity::unique', \App\Database\User::class, 'email', ['company']]
        ]
    ];
}
```

> 检查指定 email 以及 company 的组合在用户表中的记录是否唯一。

通过验证器上下文，你可以传递当前的值，这样它们不会与数据库中的当前实体发生冲突。

```php
/** @var \App\Database\User $user */
$request->setContext($user);
```

## 自定义验证规则

可以通过自定义检查器的实现来创建特定于应用程序的验证规则。

```php
namespace App\Security;

use Spiral\Database\Database;
use Spiral\Validation\AbstractChecker;

class DBChecker extends AbstractChecker
{
    public const MESSAGES = [
        'user' => '用户不存在'
    ];

    /** @var Database */
    private $db;

    /**
     * @param Database $db
     */
    public function __construct(Database $db)
    {
        $this->db = $db;
    }

    /**
     * @param $id
     * @return bool
     */
    public function user($id): bool
    {
        return $this->db->table('users')->select()->where('id', $id)->count() === 1;
    }
}
```

> 使用预建常量 `MESSAGE` 来定义自定义错误模板。

要激活自定义的检查器，需要在 `ValidationBootloader` 中注册它：

```php
namespace App\Bootloader;

use App\Security\DBChecker;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Bootloader\Security\ValidationBootloader;

class CheckerBootloader extends Bootloader
{
    public function boot(ValidationBootloader $validation)
    {
        $validation->addChecker('db', DBChecker::class);
    }
}
```

完成上述操作后，即可通过 `db::user` 使用该自定义的规则。
