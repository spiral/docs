# Spiral Validator

You can validate your data using the [spiral/validator](https://github.com/spiral/validator) component. The component 
provides an array-based DSL to construct complex validation chains.

The component contains `Checkers`, `Conditions`, and `Validation` object. 

> **Note**
> Read more about [Validation](factory.md) in the Spiral Framework.

## Installation and Configuration

To install the component:

```bash
composer require spiral/validator
```

To enable the component, you just need to add `Spiral\Validator\Bootloader\ValidatorBootloader` to the bootloaders
list, which is located in the class of your application.

```php
namespace App;

use Spiral\Validator\Bootloader\ValidatorBootloader;

class App extends Kernel
{
    protected const LOAD = [
        // ...
        ValidatorBootloader::class,
        // ...
    ];
}
```

The configuration file for this component should be located at `app/config/validator.php`:

```php
<?php

declare(strict_types=1);

use Spiral\Validator;

return [
    // Checkers are resolved using container and provide the ability to isolate some validation rules
    // under common name and class. You can register new checkers at any moment without any
    // performance issues.
    'checkers'   => [
        'type'     => Validator\Checker\TypeChecker::class,
        'number'   => Validator\Checker\NumberChecker::class,
        'mixed'    => Validator\Checker\MixedChecker::class,
        'address'  => Validator\Checker\AddressChecker::class,
        'string'   => Validator\Checker\StringChecker::class,
        'file'     => Validator\Checker\FileChecker::class,
        'image'    => Validator\Checker\ImageChecker::class,
        'datetime' => Validator\Checker\DatetimeChecker::class,
        'entity'   => Validator\Checker\EntityChecker::class,
        'array'    => Validator\Checker\ArrayChecker::class,
    ],

    // Enable/disable validation conditions
    'conditions' => [
        'absent'     => Validator\Condition\AbsentCondition::class,
        'present'    => Validator\Condition\PresentCondition::class,
        'anyOf'      => Validator\Condition\AnyOfCondition::class,
        'noneOf'     => Validator\Condition\NoneOfCondition::class,
        'withAny'    => Validator\Condition\WithAnyCondition::class,
        'withoutAny' => Validator\Condition\WithoutAnyCondition::class,
        'withAll'    => Validator\Condition\WithAllCondition::class,
        'withoutAll' => Validator\Condition\WithoutAllCondition::class,
    ],

    // Aliases are only used to simplify developer life.
    'aliases'    => [
        'notEmpty'   => 'type::notEmpty',
        'notNull'    => 'type::notNull',
        'required'   => 'type::notEmpty',
        'datetime'   => 'type::datetime',
        'timezone'   => 'type::timezone',
        'bool'       => 'type::boolean',
        'boolean'    => 'type::boolean',
        'arrayOf'    => 'array::of',
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

## Validation DSL

The default Spiral Validator accepts validation rules in form of `nested array`. The key is the `name` of the `property`
to be validated, where the value is an `array of rules` to be applied to the value sequentially:

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty', // key must not be empty
            'string'    // must be string
        ]
    ]
);

if (!$validator->isValid()) {
    dump($validator->getErrors());
}
```

The rule, in this case, is the name of the checker method or any available PHP function, which can accept `value` as the
first argument.

For example, we can use `is_numeric` directly inside your rule:

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            'notEmpty',  // key must not be empty
            'is_numeric' // must be numeric
        ]
    ]
);
```

### Extended Declaration

In many cases, you would need to declare additional rule parameters, conditions, or custom error messages. To achieve
that, wrap the rule declaration into an array (`[]`).

```php
$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty'],  // key must not be empty
            ['is_numeric'] // must be numeric
        ]
    ]
);
```

> **Note**
> You can omit the `[]` if the rule does not need any parameters.

### Checker Rules

You can split your rule name using `::` prefix, where the first part is the checker name and the second is the method name:

Let's get `Spiral\Validator\Checker\FileChecker` checker, for example:

```php
final class FileChecker extends AbstractChecker
{
    // ...
    public function exists(mixed $file): bool // -> file::exists rule
    {
        return // check if the given file exists;
    }
    
