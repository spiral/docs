## ODM Component Overview
Spiral ODM components provides ability to define [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) of DataEntity classes. Such functionality can be really useful to specify application specific database schema, as result ODM has deep integration with [MongoDB] (https://www.mongodb.org/), hovewer you might also use ODM documents for other purposes, such as API mapping, other database sources and JSON documents. 

> ODM component does not have Entity Cache as ORM one. Be careful loading related models data (see aggregations).

Before we will start, it's recommeneded to read following sections: [DataEntities] (/components/entity.md), [Behaviour Schemas] (/schemas.md). 

## Document and ActiveDocument models
Spiral ODM models can be represented by two primary classes - `Document` and `ActiveDocument`. Document model are responsible for ability to be embedded into parent model and to composite other Documents inside. ActiveDocument simply extends functionality of Document and adds ActiveRecord like methods (`find`, `findOne`, `findByPK`).

#### Document
You can create Document model by simply extending `Spiral\ODM\Document` class and via defining model behaviour using protected property "schema". To simplfy model creation, you can also use console command "create:embeddable name -f field:type ...". We can pre-create our fist model using command "create:embed data -f name:string -f value:int -f time:MongoDate", as result we will get our class in "application/classes/Database" folder.

```php
class Data extends Document 
{
    /**
     * @var array
     */
    protected $fillable = [
        
    ];

    /**
     * Entity schema.
     * 
     * @var array
     */
    protected $schema = [
        'name'  => 'string',
        'value' => 'int',
        'time'  => 'MongoDate'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'  => [
            'notEmpty'
        ],
        'value' => [
            'notEmpty'
        ],
        'time'  => [
            'notEmpty'
        ]
    ];
}
```

You can edit desired fillable, validations or schema property to add more field. Once you satified with your model, you can generate behaviour schema by running command "spiral update". ODM component will analyse your model and automatically create set of setters, getters and accessors based on component configuration, we can check such configuration (config/odm.php):

```php
'schemas'   => [
    'mutators'       => [
        'int'       => ['setter' => 'intval'],
        'float'     => ['setter' => 'floatval'],
        'string'    => ['setter' => 'strval'],
        'bool'      => ['setter' => 'boolval'],
        'MongoId'   => ['setter' => [ODM::class, 'mongoID']],
        'array'     => ['accessor' => ScalarArray::class],
        'MongoDate' => ['accessor' => Accessors\ODMTimestamp::class],
        'timestamp' => ['accessor' => Accessors\ODMTimestamp::class],
        'storage'   => ['accessor' => Accessors\StorageAccessor::class]
    ],
    'mutatorAliases' => [
    ]
]
```

You can correlate field types you declated in your model with set of default mutators. Once schema is updated, spiral will generate set of IDE tooltips (PHPStorm support only for now), which will help you to remember what fields and accessors you have in your model.

> Document model still uses mongo atomic operations, you can change such behaviour or ignore it.

We are ready use such model in our code, since there is Mongo Collection associated to it (we need ActiveDocument for that) we can use our entity as usual DataEntity class.

```php
public function index()
{
    $model = new Data();

    $model->name = 900;
    $model->setValue('12');

    //Since ODM gave us time accessor we can do that:
    $model->time->setTimezone('America/Los_Angeles');
    $model->time = 'tomorrow 10am';

    dump($model);
}
```

> You might notice that every value got type casted, this is required since MongoDB needs string types.

#### ActiveDocument
ActiveDocument models are almost identical in it's definition to Document one, it only provides two additional properties "collection" and "database" which you can define to specify where your model data must be stored into. By default spiral will generate collection name based on class and use default database. To generate ActiveModel class we have to run command 'create:document'. Since our documents are going to be stored in MongoDB we have to specify `_id` field. Let's try to execute command "create:document user -f id:MongoId -f name:string -f email:string -f balance:float", as result:

```php
class User extends ActiveDocument 
{
    /**
     * @var array
     */
    protected $fillable = [
        'name',
        'email',
        'balance'
    ];

    /**
     * Entity schema.
     * 
     * @var array
     */
    protected $schema = [
        'id'      => 'MongoId',
        'name'    => 'string',
        'email'   => 'string',
        'balance' => 'float'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => [
            'notEmpty'
        ],
        'email'   => [
            'notEmpty'
        ],
        'balance' => [
            'notEmpty'
        ]
    ];
}
```

> You can use same validation principles as in ORM.

And again, you will have to run command "spiral update" to register such entity in ODM component. In my case i'v specified what fields can be fillable. Once done, we can try to create our first document and store it into Mongo collection ("users").

```php
public function index()
{
    $user = new User();
    $user->name = 'Anton';
    $user->email = 'test@email.com';
    $user->balance = mt_rand(0, 10000) / 100;
    $user->save();

    dump($user);
}
```

> Since we defined fillable fields, we can also use User::create(data).

If everything is OK you might notice that `_id` field got populated in last dump, meaning we just pushed our data into database.

## Mongo Queries



## Atomic Operations and Solid State


### Scalar Arrays


## Inheritance and Class Definition



## Compositions

## Aggregations

