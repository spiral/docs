# Compositions, Aggreagations, Inheritance
ODM component does not use terming relation(s) to describe it's functionality but switches to OOP definitions of [Composition, Aggregration] (https://en.wikipedia.org/wiki/Object_composition) and Inheritance. 

## Aggregations
Aggregations is probably the closest to classical ORM relations as it defines rule to perform selection of outer models, as in case wiuth classical relations agregations can reference to one or multiple models. First of all, let's try to create model our User can reference to "spiral create:document post -f _id:MongoId -f userId:MongoId -f title:string -f content:string":

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
        '_id'      => 'MongoId',
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

To create composition of posts in our User model, we have to modify it's schema in such way:

```php
protected $schema = [
    '_id'             => 'MongoId',
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

We can also create reverted relation in our `Post` model:

```php
protected $schema = [
    '_id'      => 'MongoId',
    'userId'  => 'MongoId',
    'title'   => 'string',
    'content' => 'string',
    'author'  => [
        self::MANY => Post::class,
        ['_id' => 'self::userId']
    ]
];
```

After schema update we are able to use such aggreation using `Document` method `aggregation` or magic `__call` function. Aggregation of many models (posts) will return us pre-configured instance of Collection where query will be based on second element of our aggregation array.

You can specify aggregation query in any form supported by MongoDB including keywords `$or`, `$in` and etc. At moment of aggeration request, Document will replace constructions "self::key" with appropriate field value (you are able to use dot notation to get values of composited models "self::author.id", see next).

```php
$user = User::findOne();

dump($user->posts()->count());

foreach ($user->posts() as $post) {
    dump($post);
}
```

Aggreations are differs from ORM relations as they **can not be cached** or automatically linked, meaning you have to set every model key manually:

```php
$post = new Post();
$post->title = 'Some post';
$post->userId = $user->_id;
$post->content = 'Some content';
$post->save();
```

## Compositions
Compositions is one of the most powerful featured of ODM component, such thing provides ability to nest multiple models inside each other. It can also be combined with inheritance and atomic operations.

#### DocumentEntity
Before we will jump to compositions, let's review class `DocumentEntity`, such class is very similar to Document (in a fact it's Document parent) but it does not provide ActiveRecord functionality, which makes it perfect candicate to be embedded. To create DocumentEntity model, let's try to execute command "create:entity profile -f biography:text -f facebookUID:int", generated entity looks exacly the same as other Documents and can be configured same way, however it does not have associated collection.

```php
class Profile extends DocumentEntity
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
To composite one document inside another, you are simply need to declare field in your schema linked to composited document class, for example let's say that every user has one profile:

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

Once schema is updates, you are able to user User profile in your code:

```php
$user = User::findOne();
$user->profile->biography = 'some bio';
$user->profile->facebookUID = 2345678;

dump($user);
```

As in other cases DocumentEntity data will set using atomic operataions:

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

You can also assign new instance of Profile to your User at any moment:

```php
$user = User::findOne();
$user->profile = new Profile([
    'biography'   => 'Some biography.',
    'facebookUID' => 4567890
]);

dump($user);
$user->save();
```

#### Multiple Composition
You can also create array compositions of documents. To start, let's create a simple `DocumentEntity` we would to store inside the user "session -f timeCreated:MongoDate -f accessToken:string":

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

Now we only need to modify user schema and include our composited model name as array element (same way as for ScalarArray):

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

After schema update we able to manipulate with array of sessions. Such array will be represented by ODM class `Spiral\ODM\Entities\Compositor`:

```php
$user = User::findOne();

$user->sessions[] = new Session([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'random'
]);

dump($user);
$user->save();
```

You can also iterate thought such array:

```php
$user = User::findOne();

foreach ($user->sessions as $session) {
    dumP($session->accessToken);
}
```

In addition to that, Compositor provides simplistic method `find` and `findOne` which can be used to locate composited object by it's field value(s):

```php
dump($user->sessions->findOne(['accessToken' => 'random']));
```

> Method will return `null` if no objects can be fond. Attention, method can only access array of fields associated values, no mongo query like system can be supported here.

##### Atomic Operations
As in case with `ScalarArray`, `Compositor` are stated as **solid** by default, if you wish to apply atomic operations to it's documents you have to reset this state first:

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

