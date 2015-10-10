# Compositions, Aggreagations, Inheritance
ODM component does not use terming relation(s) to describe it's functionality but switches to OOP definitions of [Composition, Aggregration] (https://en.wikipedia.org/wiki/Object_composition) and Inheritance. 

Make sure you already read about [ODM Basics] (basics.md).

## Aggregations
Aggregations is probably the closest to classical ORM relations as they define rule to create selection of outer models, as in case with classical relationships, agregations can reference to one or multiple models. First of all, let's try to create document our User can reference to "create:document post -f _id:MongoId -f userId:MongoId -f title:string -f content:string":

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

To create composition of posts in our User model, we have to modify it's schema in such way:

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

We can also create reverted relation in our `Post` model:

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

After schema update we are able to use such aggreation using `Document` method `aggregation` or magic `__call` function. Aggregation of many models (posts) will return us pre-configured instance of `Spiral\ODM\Entities\Collection` where query will be based on second element of our aggregation array.

You can specify aggregation query in any form supported by MongoDB including keywords `$or`, `$in` and etc. At moment of aggeration request, Document will replace constructions like "self::key" with appropriate field value (you are able to use dot notation to get values of composited models "self::author.id", see next).

```php
$user = User::findOne();

dump($user->posts()->count());

foreach ($user->posts() as $post) {
    dump($post);
}
```

Aggreations are differs from ORM relations as they **can not be cached** or automatically linked, meaning you have to configure every outer model manually:

```php
$post = new Post();
$post->title = 'Some post';
$post->userId = $user->_id;
$post->content = 'Some content';
$post->save();
```

Singilar aggregations will return outer Document or `null` if nothing can be found:

```php
dump($post->author());

//Alternative
dump($post->aggregation('author'));
```

## Compositions
Compositions is one of the most powerful features of ODM component, such functionality provides ability to nest multiple models inside each other. It can also be combined with inheritance and atomic operations.

#### DocumentEntity
Before we will jump to compositions, let's review class `DocumentEntity`, such class is very similar to `Document` by it's definition (in a fact, it's `Document` parent) but it does not provide ActiveRecord like functionality which makes it perfect candicate to be embedded. To create `DocumentEntity` model, we have to execute command "create:entity profile -f biography:text -f facebookUID:int", generated entity looks exacly the same as other Documents and can be configured same way:

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
To composite one document inside another, you are simply required to declare field in your schema linked to composited document class, for example let's say that every user has one profile:

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

Once schema is updated, you are able to use User profile in your code:

```php
$user = User::findOne();
$user->profile->biography = 'some bio';
$user->profile->facebookUID = 2345678;

dump($user);
```

As in other cases DocumentEntity data will be set using atomic operataions:

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
In many cases you will need an array compositions of documents. To start, let's create a simple `DocumentEntity` we would to store inside the User - "create:entity session -f timeCreated:MongoDate -f accessToken:string":

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

After schema update we able to manipulate with array of sessions. Such array are represented by ODM class `Spiral\ODM\Entities\Compositor`:

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

Additionally, Compositor provides simplistic methods `find` and `findOne` which can be used to locate composited object by it's field value(s):

```php
dump($user->sessions->findOne(['accessToken' => 'random']));
```

> Method will return `null` if no objects can be found. Attention, method can only access array of fields associated values, no mongo query like system can be supported here.

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

## Inheritance
In addition to aggregation and composition ODM engine provides ability to store Document and it's childs in a same location. At moment of class creation ODM will automatically resolve what instance has to be used to represent such data. We can demonsrate this functionality using following example:

```php
class Moderator extends User
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'moderates' => ['string']
    ];
}
```

Now we can create our Moderator and save it in a same collection as user (you can also redefine collection property):

```php
$user = new Moderator();
$user->name = 'Moderator';
$user->email = 'moderator@email.com';
$user->balance = 99;
$user->moderates = ['a', 'b', 'c'];

//Required
$user->profile = new Profile([
    'biography'   => 'Some biography.',
    'facebookUID' => 4532467890
]);

if (!$user->save()) {
    dump($user->getErrors());
}
```

While iteration thought our users and at one moment entity will be represented as `Moderator`:

```php
foreach (User::find() as $user) {
    dump($user);

    if ($user instanceof Moderator) {
        echo 'Found it!';
    }
}
```

> One of the side effects of multiple types stored in one collection - you have to verify class type supplied by `find`, `findOne` and `findByPK` methods, as calling this method in Moderator can and will return instance of User in some cases (i'm thinking about doing this check automatically in `findOne` and `findByPK` methods, but not in `find`).

You can also assign Document or DocumentEtity childs to compositions, for example:

```php
class SuperSession extends Session
{
    protected $schema = [
        'superKey' => 'string'
    ];
}
```

And, in your code:

```php
$user = User::findOne();
dump($user->sessions->all());

$user->sessions[] = new SuperSession([
    'timeCreated' => new \MongoDate(),
    'accessToken' => 'random',
    'superKey'    => 'some value'
]);

$user->save();
```

#### How Inheritance Work
Every time Compositor or Collection trying to create `Document` or `DocumentEntity` entity data passed to ODM component method `document`. Such method will analyze provided data, and, based on presented fields select specific class to be used as data representer. Such methodic automatically adds constrain to your code - no entity can be extended without adding unique fields into it's schema.

#### Logical Class Definition
In some cases you might want to control which class being selected for data array manually. To do that you have to switch inheritance logic for your entity to `DEFINITION_LOGICAL` - declare constant `DEFINITION`.

```php
class User extends Document
{
    use TimestampsTrait;

    /**
     * Indication to ODM component of method to resolve Document class using it's fieldset. This
     * constant is required due Document can inherit another Document.
     */
    const DEFINITION = self::DEFINITION_LOGICAL;

    ...
```

Now, every time when ODM will try to constuct instance of User it will previously try to find appropriate class by calling static method `defineClass` or User model, let's try to write such method in a way to emulate default inheritance logic:

```php
public static function defineClass(array $fields, ODM $odm)
{
    if (array_key_exists('moderates', $fields)) {
        return Moderator::class;
    }

    return self::class;
}
```

> You can obviosuly use more complex logic.
