# Request Filters
In many cases you might want to validate user request before any further action, you can do that by manually creating Validator instance in your controllers or dedicate such functionality to RequestFilters.

> It may be best to read about [DataEntities](/components/data-entity.md) and [Validations](/components/validation.md) first.

## Scaffolding
You can create a new `RequestFilter` by using the command `create:request user` (make sure spiral/scaffolder module is installed). Generated class will look like:

```php
class UserRequest extends RequestFilter
{
    const SCHEMA    = [];
    const SETTERS   = [];
    const VALIDATES = [];
}
```

In many cases you might want to pre-create request with a specific set of fields without manually entering them every time, scaffolder config provides you ability to define shortcuts for field definitions:

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

Let's create new request: `spiral create:request sample -f image:image -f name:string -f test:int`

Our generated class is going to look like:

```php
class SampleRequest extends RequestFilter
{
    const SCHEMA = [
        'image' => 'file:image',
        'name'  => 'data:name',
        'test'  => 'data:test'
    ];

    const SETTERS = [
        'name' => 'strval',
        'test' => 'intval'
    ];

    const VALIDATES = [
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

Requests will be automatically created and populated with data received from request active in current IoC scope, we can define field/input mapping using our SCHEMA:

```php
const SCHEMA = [
    'image' => 'file:image',
    'name'  => 'data:name',
    'test'  => 'data:test'
];
```

Schema definition will include the target field name, it's source and origin (client name) name can specified using dot notation. Let's switch request to read values from query string:

```php
const SCHEMA = [
    'name'  => 'query:name',
    'test'  => 'query:test'
];
```

And demonstrate dot notation:

```php
const SCHEMA = [
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

> Read more about InputManager [here](/http/input.md).

## Getting request fields
You can always access request values using DataEntity methods:

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

> You can also set fields using `setField` method, this can be beneficial when using requests in non http environment.

## Errors
Every error generated by request or target entity will be mapped to it's origin name. 

```php
const SCHEMA = [
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
You can nest RequestFilters into each other to create more complex validations:

```php
class AddressRequest extends RequestFilter
{
    const SCHEMA = [
        'country' => 'data:countryCode',
        'city'    => 'data',
        'address' => 'data',
    ];

    const VALIDATES = [
        'country' => ['notEmpty'],
        'city'    => ['notEmpty'],
        'address' => ['notEmpty'],
    ];
}
```

Now we can use this classes as sub-requests:

```php
class DemoRequest extends RequestFilter
{
    const SCHEMA = [
        'name'    => 'data',
        'address' => AddressRequest::class
    ];

    const VALIDATES = [
        'name'    => ['notEmpty'],
        'uploads' => [
            ['notEmpty', 'message' => '[[Please upload at least one file]]']
        ]
    ];

    const SETTERS = [
        'name' => 'strval'
    ];
}
```

In a given request AddressRequest will be automatically populated based on values located in a sub-array "address" in incoming request:

```json
{
  "address":{
    "city": "value", 
    ...
  }
}
```

And will be represented as property in a parent request:

```php
dump($demoRequest->address->getField('city'));
```

> All nested requests will be validated with parent.

### Array of Sub Requests 

todo: WRITE ABOUT IT

On another end, UploadRequest will be represented as array of requests:

```php
foreach($demoRequest->uploads as $upload)
{
    dump($upload->upload->getClientFilename());
}
```

> You can find demonstration of how nested requests work [here](https://github.com/spiral/spiral/blob/master/tests/Http/RequestFilters/DemoRequestTest.php).