    public function uploaded(mixed $file): bool // -> file::uploaded rule
    {
        return // check if the given file uploaded;
    }
    
    public function size(mixed $file, int $size): bool // -> file::size rule
    {
        return // check the given file size;
    }
}
```

Register it in `app/config/validator.php` config file:

```php
<?php

declare(strict_types=1);

use Spiral\Validator;

return [
    'checkers' => [
        'file' => Validator\Checker\FileChecker::class,
    ],

    // Register aliases if you need to simplify developer life.
    'aliases' => [
        'file' => 'file::exists',
        'uploaded' => 'file::uploaded',
        'filesize' => 'file::size',
    ]
];
```

And use validation rules to validate a file:

```php
$validator = $validation->validate(
    ['file' => null],
    [
        'file' => [
            'file::uploaded', // you can use alias 'uploaded'
            ['file::size', 1024] // FileChecker::size($file, 1024)
        ]
    ]
);
```

### Parameters

All the values listed in rule array will be passed as rule arguments. For example, to check value using `in_array`:

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

To specify regexp pattern:

```php
$validator = $validation->validate(
    ['name' => 'b'],
    [
        'name' => [
            'notEmpty',
            ['regexp', '/^a+$/'] // aaa...
        ]
    ]
);
```

### Error Messages

Validator will render default error message for any custom rule, to set custom error message set the rule attribute:

```php
$validator = $validation->validate(
    ['file' => 'b'],
    [
        'file' => [
            'notEmpty',
            ['regexp', '/^a+$/', 'error' => 'Invalid pattern, "a+" wanted.'] // aaa...
        ]
    ]
);
```

> **Note**
> You can assign custom error messages to any rule.

#### Error Messages localization

Custom error messages will be automatically translated.

```php
// app/locale/ru/messages.php
return [
    'This value is required.' => 'Значение не должно быть пустым.',
];
```

```php
$translator->setLocale('ru');

$validator = $validation->validate(
    ['key' => null],
    [
        'key' => [
            ['notEmpty', 'error' => 'This value is required.'] // Will return ['key' => 'Значение не должно быть пустым.']
        ]
    ]
);
```

### Conditions

In some cases the rule must be activated only based on some external condition, use rule attribute `if` for this
purpose:

```php
$validator = $validation->validate(
    [
        'password' => '',
        'confirmPassword' => ''
    ],
    [
        'password' => [
            ['notEmpty']
        ],
        'confirmPassword' => [
            ['notEmpty', 'if' => ['withAll' => ['password']]]
        ]
    ]
);
```

> **Note**
> In the example, the required error on `confirmPassword` will show if `password` is not empty.

You can use multiple conditions or combine them with complex rules:

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
            ['match', 'password', 'error' => 'Passwords do not match.']
        ]
    ]
);
```

