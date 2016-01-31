# Compositions, Aggregations, Inheritance
The ODM component does not use termin "relation(s)" to describe it's functionality but switches to the OOP definitions for [Composition, Aggregration (Reference)](https://en.wikipedia.org/wiki/Object_composition) and Inheritance. 

Please make sure that you have already read about [ODM Basics] (basics.md).

> You can also refresh your memory with regards to [Composited Types](https://en.wikipedia.org/wiki/Composite_data_type).

## Aggregations
Aggregations is probably the closest thing to classical ORM relations since they define the rule to create the selection of outer models. Just like with classical relationships, agregations can reference one or multiple models. First of all, let's try to create a document that our User can reference to "create:document post -f _id:MongoId -f userId:MongoId -f title:string -f content:string":

```php
class Post extends Document
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
        '_id'     => 'MongoId',
        'userId'  => 'MongoId',
        'title'   => 'string',
        'content' => 'string'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'title'   => ['notEmpty'],
        'content' => ['notEmpty']
    ];
}
```

To create the composition of posts in our User model, we have to modify it's schema like this:

```php
protected $schema = [
    '_id'            => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate',
    'tags'           => ['string'],
    //Aggregations
    'posts'          => [
        self::MANY => Post::class,
        ['userId' => 'self::_id']
    ]
];
```

We can also create a reverted relation in our `Post` model:

```php
protected $schema = [
    '_id'     => 'MongoId',
    'userId'  => 'MongoId',
    'title'   => 'string',
    'content' => 'string',
    'author'  => [
        self::ONE => User::class,
        ['_id' => 'self::userId']
    ]
];
```

Once the schema updates, we are able to use this aggreation via the `Document` method `aggregation` or magic `__call` function. Aggregation of many models (posts) will return a pre-configured instance of `Spiral\ODM\Entities\Collection` where the query will be based on the second element of our aggregation array.

You can also specify the aggregation query in any form supported by MongoDB including keywords `$or`, `$in`, etc. Once the aggeration request is sent, the Document will replace the constructions like "self::key" with appropriate field value (You are able to use the dot notation to get the values of the composited models "self::author.id". See next).

```php
$user = User::findOne();

dump($user->posts()->count());

foreach ($user->posts() as $post) {
    dump($post);
}
```

Aggregations differ from ORM relations because they **can't be cached** or automatically linked. This means that you have to configure every outer model manually:

```php
$post = new Post();
$post->title = 'Some post';
$post->userId = $user->_id;
$post->content = 'Some content';
$post->save();
```

Singular aggregations will return the outer Document or `null` if nothing is found:

```php
dump($post->author());

//Alternative
dump($post->aggregation('author'));
```

## Compositions and Complex Types
Compositions is one of the most powerful features within the ODM component. This functionality provides the ability to nest multiple models inside each other. It can also be combined with inheritance and atomic operations.

#### DocumentEntity
Before we jump to compositions, let's first review the class `DocumentEntity`. This class is very similar to `Document` (in fact, it's `Document` parent) but it does not provide the ActiveRecord like functionality which makes it perfect a candicate to be embedded. To create the `DocumentEntity` model, we have to execute the command "create:entity profile -f biography:text -f facebookUID:int". The generated entity looks exactly the same as the other Documents and can be configured the same way:

```php
class Profile extends DocumentEntity
{
    /**
     * @var array
     */
    protected $fillable = [
        'biography'
    ];

    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'biography'   => 'text',
        'facebookUID' => 'int'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'biography'   => ['notEmpty'],
        'facebookUID' => ['notEmpty']
    ];
}
```

#### Singural Composition
To composite a document inside another, you just need to declare the field in your schema that is linked to the composited document class. For example let's say that every user has one profile:

```php
protected $schema = [
    '_id'            => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate',
    'tags'           => ['string'],
    'profile'        => Profile::class,
    //Aggregations
    'posts'          => [
        self::MANY => Post::class,
        ['userId' => 'self::_id']
    ]
];
```

Once the schema is updated, you can use the User profile in your code:

```php
$user = User::findOne();
$user->profile->biography = 'some bio';
$user->profile->facebookUID = 2345678;

dump($user);
```

DocumentEntity data will be set up using atomic operations:

```
atomics:dynamic = array(1)
(
·    ['$set'] = array(4)
·    (
·    ·    ['profile.biography'] = string(8) some bio
·    ·    ['profile.facebookUID'] = integer(7) 2345678
·    )
)
```

You can also assign new instances of Profile to your User at any moment:

```php
$user = User::findOne();
$user->profile = new Profile([
    'biography'   => 'Some biography.',
    'facebookUID' => 4567890
]);

dump($user);
$user->save();
```

##### Mass Assignment
ODM engine provides you ability to set cascade data using setFields method of top model, to do that you have to ensure that your composited document are listed in **fillable** fields.

```php
protected $fillable = ['profile'];
```

```php
$user->setFields([
    'profile' => [
        'biography' => 'Some bio'
    ]
]);
```

#### Multiple Composition
In many cases, you will need an array compositions of the documents. To start, let's create a simple `DocumentEntity` that we store inside the User - "create:entity session -f timeCreated:MongoDate -f accessToken:string":

```php
class Session extends DocumentEntity
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
        'timeCreated' => 'MongoDate',
        'accessToken' => 'string'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'timeCreated' => ['notEmpty'],
        'accessToken' => ['notEmpty']
    ];
}
```

Now we can modify the user schema and include our composited model name as an array element (the same way as ScalarArray):

```php
protected $schema = [
    '_id'            => 'MongoId',
    'name'           => 'string',
    'email'          => 'string',
    'balance'        => 'float',
    'timeRegistered' => 'MongoDate',
    'tags'           => ['string'],
    'profile'        => Profile::class,
    'sessions'       => [Session::class],
    //Aggregations
    'posts'          => [
        self::MANY => Post::class,
        ['userId' => 'self::_id']
    ]
];
```

After the schema updates, we are able to manipulate with an array of sessions. This array is represented by the ODM class `Spiral\ODM\Entities\Compositor`:

```php
$user = User::findOne();

$user->sessions[] = new Session([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'random'
]);

dump($user);
$user->save();
```

You can also iterate though this array:

```php
$user = User::findOne();

foreach ($user->sessions as $session) {
    dumP($session->accessToken);
}
```

Additionally, Compositor provides the easy to use methods `find` and `findOne`, which can locate composited objects by it's field value(s):

```php
dump($user->sessions->findOne(['accessToken' => 'random']));
```

> Method will return `null` if no objects are found. Attention, method can only access an array of fields associated values. There isn't a mongo query like system supported here.

##### Atomic Operations
Just like `ScalarArray`, `Compositor` is **solid** by default. If you want to apply atomic operations to its' documents, you have to reset this state first:

```php
$user = User::findOne();
$user->sessions->solidState(false);
$user->sessions->push(new Session([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'newrandom'
]));

dump($user);
$user->save();
```
