# Compositions, Aggregations, Inheritance
The ODM component does not use termin "relation(s)" to describe it's functionality but switches to the OOP definitions for [Composition, Aggregration (Reference)] (https://en.wikipedia.org/wiki/Object_composition) and Inheritance. 

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

## Inheritance
In addition to aggregation and composition, the ODM engine provides the ability to store Document and it's children in the same location. When class is created, ODM will automatically resolve what instance has to be used to represent such data. We can demonsrate this functionality using the following example:

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

Now we can create our Moderator and save it in the same collection as User (you can also re-define the collection property):

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

Iteration through our users and entity will be represented as `Moderator`:

```php
foreach (User::find() as $user) {
    dump($user);

    if ($user instanceof Moderator) {
        echo 'Found it!';
    }
}
```

> One of the side effects of using multiple types stored in one collection is that you have to verify the class type supplied by `find`, `findOne` and `findByPK` methods. Calling this method in Moderator can and will return an instance of User in some cases (This could be checked automatically in `findOne` and `findByPK` methods, but not in `find`).

You can also assign Document or DocumentEtity children to compositions, for example:

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
Every time Compositor or Collection tries to create `Document` or `DocumentEntity`, entity data passed to ODM component method `document`. This method will analyze the provided data and based on the shown fields, select the specific class to be used as a data representer. This method automatically adds constraints to your code. No entity can be extended without adding unique fields into it's schema.

#### Logical Class Definition
In some cases you might want to manually control which class is selected for the data array. To do this, you have to switch inheritance logic for your entity to `DEFINITION_LOGICAL` and declare constant `DEFINITION`.

```php
class User extends Document
{
    use TimestampsTrait;

    /**
     * Indication to ODM component of method to resolve Document class using it's fieldset. This
     * constant is required since Document can inherit another Document.
     */
    const DEFINITION = self::DEFINITION_LOGICAL;

    ...
```

Now, every time ODM will constucts an instance of User, it will try to find the appropriate class by calling the static method `defineClass` or the User model. Let's try to write this method in a way to emulate the default inheritance logic:

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
