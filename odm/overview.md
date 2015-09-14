## ODM Component Overview
Spiral ODM components provides ability to define [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) of DataEntity classes. Such functionality can be really useful to specify application specific database schema, as result ODM has pre-built integration with [MongoDB] (https://www.mongodb.org/), hovewer you might also use ODM documents for other purposes, such as API mapping, other database sources and JSON documents. 
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

You can correlate field types you declated in your model with set of default mutators. Once schema is updated, spiral will generate set of IDE tooltips (PHPStorm only for now), which will help you to remember what fields and accessors you have in your model.

> Document model still uses mongo atomic operations, you can change such behaviour or ignore it.

We are ready use such model in our code, since there is Mongo Collection associated to it (we need ActiveDocument for that) we can use our entity as usual DataEntity class.

```php
public function index()
{
    $model = new Data();
    $model->setValue('12')->setName(900);

    //Since ODM gave us time accessor we can do that:
    $model->time->setTimezone('America/Los_Angeles');
    $model->time = 'tomorrow 10am';

    dump($model);
}
```

> You might notice that every value got type cases, this is required since MongoDB needs string types.



#### ActiveDocument

## Mongo Queries

## Compositions

## Aggregations

## Inheritance

