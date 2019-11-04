# Validation - WIP
You can validate your data using `spiral/validation` component. The component provides array based DSL to construct
complex validation chains.

The component contains of Checkers, Conditions and Validation object. The Web and GRPC bundle of spiral includes
this component by default.

## Installation and Configuration
The component can be installed:

```bash
$ composer install spiral/validation
```

To install in spiral use bootloader `Spiral\Bootloader\Security\ValidationBootloader`. 

To edit the default component configuration create and edit file `app/config/validation.php`:

```php
<?php

declare(strict_types=1);

use Spiral\Validation;

return [
    // Checkers are resolved using container and provide ability to isolate some validation rules
    // under common name and class. You can register new checkers at any moment without any
    // performance issues.
    'checkers'   => [
        'type'    => Validation\Checker\TypeChecker::class,
        'number'  => Validation\Checker\NumberChecker::class,
        'mixed'   => Validation\Checker\MixedChecker::class,
        'address' => Validation\Checker\AddressChecker::class,
        'string'  => Validation\Checker\StringChecker::class,
        'file'    => Validation\Checker\FileChecker::class,
        'image'   => Validation\Checker\ImageChecker::class,
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
        'datetime'   => 'type::datetime',
        'timezone'   => 'type::timezone',
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

> You can use `validator` prototype property.

## The ValidationInterface
The result of `ValidationInterface`->`validate` method is `ValidationInterface`. The interface provides basic 
api to get result errors and allows to attach to new data or context (immutable).

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
Validator can accept any array data source, but internally it will be converted into array form (unless ArrayAccess). 

### Error Format
The validation component will always return one error and first fired error per key.

```php
[
    'name' => 'This field is required.',
    'key'  => 'Another error'
]
```

> Error messages can be localized using `spiral/translator`.

## Validation DSL



## Validation Rules and Messages
To define multiple validation rules for one field:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class, 
        [
            'data'  => [
                'name' => $this->input->query('name')
            ],
            'rules' => [
                'name' => [
                    'notEmpty',
                    'email'     //We want our name field to contain a valid email address
                ]
            ]
        ]
    );

    dump($validator->getErrors());
}
```


```php
'rules' => [
    'name' => [
        'notEmpty',
        'is_numeric'
    ]
]
```


```php
[
    'name' => "Condition 'is_numeric' does not meet."
]
```

### Validation Messages and Complex Rule Definition
Based on the given example, validator will create a default error message if some condition fails. We can provide a custom error message by switching the rule definition from a simple to complex form. For this purpose, our rule must be defined as an array and must declare an additional key "message":

```php
'rules' => [
    'name' => [
        'notEmpty',
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

Every existing rule can be declared in a complex form by putting it's name into an array. Let's rewrite our rules to demonstrate that:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['is_numeric', 'message' => 'Must be numeric.']
    ]
]
```

> You can only use a simple rule definition form when you want to use a default error message without any rule parameters (see below). Validator performs an automatic translation only for default error messages!

### Rule parameters
It was mentioned before that the complex form lets you set custom error message and provide a set of arguments to be passed into your validation method. We can perform that by listing the necessary arguments after the rule name/function. Let's try to use the 'in_array' function for example:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['in_array', ['a', 'b', 'c'], 'message' => 'Invalid value.']
    ]
]
```

> You can use any function as a validation rule, including your own method declared in a form "Class::method". The only requirement is that value needs to be the first argument.

Let's try to create our own validation method in our controller or service.

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class, 
        [
            'data'  => [
                'name' => $this->input->query('name')
            ],
            'rules' => [
                'name' => [
                    ['notEmpty'],
                    ['Controllers\HomeController::validate', 'abc', 'message' => 'Invalid value.']
                ]
            ]
        ]
    );

    dump($validator->getErrors());
}

public function validate($value, $compare)
{
    return $value == $compare;
}
```

