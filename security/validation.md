# Validation Interfaces
You can validate your data using the `spiral/validation` component. The component provides an array-based DSL to construct
complex validation chains.

The component contains Checkers, Conditions, and Validation object. The Web and GRPC bundle of spiral includes
this component by default.

> Check [Filter/Request Object](/filters/configuration.md) for deep structural validations.

## Installation and Configuration
To install the component:

```bash
$ composer require spiral/validation
```

> Please note that the spiral/framework >= 2.6 already includes this component.

To install in spiral use bootloader `Spiral\Bootloader\Security\ValidationBootloader`.

To edit the default component configuration create and edit file `app/config/validation.php`:

```php
<?php

declare(strict_types=1);

use Spiral\Validation;

return [
    // Checkers are resolved using container and provide the ability to isolate some validation rules
    // under common name and class. You can register new checkers at any moment without any
    // performance issues.
    'checkers'   => [
        'type'     => Validation\Checker\TypeChecker::class,
        'number'   => Validation\Checker\NumberChecker::class,
        'mixed'    => Validation\Checker\MixedChecker::class,
        'address'  => Validation\Checker\AddressChecker::class,
        'string'   => Validation\Checker\StringChecker::class,
        'file'     => Validation\Checker\FileChecker::class,
        'image'    => Validation\Checker\ImageChecker::class,
        'datetime' => Validation\Checker\DatetimeChecker::class,
        'entity'   => Validation\Checker\EntityChecker::class,
    ],

    // Enable/disable validation conditions
    'conditions' => [
        'withAny'    => Validation\Condition\WithAnyCondition::class,
        'withoutAny' => Validation\Condition\WithoutAnyCondition::class,
        'withAll'    => Validation\Condition\WithAllCondition::class,
        'withoutAll' => Validation\Condition\WithoutAllCondition::class,
    ],

    // Aliases are only used to simplify developer life.
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

Use the component via provider factory:

```php
namespace App\Controller;

use Spiral\Validation;