There are two composition conditions: `anyOf` and `noneOf`, they contain nested conditions:

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
            ['notEmpty', 'if' => ['anyOf' => ['withAll' => ['password'], 'withoutAll' => ['otherField']]]],
            [
                'match', 
                'password',
                'error' => 'Passwords do not match.',
                'if' => ['noneOf' => ['some condition', 'another condition']]
            ]
        ]
    ]
);
```

### Available Conditions

Following conditions available for the usage:

| Name       | Options | Description                                   |
|------------|---------|-----------------------------------------------|
| withAny    | *array* | When at least one field is not empty.         |
| withoutAny | *array* | When at least one field is empty.             |
| withAll    | *array* | When all the fields are not empty.                |
| withoutAll | *array* | When all the fields are empty.                    |
| present    | *array* | When all the fields are presented in the request. |
| absent     | *array* | When all the fields are absent in the request.    |
| noneOf     | *array* | When none of the nested conditions is met.        |
| anyOf      | *array* | When any of the nested conditions is met.         |

> **Note**
> You can create your conditions using `Spiral\Validator\ConditionInterface`.

## Validation Rules

The following validation rules are available.

> **Note**
> You can create your own validation rules using `Spiral\Validator\AbstractChecker`
> or `Spiral\Validator\CheckerInterface`.

### Rules Aliases

The most used rule-set is available thought the set of shortcuts:

| Alias      | Rule               |
|------------|--------------------|
| notEmpty   | type::notEmpty     |
| required   | type::notEmpty     |
| datetime   | datetime::valid    |
| timezone   | datetime::timezone |
| bool       | type::boolean      |
| boolean    | type::boolean      |
| arrayOf    | array::of,         |
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

### Type

> **Note**
> prefix `type::`

| Rule     | Parameters             | Description                                   |
|----------|------------------------|-----------------------------------------------|
| notEmpty | asString:*bool* - true | The value should not be empty (same as `!empty`). |
| notNull  | ---                    | The value should not be null.                     |
| boolean  | ---                    | The value has to be boolean or integer[0,1].      |

> **Note**
> All of the rules of this checker are available without the prefix.

### Required

| Rule     | Parameters             | Description                |
|----------|------------------------|----------------------------|
| notEmpty | asString:*bool* - true |The value should not be empty. |

Examples:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required', 'my::abc']
        ]);
    }
}
```

### Mixed

> **Note**
> prefix `mixed::`

| Rule       | Parameters                            | Description                                      |
|------------|---------------------------------------|--------------------------------------------------|
| cardNumber | ---                                   | Checks the credit card passed by Luhn algorithm.      |
| match      | field:*string*, strict:*bool* - false | Checks if the value matches the value from another field. |

> **Note**
> All of the rules of this checker are available without prefix.

### Address

> **Note**
> prefix `address::`

| Rule  | Parameters                                              | Description              |
|-------|---------------------------------------------------------|--------------------------|
| email | ---                                                     | Checks if the email is valid. |
| url   | schemas:*?array* - null, defaultSchema:*?string* - null | Checks if the URL is valid.   |
| uri   | ---                                                     | Checks if the URI is valid.   |

> **Note**
> `email` and `url` rules are available without `address` prefix via aliases, for `uri` use `address::uri`.

### Number

> **Note**
> prefix `number::`

| Rule   | Parameters                 | Description                                                        |
|--------|----------------------------|--------------------------------------------------------------------|
| range  | begin:*float*, end:*float* | Checks if the number is in the specified range.                       |
| higher | limit:*float*              | Checks if the value is bigger or equal to the specified one.  |
| lower  | limit:*float*              | Checks if the value is smaller or equal to the specified one. |

### String

> **Note**
> prefix `string::`

| Rule    | Parameters              | Description                                                            |
|---------|-------------------------|------------------------------------------------------------------------|
| regexp  | expression:*string*     | Checks the string using regexp.                                             |
| shorter | length:*int*            | Checks if the string length is shorter or equal to the specified value.       |
| longer  | length:*int*            | Checks if the string length is longer or equal to the specified value. |
| length  | length:*int*            | Checks if the string length is equal to the specified value.                |
| range   | left:*int*, right:*int* | Checks if the string length fits within the specified range.            |

Examples:

```php
namespace App\Filter;

use Spiral\Filters\Attribute\Input\Post;
use Spiral\Filters\Model\Filter;
use Spiral\Filters\Model\FilterDefinitionInterface;
use Spiral\Filters\Model\HasFilterDefinition;
use Spiral\Validator\FilterDefinition;

class MyFilter extends Filter implements HasFilterDefinition
{
    #[Post]
    public string $name;
    
    public function filterDefinition(): FilterDefinitionInterface
    {
        return new FilterDefinition([
            'name' => ['required', ['string::length', 5]]
        ]);
    }
}
```

### Array

> prefix `array::`

