## ODM Component, Basics
Spiral ODM component has some similaries to [ORM engine] (/orm/basics.md) as they both based on [**DataEntity**] (/components/entity.md) model and [Behaviour Schemas] (/schemas.md). However ODM removes termin "relation" and replaces it with classical [compositions and aggregations] (https://en.wikipedia.org/wiki/Object_composition) temninalogy. In addition, ODM classes support inheritance and also can be embedded into ORM as JSON objects. 

ODM component was mainly designed to work with MongoDB database including support of atomic operations, however `DocumentEntity` class and it's compositions can be used outside of Mongo scope to create OOP data representation of various structures (XML files, API responses, etc).

Spiral ODM does not use entity cache like in ORM, instead you are given with "streaming like" functionality which can be used to process huge massives of data using Document models.

## Mongo Connection and Databases
As in case with ORM you are not required to spend fortune of time to configuring ODM component, the only thing you have to make sure (in case if you want to store your data in MongoDB) is that mongo connection is properly set, to do that simply check configuration file "odm.php" located inside your "applicatio/config" directory:

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

Configration is very similar to [DBAL] (/databases/overview.md) and fully support database aliases and controllable/contextual injections. If you wish to get direct access to `MongoDatabase` in your controller simply request it using `ODM->db()` factory method or via `MongoDatabase` dependency using database alias:

```php
public function index(MongoDatabase $database, ODM $odm)
{
    dump($database->getCollectionNames());
    dump($odm->db('default'));
}
```

> `MongoDatabase` class extends original `MongoDB` class so you can use it as regular mongo database.

## Document
To define your first model related to MongoDB collection we only have to create/generate declaration of Document class whith desired set of fields listed in model **schema**. We can either create such model manually or generate it using console command "create:document", let's create our first model User, as in case with ORM we are able to pre-define set of desired fields: "create:document user -f id:MongoId -f name:string -f email:string -f balance:float". Resulted entity will be located in application/classes/Database/User.php:

```php
class User extends Document 
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
        'id'      => 'MongoId',
        'name'    => 'string',
        'email'   => 'string',
        'balance' => 'float'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name'    => ['notEmpty'],
        'email'   => ['notEmpty'],
        'balance' => ['notEmpty']
    ];
}
```

You might notice that declaration is very similar to ORM Record, hovewer ODM component does not affect MongoDB schema, but rather stores it in your application. 

> You can read more about entity configuration (fillable, hidden, secured and validates properties) in a section declared to [DataEntity] (components/entity.md).

#### Schema
ODM classes Document (and DocumentEntity, see next) will only allow you to manipulate with set of fields described in model schema. Declaration of field can be performed by simply creating associated array between field name and it's type. 

```php
protected $schema = [
    'id'      => 'MongoId',
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float'
];
```

You can associate your field with any desired type, however, since MongoDB does not allow dynamic typing we have to make sure that every value listed in schema is always typecased into specified type. You can solve such problem by utilizing model getters and setters or simply using set of pre-declared types, check ODM config:

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

As you can see, every string field will automatically get associated setter `setval` while schema update.

> Schema can also contain aggregations and compositions, you can read about it below.

#### Default Values
ODM will automatically force default values based on typecasted null value (string => "", int => 0), you can set your own default values using model property "defaults":

```php
protected $defaults = [
    'balance' => 100.00    
];
```

#### Associated Collection and Database
By default spiral will associate your `Document` model with the default ODM database and collection which name are generated based on class name (in our case User => users). You can alter both of this values by declaring non empty properties `database` and `collection`.

> If you unsure what collection will be used based on model name - simply force `collection` for every Document, it's not going to harm anyone but will make your code more readable.

#### Indexes
To declare indexes to be created in associated collection simply declare them in `indexes` property of your Document. Declaration form is similar to MongoCollection
ensureIndexes method. Indexes will be created as moment of schema update (see next).

```php
protected $indexes = [
    ['email' => 1, '@options' => ['unique' => true]],
    ['name' => 1, 'balance' => -1]
];
```

> Use "@options" key to specify index options.

## Schema Update
Once you created your first Document or Documents, you have to register them inside your ODM component schema. Such operation called "schema update" and does not require any operation rather then rutting console command "spiral update" or "spiral up". Schema update will automatically located all your Document and DocumentEntity models, analyze them, create automatic set of mutators and store information about compiled behaviours inside application memory.

The only database related operation performed while schema update - ensuring that requested indexes exists in associated collection.

> Spiral Framework in addion will generate virtual documentation for your models, so you IDE will highligt every aviable field and composition.

#### Modifying Documents schema
Since ODM schema does not stored in database it can be efficiently updated without any storage change, as result you are able to add new columns, compositions and etc into your Documents at any moment - simply declare them in your model schema and update ODM after.

```php
protected $schema = [
    'id'      => 'MongoId',
    'name'    => 'string',
    'email'   => 'string',
    'balance' => 'float',
    'data'    => 'string'
];
```