class HomeController
{
    public function index(Validation\ValidationInterface $validation)
    {
        $validator = $validation->validate(
        // data
            [
                'key' => null
            ],
            // rules
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

> You can use the `validator` prototype property.

## The ValidatorInterface
The result of `ValidationInterface`->`validate` method is `ValidatorInterface`. The interface provides basic
API to get result errors and allows them to attach to new data or context (immutable).

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

The proper flow is valid:

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

### Validated Data
A validator can accept any array data source, but internally it will be converted into array form (unless ArrayAccess).

### Error Format
The validation component will always return one error and first fired error per key.

```php
[
    'name' => 'This field is required.',
    'key'  => 'Another error'
]
```

> Error messages can be localized using a `spiral/translator`.

## Validation DSL
The default spiral validator accepts validation rules in form of nested array. The key is the name of the property
to be validated, where the value is an array of rules to be applied to the value sequentially:

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

The rule, in this case, is the name of the checker method or any available PHP function, which can accept `value` as the first argument.

For example we can use `is_numeric` directly inside your rule:

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
In many cases, you would need to declare additional rule parameters, conditions, or custom error messages. To achieve that, wrap the rule declaration into an array (`[]`).

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

> You can omit the `[]` if the rule does not need any parameters.

### Checker Rules
You can split your rule name using `::` prefix, where first part is checker name and second is method name, for example:

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

### Parameters
All values listed in rule array will be passed as rule arguments. For example to check value using `in_array`:

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
Validator will render default error message for any custom rule, to set custom error message set the rule
attribute:

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

> You can assign custom error messages to any rule.

### Conditions
In some cases the rule must only be activated based on some external condition, use rule attribute `if` for this
purpose:

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

Name | Options | Description
--- | --- | ---
withAny | *array* | When at least one field is not empty.
withoutAny | *array* | When at least one field is empty.
withAll | *array* | When all fields are not empty.
withoutAll | *array* | When all fields are empty.
present | *array* | When all fields are presented in the request.
absent | *array* | When all fields are absent in the request.
noneOf | *array* | When none of nested conditions is met.
anyOf | *array* | When any of nested conditions is met.

> You can create your conditions using `Spiral\Validation\ConditionInterface`.

## Validation Rules
The following validation rules are available.

> You can create your own validation rules using `Spiral\Validation\AbstractChecker` or `Spiral\Validation\CheckerInterface`.

### Rules Aliases
The most used rule-set is available thought the set of shortcuts:

Alias | Rule
--- | ---
notEmpty|  type::notEmpty
required|  type::notEmpty
datetime|  datetime::valid
timezone|  datetime::timezone
bool|      type::boolean
boolean|   type::boolean
arrayOf|   array::of,
cardNumber|mixed::cardNumber
regexp|    string::regexp
email|     address::email
url|       address::url
file|      file::exists
uploaded|  file::uploaded
filesize|  file::size
image|     image::valid
array|     is_array
callable|  is_callable
double|    is_double
float|     is_float
int|       is_int
integer|   is_integer
numeric|   is_numeric
long|      is_long
null|      is_null
object|    is_object
real|      is_real
resource|  is_resource
scalar|    is_scalar
string|    is_string
match|     mixed::match

### Type
> prefix `type::`

| Rule          | Parameters             | Description         
| ---           | ---                    | ---       
| notEmpty      | asString:*bool* - true | Value should not be empty (same as `!empty`).                                             
| notNull       | ---                    | Value should not be null.                                            
| boolean       | ---                    | Value has to be boolean or integer[0,1].                                  

> All of the rules of this checker are available without prefix.

### Required
| Rule          | Parameters             | Description    
| ---           | ---                    | ---  
| notEmpty      | asString:*bool* - true | Value should not be empty.                                             

Examples:

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

### Mixed
> prefix `mixed::`

| Rule          | Parameters                            | Description                                      
| ---           | ---                                   | ---                                              
| cardNumber    | ---                                   | Check credit card passed by Luhn algorithm.      
| match         | field:*string*, strict:*bool* - false | Check if value matches value from another field.

> All of the rules of this checker are available without prefix.

### Address
> prefix `address::`

| Rule          | Parameters                    | Description              
| ---           | ---                           | ---                      
| email         | ---                           | Check if email is valid.
| url           | schemas:*?array* - null, defaultSchema:*?string* - null   | Check if URL is valid.
| uri           | ---                           | Check if URI is valid.

> `email` and `url` rules are available without `address` prefix via aliases, for `uri` use `address::uri`.

### Number
> prefix `number::`

| Rule          | Parameters                 | Description           
| ---           | ---                        | ---                   
| range         | begin:*float*, end:*float* | Check if the number is in a specified range.
| higher        | limit:*float*              | Check if the value is bigger or equal to that which is specified.
| lower         | limit:*float*              | Check if the value is smaller or equal to that which is specified.

### String
> prefix `string::`

| Rule          | Parameters              | Description           
| ---           | ---                     | ---                   
| regexp        | expression:*string*     | Check string using regexp.                  
| shorter       | length:*int*            | Check if string length is shorter or equal that specified value.              
| longer        | length:*int*            | Check if the string length is longer or equal to that specified value.                
| length        | length:*int*            | Check if the string length is equal to specified value.               
| range         | left:*int*, right:*int* | Check if the string length fits within the specified range.                  

Examples:

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

### File Checker
> prefix `file::`

File checker fully supports the filename provided in a string form or using `UploadedFileInterface` (PSR-7).

| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| exists        | ---                   | Check if file exist.
| uploaded      | ---                   | Check if file was uploaded.
| size          | size:*int*            | Check if file size less that specified value in KB.
| extension     | extensions:*array*    | Check if file extension in whitelist. Client name of uploaded file will be used!

### Image Checker
> prefix `image::`

The image checker extends the file checker and fully supports its features.

| Rule          | Parameters                | Description           |
| ---           | ---                       | ---                   |
| type          | types:*array*             | Check if the image is within a list of allowed image types.
| valid         | ---                       | Shortcut to check if the image has an allowed type (JPEG, PNG, and GIF are allowed).
| smaller       | width:*int*, height:*int* | Check if image is smaller than a specified shape (height check if optional).
| bigger        | width:*int*, height:*int* | Check if image is bigger than a specified shape (height check is optional).

### Datetime
> prefix `datetime::`

This checker can apply `now` value in the constructor

| Rule          | Parameters             | Description         
| ---           | ---                    | ---       
| future        | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false| Value has to be a date in the future.
| past          | orNow:*bool* - false,<br/>useMicroSeconds:*bool* - false| Value has to be a date in the past.
| format        | format:*string*                    | Value should match the specified date format
| before        | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false| Value should come before a given threshold.
| after         | field:*string*,<br/>orEquals:*bool* - false,<br/>useMicroSeconds:*bool* - false | Value should come after a given threshold.                
| valid         | ---                    | Value has to be valid datetime definition including numeric timestamp. 
| timezone      | ---                    | Value has to be valid timezone.         

> Setting `useMicroSeconds` into true allows to check datetime with microseconds.<br/>
Be careful, two `new \DateTime('now')` objects will 99% have different microseconds values so they will never be equal.

### Entity
> prefix `entity::`
      
Cycle ORM specific checker.

| Rule   | Parameters             | Description         
| ---    | ---                    | ---       
| exists | class:*string*, field:*string* - null, ignoreCase:*bool* - false| If an entity is presented in the db by a given PK or a custom field. `class` is an entity class name. `ignoreCase` option is only available since v2.9.
| unique | class:*string*, field:*string*, withFields:*string[]*, ignoreCase:*bool* - false| Value has to be unique. `withFields` represents an array of fields to be fetched from the validator input so all of them will be used in the unique check. `ignoreCase` option is only available since v2.9.

#### exists
Exists by PK example:
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
> Checks if the user exists by a given PK

Exists by custom field example:
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
> Checks if the user exists by a given email

#### unique
You can pass an active entity as a context object. If the value is presented in the context then it is counted
as unchanged, and the checker will return true, otherwise the checker will look into the database.

Example:
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
> It says that the given value should be unique in the database as an `email` field in a combination with a `company` value
 
With the validator context you can pass the current values so they will not conflict with the current entity in the database:
```php
/** @var \App\Database\User $user */
$request->setContext($user);
``` 

### arrayOf
Special composition checker to validate all array values:
```php
class MyRequest extends \Spiral\Filters\Filter
{
    public const VALIDATES = [
        'emails' => [
            ['arrayOf', 'address::email'] 
        ]
    ];
}
```


## Custom Validation Rules
It is possible to create application-specific validation rules via custom checker implementation.

```php
namespace App\Security;

use Spiral\Database\Database;
use Spiral\Validation\AbstractChecker;

class DBChecker extends AbstractChecker
{
    public const MESSAGES = [
        'user' => 'No such user.'
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

> Use prebuild constant `MESSAGES` to define a custom error template.

To activate checker, register it in `ValidationBootloader`:

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

You can use the validation now via `db::user` rule.