> As you can see, we declared our method as non static. This is because Validator will resolve the rule class using container. This allows us to create specialized services to perform logical validations (for example ensure that outer record exists by it's ID) and much more (technically you can use something like that UserService::findByPK to validate foreign key value).

## Checkers
In some cases, you may want to create a set of rules with predefined error messages and skip the step where you have to create an alias for every rule. For such purposes, the spiral Validator lets you create your own `Checker` class. Checkers are dedicated to organizing a set of validation rules with their error messages and provide access to such validations using a simple prefix.

If you check the validation configuration file, you will see some checkers already present:

```php
'checkers'        => [
    "type"     => Checkers\TypeChecker::class,
    "required" => Checkers\RequiredChecker::class,
    "number"   => Checkers\NumberChecker::class,
    "mixed"    => Checkers\MixedChecker::class,
    "address"  => Checkers\AddressChecker::class,
    "string"   => Checkers\StringChecker::class,
    "file"     => Checkers\FileChecker::class,
    "image"    => Checkers\ImageChecker::class,
],
```

Every checker must has it's own name. This name can be used as prefix (using :: separator) while defining the validation rules. For example, StringChecker provides a method named "regexp" that has the ability to apply regular expression to a field value. Lets try it:

```php
/**
 * Check string using regexp.
 *
 * @param string $string
 * @param string $expression
 * @return bool
 */
public function regexp($string, $expression)
{
    return is_string($string) && preg_match($expression, $string);
}
```

```php
'rules' => [
    'name'  => [
        ['notEmpty'],
        ['string::regexp', '/^a+$/i']
    ]               
]
```

As you may have noticed, the default error message will look something like "Your value does not match the required pattern.". This is possible because Checkers allow us to define their own default messages. Let's try to create our own Checker (every Checker has Translation support by default):

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    const MESSAGES = [
        'abc' => '[[Invalid value, only a,b,c are allowed.]]'
    ];

    public function abc($value): bool
    {
        return in_array($value, ['a', 'b', 'c']);
    }
}
```

> You may notice that the error message for method check is surrounded by `[[]]`. This technique allows spiral Translator to index and pre-cache checker translations. Read more about it in the section dedicated to Translator.

Now we can register our checker in the validation config under the name "my" and use it our rules:

```php
'rules' => [
    'name' => [
        'notEmpty',
        'my::abc'
    ]
]
```

> You can create aliases for checker methods.

Besides the ability to aggregate many validation methods under one roof, Checkers is more similar with integration with Validator than the usual validation functions. We can demonstrate such ability by accessing the validator instance from the inside of our checker method:

```php
class MyChecker extends Checker
{
    /**
     * Default error messages associated with checker method by name.
     *
     * @var array
     */
    const MESSAGES = [
        'abc'   => '[[Invalid value, only a,b,c are allowed.]]',
        'equal' => '[[Two fields must equal.]]'
    ];

    public function abc($value): bool
    {
        return in_array($value, ['a', 'b', 'c']);
    }

