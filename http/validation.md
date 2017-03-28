# Request Filters
Request filters is a model designed specially to handle user request parameters and populate the target entity (for example ORM or ODM) or return request data using `getFields`, `getField` or `__get` methods. In other words, this model maps form values to model fields. RequestFilter can define it's own set of getters, setters, accessors and validations for fields. RequestFilter utilizes functionality of InputManager to populate it's data.

> It may be best to read about [DataEntities](/components/entity.md), ORM, ODM and [Validations](/components/validation.md) first.

## Scaffolding
You can create a new `RequestFilter` by using the command `create:request user`. Generated class will look like:

```php
class UserRequest extends RequestFilter
{
    /**
     * @var array
     */
    protected $schema = [];

    /**
     * @var array
     */
    protected $setters = [];

    /**
     * @var array
     */
    protected $validates = [];
}
```

In many cases you might want to pre-create request with a specific set of fields without manually entering them every time, scaffolder config provides you ability to define shortcust for field definitions:

```php
'request'        => [
    'namespace' => 'Requests',
    'postfix'   => 'Request',
    'class'     => Declarations\RequestDeclaration::class,
    'mapping'   => [
        'int'    => [
            'source'    => 'data',
            'setter'    => 'intval',
            'validates' => [
                'notEmpty',
                'integer'
            ]
        ],
        'float'  => [
            'source'    => 'data',
            'setter'    => 'floatval',
            'validates' => [
                'notEmpty',
                'float'
            ]
        ],
        'string' => [
            'source'    => 'data',
            'setter'    => 'strval',
            'validates' => [
                'notEmpty',
                'string'
            ]
        ],
        'bool'   => [
            'source'    => 'data',
            'setter'    => 'boolval',
            'validates' => [
                'notEmpty',
                'boolean'
            ]
        ],
        'email'  => [
            'source'    => 'data',
            'type'      => 'string',
            'setter'    => 'strval',
            'validates' => [
                'notEmpty',
                'string',
                'email'
            ]
        ],
        'file'   => [
            'source'    => 'file',
            'type'      => '\Psr\Http\Message\UploadedFileInterface',
            'validates' => [
                'file::uploaded'
            ]
        ],
        'image'  => [
            'source'    => 'file',
            'type'      => '\Psr\Http\Message\UploadedFileInterface',
            'validates' => [
                "image::uploaded",
                "image::valid"
            ]
        ],
        /*{{request.mapping}}*/
    ]
],
```

Based on a such config we can now create more complex request using folling command: `spiral create:request sample -f image:image -f name:string -f test:int`

Our generated class is going to look like:

```php
/**
 * @property-read \Psr\Http\Message\UploadedFileInterface $image
 * @property-read string $name
 * @property-read int $test
 */
class SampleRequest extends RequestFilter
{
    /**
     * @var array
     */
    protected $schema = [
        'image' => 'file:image',
        'name'  => 'data:name',
        'test'  => 'data:test'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name' => 'strval',
        'test' => 'intval'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'image' => [
            'image::uploaded',
            'image::valid'
        ],
        'name'  => [
            'notEmpty',
            'string'
        ],
        'test'  => [
            'notEmpty',
            'integer'
        ]
    ];
}
```

## Using RequestFilters
Let's check out an example of request usage in controller method and then walk through it's schema definition:

```php
public function createUser(SampleRequest $request)
{
    if(!$request->isValid()) {
        dump($request->getErrors());
    }
    
    //Doing something with request data
}
```

As you can see, we declared method dependency for our request. This will automatically allow request access to InputManager and popuplate fields described in it's schema.

```php
protected $schema = [
    'image' => 'file:image',
    'name'  => 'data:name',
    'test'  => 'data:test'
];
```

Schema definition will include the target field name, it's source and origin (client name) name specified using dot notation. We can easily switch our request to read the values from query:

```php
protected $schema = [
    'name'  => 'query:name',
    'test'  => 'query:test'
];
```

In addition, we can specify the origin name using dot notation.

```php
protected $schema = [
    'name'    => 'query:user.name',  //user[status]
    'test'    => 'query:user.test'   //user[test]
];
```

RequestFilter supports different sources you can use for definition.  Any method in `InputManager` can be used as source:

```php
protected $schema = [
  'name'   => 'post:name',           //identical to "data:name"
  'field'  => 'query:field',         
  'file'   => 'file:images.preview', //Will be represented by UploadedFile Interface
  'secure' => 'isSecure'             //Alias for InputManager->isSecure()
];
```

### Input Sources
You can use following sources for your request filters:

* uri (UriInterface)
* path (PAGE URI PATH)
* method (HTTP METHOD)
* isSecure (bool)
* isAjax (bool)
* isJsonExpected (bool)
* remoteAddress (string)
* header:**origin**
* data:**origin**
* post:**origin**
* query:**origin**
* cookie:**origin**
* file:**origin**
* server:**origin**
* attribute:**origin**

## Getting request fields
If you don't want to populate any entity you can get access to request fields directly using magic getters or `getFields` method.

```php
public function doSomething(SomeRequest $request)
{
    if (!$request->isValid()) {
        //Working with errors
    }
    
    $data = $request->getFields();
    
    //You can also access some fields using magic methods
    dump($request->image);
}
```

## Errors
Every error generated by request or target entity will be mapped to it's origin name. 

```php
protected $schema = [
    'name'    => 'query:user.name'
];
```

Fox example, if the request within a given schema returns any errors related to "name" field `getErrors`, method will return the following structure:
```php
[
    'user' => [
        'name' => 'Error message.'
    ]
];
```

## Nested Validations