| Rule           | Parameters           | Description                                                             |
|----------------|----------------------|-------------------------------------------------------------------------|
| count          | length:*int*         | Checks if an array has a size equal to the given value.                 |
| shorter        | length:*int*         | Checks if an array has a size smaller than or equal to the given value. |
| longer         | length:*int*         | Checks if an array has a size bigger than or equal to the given value.  |
| range          | min:*int*, max:*int* | Checks if an array has a size between the given min and max.            |
| isList         |                      | Checks array is list                                                    |
| isAssoc        |                      | Checks array is assoc                                                   |
| expectedValues | values:*array*       | Checks the array values are only allowed from a list                    |

Examples:

```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'tags' => [
            ['notEmpty'],
            ['array::range', 1, 10]
        ],
        'person' => [
            ['array::isList'],
            ['array::shorter', 3],
            ['array::expectedValues', ['good', 'bad', 'ugly']]
        ]
    ];
}
```

### File Checker

> **Note**
> prefix `file::`

File checker fully supports the filename provided in a string form or using `UploadedFileInterface` (PSR-7).

| Rule      | Parameters            | Description                                                                      |
|-----------|-----------------------|----------------------------------------------------------------------------------|
| exists    | ---                   | Checks if the file exist.                                                             |
| uploaded  | ---                   | Checks if the file was uploaded.                                                      |
| size      | size:*int*            | Checks if the file size is smaller than the specified value in KB.                              |
| extension | extensions:*array*    | Checks if the file extension in whitelist. The client name of the uploaded file will be used! |                                                     |

### Image Checker

> **Note**
> prefix `image::`

The image checker extends the file checker and fully supports its features.

| Rule    | Parameters                    | Description                                                                          |
|---------|-------------------------------|--------------------------------------------------------------------------------------|
| type    | types:*array*                 | Checks if the image is within the list of the allowed image types.                          |
| valid   | ---                           | A shortcut to check if the image has an allowed type (JPEG, PNG, and GIF are allowed). |
| smaller | width:*int*, height:*int*     | Checks if the image is smaller than the specified shape (height check if optional).         |
| bigger  | width:*int*, height:*int*     | Checks if the image is bigger than the specified shape (height check is optional).          |       

### Datetime

> **Note**
> prefix `datetime::`

This checker can apply `now` value in the constructor

| Rule          | Parameters                                                                      | Description                                                            |
|---------------|---------------------------------------------------------------------------------|------------------------------------------------------------------------|
| future        | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false                        | The value has to be a date in the future.                                  |
| past          | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false                        | The value has to be a date in the past.                                    |
| format        | format:*string*                                                                 | The value should match the specified date format.                           |
| before        | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false | The value should come before the given threshold.                            |
| after         | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false | The value should come after the given threshold.                             |
| valid         | ---                                                                             | The value has to be a valid datetime definition including numeric timestamp. |
| timezone      | ---                                                                             | The value has to be a valid timezone.                                        |

> **Note**
> Setting `useMicroSeconds` into true allows to check datetime with microseconds.<br/>
> Be careful, two `new \DateTime('now')` objects will 99% have different microseconds values so they will never be
> equal.

## Custom Validation Rules

It is possible to create application-specific validation rules via custom checker implementation.

```php
namespace App\Security;

use Cycle\Database\Database;
use Spiral\Validator\AbstractChecker;

class DBChecker extends AbstractChecker
{
    public const MESSAGES = [
        // Method => Error message
        'user' => 'No such user.'
    ];

    public function __construct(
        private Database $db
    ) {
    }

    public function user(int $id): bool
    {
        return $this->db->table('users')->select()->where('id', $id)->count() === 1;
    }
}
```

> **Note**
> Use prebuild constant `MESSAGES` to define a custom error template.

To activate checker, register it in `ValidationBootloader`:

```php
namespace App\Bootloader;

use App\Security\DBChecker;
use Spiral\Boot\Bootloader\Bootloader;
use Spiral\Validator\Bootloader\ValidatorBootloader;

class CheckerBootloader extends Bootloader
{
    public function boot(ValidationBootloader $validation): void
    {
        // Register custom checker
        $validation->addChecker('db', DBChecker::class);
        
        // Register alias for checker
        $validation->addAlias('db_user', 'db::user');
    }
}
```

You can use the validation now via `db::user` (or alias `db_user`) rule.