    public function equal($value, string $field): bool
    {
        return $value == $this->getValidator()->getValue($field, null);
    }
}
```

We can now modify our validation code within controller:

```php
protected function indexAction(FactoryInterface $factory)
{
    /**
     * @var ValidatorInterface $validator
     */
    $validator = $factory->make(
        ValidatorInterface::class,
        [
            'data'  => [
                'name'  => $this->input->query('name'),
                'email' => $this->input->query('email'),
            ],
            'rules' => [
                'name'  => [
                    'notEmpty',
                    'my::abc'
                ],
                'email' => [
                    'notEmpty',
                    ['my::equal', 'name']
                ]
            ]
        ]
    );

    dump($validator->getErrors());
}
```

As result, we have a method that will check two validation fields.

## Validation Rules
The following validation rules are available.

### Rules Aliases
The most used rule-set is available thought the set of shortcuts:

Alias | Rule
--- | ---
notEmpty|  type::notEmpty
required|   type::notEmpty
datetime|  type::datetime
timezone|  type::timezone
bool|      type::boolean
boolean|   type::boolean
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

### Type Checker- prefix type::
| Rule          | Parameters             | Description         
| ---           | ---                    | ---       
| notEmpty      | asString:*bool* - true | Value should not be empty (same as `!empty`).                                             
| notNull       | ---                    | Value should not be null.                                            
| boolean       | ---                    | Value has to be boolean or integer[0,1].                            
| datetime      | ---                    | Value has to be valid datetime definition including numeric timestamp. 
| timezone      | ---                    | Value has to be valid timezone.                                        

> All of the rules of this checker are available without prefix.

### Required Checker - prefix required::
| Rule          | Parameters             | Description    
| ---           | ---                    | ---  
| notEmpty      | asString:*bool* - true | Value should not be empty.                                             
| with          | with:*array*           | Check if field is not empty but only if any of listed fields presented or not empty.
| withAll       | with:*array*           | Check if field is not empty but only if all of listed fields presented and not empty.
| without       | without:*array*        | Check if field is not empty but only if one of listed fields missing or empty.
| withoutAll    | without:*array*        | Check if field is not empty but only if all of listed fields missing or empty.

> All of the rules of this checker are available without prefix.

Examples:

```php
'rules' => [
    'name'  => [
        ['notEmpty'],
        ['my::abc']
    ],
    'email' => [
        ['required::with', ['name']], //Only required when name is not empty
        ['email']
    ]
]
```

### Mixed Checker - prefix mixed::
| Rule          | Parameters                            | Description                                      
| ---           | ---                                   | ---                                              
| cardNumber    | ---                                   | Check credit card passed by Luhn algorithm.      
| match         | field:*string*, strict:*bool* - false | Check if value matches value from another field. 

> All of the rules of this checker are available without prefix.

### Address Checker - prefix address::
| Rule          | Parameters                    | Description              
| ---           | ---                           | ---                      
| email         | ---                           | Check if email is valid. 
| url           | requireScheme:*bool* - true   | Check if URL is valid. 

> All of the rules of this checker are available without prefix.

### Number Checker - prefix number::
| Rule          | Parameters                 | Description           
| ---           | ---                        | ---                   
| range         | begin:*float*, end:*float* | Check if the number is in a specified range.
| higher        | limit:*float*              | Check if the value is bigger or equal to that which is specified.
| lower         | limit:*float*              | Check if the value is smaller or equal to that which is specified.


### String Checker - prefix string::
| Rule          | Parameters              | Description           
| ---           | ---                     | ---                   
| regexp        | expression:*string*     | Check string using regexp.                  
| shorter       | length:*int*            | Check if string length is shorter or equal that specified value.              
| longer        | length:*int*            | Check if the string length is longer or equal to that specified value.                
| length        | length:*int*            | Check if the string length is equal to specified value.               
| range         | left:*int*, right:*int* | Check if the string length fits within the specified range.                  

Examples:

```php
'rules' => [
    'name' => [
        ['notEmpty'],
        ['string::length', 5] 
    ]
]
```

### File Checker - prefix file::
File checker fully supports the filename provided in a string form or using `UploadedFileInterface` (PSR-7).
This makes the checker very useful for uploading files.

| Rule          | Parameters            | Description           
| ---           | ---                   | ---                   
| exists        | ---                   | Check if file exist.
| uploaded      | ---                   | Check if file was uploaded.
| size          | size:*int*            | Check if file size less that specified value in KB.
| extension     | extensions:*array*    | Check if file extension in whitelist. Client name of uploaded file will be used!

### Image Checker - prefix image::
Image checker extends the file checker and fully supports it's features.

| Rule          | Parameters                | Description           |
| ---           | ---                       | ---                   |
| type          | types:*array*             | Check if image is within a list of allowed image types.
| valid         | ---                       | Shortcut to check if the image has a valid type (JPEG, PNG and GIF are allowed).
| smaller       | width:*int*, height:*int* | Check if image is smaller than a specified shape (height check if optional).
| bigger        | width:*int*, height:*int* | Check if image is bigger than a specified shape (height check is optional).







## Checker Conditions - WIP
Sometimes we want to validate field only in certain cases, for example when creating a blog post you want it to have a 
thumbnail. Thumbnail is required for a new post, but is optional when you update it (unless you upload a new thumnail for it).

Conditions are provided by `\Spiral\Validation\CheckerConditionInterface` interface, to see basic implementation refer to `\Spiral\Validation\Prototypes\AbstractCheckerCondition` class
```php
interface CheckerConditionInterface
{
    /**
     * @param ValidatorInterface $validator
     *
     * @return CheckerConditionInterface
     */
    public function withValidator(ValidatorInterface $validator): CheckerConditionInterface;

    /**
     * @return bool
     */
    public function isMet(): bool;
}
```

### Example usage
Request filter:
```php
class Request extends RequestFilter
{
    const SCHEMA = ['file' => 'file:image'];

    const VALIDATES = [
        'file' => [
            ['file::uploaded', 'condition' => NewPostOrUpdatedFileCondition::class],
            ['image::valid', 'condition' => NewPostOrUpdatedFileCondition::class]
        ],
    ];

    public function getFile(): UploadedFileInterface
    {
        return $this->file;
    }
}
```

Condition file:

```php
class NewPostOrUpdatedFileCondition extends AbstractCheckerCondition
{
    use FileTrait;

    /**
     * @return bool
     */
    public function isMet(): bool
    {
        $context = $this->validator->getContext();

        /** @var \Spiral\ORM\RecordEntity $entity */
        $entity = $context['entity'] ?? null;

        /** @var \Psr\Http\Message\UploadedFileInterface $entity */
        $file = $context['file'] ?? null;

        //Entity is new
        if (empty($entity) || !$entity->isLoaded()) {
            return true;
        }

        //File is provided and uploaded
        if (!empty($file) && $this->isUploaded($file)) {
            return true;
        }

        return false;
}
```
So in this case request filter will require file when there's no entity yet or (entity exists) file is uploaded again.
