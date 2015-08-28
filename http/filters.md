# Request Filters
Request filters is models specially designed to handle user request perameters and populate target entity (for example ORM or ODM). In other words, this is mapper
of form values to model fields. RequestFilter can define it's own set of getters, setters, accessors and validations for fields. RequestFilter utilizes functionaly of
InputManager to populate it's fields.
> You might read about DataEnties, ORM, ODM and Validations first.

## Scaffolding
You can create new `RequestFilter` using command `create:request name` as in case with ORM and ODM entities you can pre-specify fields to be grabbed from request. 
If you wish to generate request for existed database entity simply use key '-e'.

Let's look to some examples, first of all we can define some database entity:

```php
class User extends Record
{
    /**
     * @var array
     */
    protected $fillable = [
        'name',
        'status'
    ];
    
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'id'      => 'primary',
        'name'    => 'string',
        'status'  => 'enum(active,blocked)'
    ];
}
```

Now using console command `create:request user -e user` we will get something like that:

```php
/**
 * @property string $name
 * @property string $status
 */
class UseryRequest extends RequestFilter
{
    /**
     * Input schema.
     *
     * @var array
     */
    protected $schema = [
        'name'    => 'data:name',
        'status'  => 'data:status'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name'    => 'trim',
        'status'  => 'trim'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => [
            'notEmpty',
            'string'
        ],
        'status'  => [
            'notEmpty',
            'string'
        ]
    ];

    /**
     * {@inheritdoc}.
     *
     * @param \Database\User $entity Entity to be populated with request data.
     * @return bool
     */
    public function populate(EntityInterface $entity)
    {
        if (!parent::populate($entity)) {
            return false;
        }

        return $entity->isValid();
    }
}
```

Let's try to view example of request usage in controller method and then walk thought it's schema definition:

```php
public function createUser(UserRequest $request)
{
    $user = new User();
    if(!$request->populate($user)) {
        dump($request->getErrors());
    }
}
```

As you can see we declared method dependency for our request, this will automatically let request access to InputManager and popuplate it's fields described in it's schema.

```php
protected $schema = [
    'name'    => 'data:name',
    'status'  => 'data:status'
];
```

Schema definition includes target field name, it's source and origin name specified using dot notation. We can easlity switch our request to read values from query:

```php
protected $schema = [
    'name'    => 'query:name',
    'status'  => 'query:status'
];
```

In addition to that we can specify origin name using dot notation.

```php
protected $schema = [
    'name'    => 'query:user.name',  //user[status]
    'status'  => 'query:user.status' //user[status]
];
```

RequestFilter support differnt sources you can use for defininition, basicaly any method in `InputManager` can be used as source:
```php
protected $schema = [
  'name'   => 'post:name',           //identical to "data:name"
  'field'  => 'query:field',         
  'file'   => 'file:images.preview', //Will be represented by UploadedFile Interface
  'secure' => 'isSecure'             //Alias for InputManager->isSecure()
];
```

**Other sources you can use:** uri (UriInterface), path (PAGE URI PATH), method (HTTP METHOD), isSecure (bool), isAjax (bool), isJsonExpected (bool), remoteAddress (string), header:origin, data:origin, post:origin, query:origin, cookie:origin, file:origin, server:origin, attribute:origin.

## Populating target Entity
Once request generated you can it's values mapped to approprate fields to populate some database entity, you can either do it manually via get/setFields or use request `populate` method which will copy fields from request to target entity and validate it to make sure that everything is ok.

You can put your custom logic into populate method to handle more complex scenarious for example file uploads. 

```php
/**
 * @property string                $name
 * @property UploadedFileInterface $image
 */
class UserRequest extends RequestFilter
{
    /**
     * Input schema.
     *
     * @var array
     */
    protected $schema = [
        'name'  => 'data:name',
        'image' => 'file:picture'
    ];

    /**
     * @var array
     */
    protected $setters = [
        'name'   => 'trim',
        'status' => 'trim',
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'  => [
            'notEmpty',
            'string'
        ],
        'image' => [
            'image::uploaded',
            'image::valid'
        ]
    ];

    /**
     * {@inheritdoc}.
     *
     * @param \Database\User $entity Entity to be populated with request data.
     * @return bool
     */
    public function populate(EntityInterface $entity)
    {
        if (!parent::populate($entity)) {
            return false;
        }

        //Doing something with image
        $this->image->moveTo('...');


        return $entity->isValid();
    }
}
```

## Errors
Every error generated by request or target entity will be mapped to it's origin name. 

```php
protected $schema = [
    'name'    => 'query:user.name'
];
```

Fox example if request with given schema will have any errors related to "name" field `getErrors` method will return following structure:
```php
[
    'user' => [
        'name' => 'Error message.'
    ]
];
```
