## ODM Component, Basics
Spiral ODM component has some similaries to [ORM engine] (/orm/basics.md) as they both based on [DataEntity] (/components/entity.md) model and [Behaviour Schemas] (/schemas.md). However ODM removes termin "relation" and replaces it with classical [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) temninalogy. In addition, ODM classes support inheritance and also can be embedded into ORM as JSON objects. 

ODM component was mainly designed to work with MongoDB database including support of atomic operations, however `DocumentEntity` class and it's compositions can be used outside of Mongo to create OOP data representation of various structures (XML files, API responses, etc).

Spiral ODM does not use entity cache, instead of you are given with "streaming like" functionality which can be used to process huge massives of data using Document models.

## Mongo Connection and Databases
As in case with ORM you are not required to spend fortune of time to configure ODM component, the only thing you have to make sure (in case if you want to store your data in MongoDB) is that mongo connection is properly configured, to do that simply check configuration file "odm.php":

```php
'default'   => 'default',
'aliases'   => [
    'database' => 'default',
    'db'       => 'default',
    'mongo'    => 'default'
],
'databases' => [
    'default' => [
        'server'    => 'mongodb://localhost:27017',
        'profiling' => MongoDatabase::PROFILE_SIMPLE,
        'database'  => 'spiral-empty',
        'options'   => [
            'connect' => true
        ]
    ]
],
```

Configration is very similar to [DBAL] (/databases/overview.md) and fully support database aliases and controllable/contextual injections. If you wish to get direct access to MongoDatabase in your controller simple request it using `ODM->db()` factory method or via `MongoDatabase` dependency using database alias:

```php
public function index(MongoDatabase $database, ODM $odm)
{
    dump($database->getCollectionNames());
    dump($odm->db('default'));
}
```

#### DocumentEntity
The base class of ODM component which provides support for inheritance and compositions is `DocumentEntity`, such class does not have ActiveRecord functionality and mainly used 

You can create Document model by simply extending `Spiral\ODM\Document` class and via defining model behaviour using protected property "schema". To simplfy model creation, you can also use console command "create:embeddable name -f field:type ...". We can pre-create our fist model using command "create:embed data -f name:string -f value:int -f time:MongoDate", as result we will get our class in "application/classes/Database" folder.

```php
class Data extends SimpleDocument 
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

> Since we defined fillable fields, we can also use `User::create(data)`.

If everything is OK you might notice that `_id` field got populated in last dump, meaning we just pushed our data into database.

## Querying Documents




## Atomic Operations and Solid State


### Scalar Arrays


## Inheritance and Class Definition



## Compositions

## Aggregations

