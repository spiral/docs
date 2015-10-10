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
