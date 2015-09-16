# ORM Relations
The whole power of relational databases and ORMs comes when we are talking about relationships between tables and records. Spiral provide extendable mechanism to create Record relationship and even scaffold required columns, indexes and foreign keys for such connections.

At this moment spiral support following relationships: has one, has many, belongs to parent, many to many, belongs to morphed parent, many to many morphed. Let's try to walk thought every relation, it's definition and abilities.

## Relation Definition
Relation defintion does not require to create any additional structure of method, you are able to locate such relation directly in record schema. In most of cases you can relay on relation itself to create all required columns, indexes and foreign keys, in such case they the only thing you have to do to link 2 tables together - declare relation type and outer model.

```php
'relation' => [self::HAS_ONE => OuterRecord::class]
```

Most of relations have a lot of additional parameters you can customize, for example you can decide if you want to create foreign key or select different inner/outer key (rather then letting relation to do that), to specify additional realtion parameters simply include them into it's definition:

```php
'relation' => [
    self::HAS_ONE          => OuterRecord::class,
    self::INNER_KEY        => 'my_key',
    OuterRecord::OUTER_KEY => 'outer_key'
]
```

You can check list of available relation options below.

#### Relation Inversion
One very importan relation option which you might use a lot - INVERSION. Such option will be used by Spiral ORM to automatically create relation in related model. For example, when we have HAS_ONE relation, we can declate it's inversion - BELONGS_TO.

```php
'posts' => [
    self::HAS_MANY  => Post::class,
    Post::OUTER_KEY => 'author_id',
    self::INVERSE   => 'author'
]
```

## Has One Relation
One of the most common and useful relation you might use - HAS_ONE. Such relation links to records together and inversed into BELONGS_TO. Classical example - User has one Profile. To demonstrate such relation and it's behaviour let's create new Record - Profile using scaffolding: 'create:record profile -f id:primary -f biography:text'

```php
class Profile extends Record
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
        'id'        => 'primary',
        'biography' => 'text'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'biography' => [
            'notEmpty'
        ]
    ];
}
```

Now, we can link our User model to profile by declaring new value in our User schema:

```php
 protected $schema = [
    'id'              => 'primary',
    'time_registered' => 'datetime',
    'name'            => 'string(63)',
    'email'           => 'string',
    'status'          => 'enum(active,blocked)',
    'balance'         => 'decimal(11,2)',
    'profile'         => [self::HAS_ONE => Profile::class]
];
```

Let's try to update our ORM schema and check resulted "profiles" table:

```
Columns of primary.profiles:
+-----------+----------------+----------------+-----------+----------------------------------------------+
| Column:   | Database Type: | Abstract Type: | PHP Type: | Default Value:                               |
+-----------+----------------+----------------+-----------+----------------------------------------------+
| id        | serial         | primary        | int       | nextval('primary_profiles_id_seq'::regclass) |
| biography | text           | text           | string    | ---                                          |
| user_id   | integer        | integer        | int       | ---                                          |
+-----------+----------------+----------------+-----------+----------------------------------------------+

Indexes of primary.profiles:
+----------------------------------------------+-------+----------+
| Name:                                        | Type: | Columns: |
+----------------------------------------------+-------+----------+
| primary_profiles_index_user_id_55f96fa0c408f | INDEX | user_id  |
+----------------------------------------------+-------+----------+

Foreign keys of primary.profiles:
+------------------------------------------------+---------+----------------+-----------------+------------+------------+
| Name:                                          | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+------------------------------------------------+---------+----------------+-----------------+------------+------------+
| primary_profiles_foreign_user_id_55f96fa0b1336 | user_id | primary_users  | id              | CASCADE    | CASCADE    |
+------------------------------------------------+---------+----------------+-----------------+------------+------------+
```

As you can see, relation automatilly created outer key "user_id" in our profiles table, declared index and foreign key for such column. Outer key name was automatically calculated based on User model name. We can alter name of such column or link to existed column by adding addition relation options, let's list them all:

Option            | Default                             | Description
---               | ---                                 | ---
INNER_KEY         | {record:primaryKey}                 | Outer key will be based on parent record role and inner key name (**id**).
OUTER_KEY         | {record:role}_{definition:innerKey} | Outer key will be based on parent record role and inner key name (**user**_**id**).
CONSTRAINT        | true                                | Set constraints (foreign keys) by default.
CONSTRAINT_ACTION | CASCADE                             | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                | Relation allowed to create indexes in outer table.
NULLABLE          | false                               | Has one counted as not nullable by default .
EMBEDDED_RELATION | true                                | Embedded relations are validated and saved with parent model and can accept values using setFields.

You can redefine any of this option. In addition you can declare relation inversion to create relation in Profile model:

```php
'profile' => [
    self::HAS_ONE => Profile::class,
    self::INVERSE => 'user'
]
```

Once done, you can get access to such relation using __get method of User model, relation will automatically construct instance of Profile if it does not exists. In addition you can notice that HAS_ONE is stated as EMBEDDED_RELATION by default, this means that when such relation is loaded Record will validate and save with it's parent, as result we can do that:

```php
$user = User::findByPK(1);
$user->profile->biography = 'This is some biography';
$user->save();
```

If you wish to change user profile entitelly, you can simply assign new model to it:

```php
$user = User::findByPK(1);
$user->profile = (new Profile())->setBiography('New biography.');
$user->save();
```

> Attention, previously assigned Profile "user_id" key will be set as `null`.

Since Profile stated as embedded model, it will be saved and validated with it's parent User, we can demonstrate then using code:

```php
$user = User::findByPK(1);
$user->profile->biography = '';
$user->save();

dump($user->getErrors());
```

Please note that you can access relation using magic property, such property will cache Profile model, meaning it will create database query one of first call. If you wish to talk to relation class closely or even create custom query (which does not have too much sense for HAS_ONE) you can access such relation using `relation()` method of Record model or magic method:

```php
dump($user->relation('profile'));

//Relation mocks Selector instance
dump($user->profile()->count());
dump($user->profile()->findOne());
```

> Accessing relation data using magic property is very useful in combination with [eager loading] (loading.md).

## Has Many Relation
Has Many relation is very similar to has one, however it is not embedded to it's parent by default and it provides us ability to specify set of where conditions to link two tables together. Before we will start, let's create new Record Post: "create:record post -f id:primary -f published:bool -f title:string -f content:text"

```php
class Post extends Record 
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
        'id'        => 'primary',
        'published' => 'bool',
        'title'     => 'string',
        'content'   => 'text'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'title'     => [
            'notEmpty'
        ],
        'content'   => [
            'notEmpty'
        ]
    ];
}
```

> Do not forget to remove validation for `published` field.

To state that user has many posts, we can declare HAS_MANY relation in User model:

```php
protected $schema = [
    'id'              => 'primary',
    'time_registered' => 'datetime',
    'name'            => 'string(63)',
    'email'           => 'string',
    'status'          => 'enum(active,blocked)',
    'balance'         => 'decimal(11,2)',
    'profile'         => [self::HAS_ONE => Profile::class],
    'posts'           => [
        self::HAS_MANY => Post::class,
        Post::OUTER_KEY => 'author_id'
    ]
];
```

After schema update, our post table will look pretty similar to profiles one, so we may skip checking it. As in case with HAS_ONE there is many additional options you might check and define:

Option            | Default                             | Description
---               | ---                                 | ---
INNER_KEY         | {record:primaryKey}                 | Outer key will be based on parent record role and inner key name (**id**).
OUTER_KEY         | {record:role}_{definition:innerKey} | Outer key will be based on parent record role and inner key name (**user**_**id**).
CONSTRAINT        | true                                | Set constraints (foreign keys) by default.
CONSTRAINT_ACTION | CASCADE                             | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                | Relation allowed to create indexes in outer table.
NULLABLE          | true                                | Has many counted as nullable by default .
**WHERE**         | []                                  | HasMany allow us to define default WHERE statement for relation in a simplified array form.

Compared to HAS_ONE relation we have only one new option we can use WHERE, such option provides us alibity to specify loading conditions for our records, let's try to use it.

```php
'publishedPosts'  => [
    self::HAS_MANY => Post::class,
    Post::OUTER_KEY => 'author_id'
    Post::WHERE    => [
        '{@}.published' => true
    ]
]
```

> Please note that we declaring posts column name using `{@}` prefix, such prefix will be automatically replaced with valid posts table alias based on contex. You must always include it into your code.

The easies way to assign post to user will be using in inversed BELONGS_TO relation (for example author), however for now let's use create method of relation (attention, WHERE condition will NOT populate post model): 

```php
public function index()
{
    $faker = Factory::create();

    $user = User::findByPK(1);

    $post = $user->posts()->create();
    $post->title = $faker->text(50);
    $post->content = $faker->text;
    $post->published = $faker->boolean();
    $post->save();
}
```

Now we are able to use such relation in our code, again we can either use simplfied property access or relation method with additional conditions:

```php
$user = User::findByPK(1);
dump($users->posts->count());

//Cached
foreach ($user->posts as $post) {
    dump($post->title);
}

//Cached
foreach ($user->publishedPosts as $post) {
    dump($post->title);
}

foreach ($user->posts()->find()->where('post.published', false) as $post) {
    dump($post);
}
```

> Attention, removing or adding new Post model associated with user WILL NOT affect cached relation data.

## Belongs To Relation
Another powerful relation you might need to use is BELONGS_TO. Such realtion is opposite to HAS_ONE and HAS_MANY by it's idea. Since in most of cases we would need both HAS_* and BELONGS_TO relations, it's the best to declare it as inversed.

```php
 'posts' => [
    self::HAS_MANY  => Post::class,
    Post::OUTER_KEY => 'author_id',
    Post::INVERSE   => 'author'
],
```

> Relation does not support WHERE conditions.

You can always declare such relation in your outer model directly (Post):

```php
'author'    => [self::BELONGS_TO => User::class]
```

Let's try to check relation options:

Option            | Default                               | Description
---               | ---                                   | ---
INNER_KEY         | {outer:primaryKey}                    | Outer key is primary key of related record by default (user.**id**).
OUTER_KEY         | {name:singular}_{definition:outerKey} | Inner key will be based on singular name of relation and outer key name (**author**_**id**).
CONSTRAINT        | true                                  | Set constraints (foreign keys) by default.
CONSTRAINT_ACTION | CASCADE                               | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                  | Relation allowed to create indexes in outer table.
NULLABLE          | true                                  | Nullable by default.

BELONGS_TO relation can be inversed, however since ORM does not know if such relation must be inversed into HAS_ONE or HAS_MANY you have to clearly state that:

```php
'author'    => [
    self::BELONGS_TO => User::class,
    User::INVERSE    => [User::HAS_MANY, 'posts']
]
```

> You don't need to do it in our case.

Once you have such relation declared in your model, you can use to assign or deassign parent to your record:

```php
public function index()
{
    $faker = Factory::create();

    $user = User::findByPK(1);

    $post = new Post();
    $post->title = $faker->text(50);
    $post->content = $faker->text;

    //Assigning parent user
    $post->author = $user;
    $post->save();

    dump($post->author);

    //De-assigning
    $post->author = null;
    $post->save();
}
```

> You can only de-assign parent if relation is nullable.

#### Entity Cache
BELONGS_TO relation has one interesting feature, available when you are linking parent record based on it primary key (default behaviour). In this case relation will try to locate parent model using [entity cache] (loading.md) first (if it's enabled) as result you can avoid additonal queries, let's try to demonstrate that:

```php
public function index()
{
    $post = Post::findOne();

    //Will produce SQL query
    dump($post->author);

    foreach (Post::find() as $post) {
        //Since we have only one user no query will be produced here
        dump($post->author);
    }
}
```

Even more, entity cache will share instance of User between multiple posts, meaning:

```php
$postA = Post::findOne(['author_id' => 1]);
$postB = Post::findOne(['author_id' => 1, 'id' => ['!=' => $postA->id]]);

dumP($postA->author === $postB->author);
```

> Such ability can be very useful if you want to overwrite `save()` or `delete()` functions of your entities and touch/update parent inside them.

#### Belongs To Morphed
Spiral ORM provides another relation type very similar to BELONGS_TO - BELONGS_TO_MORPHED. Such relation gives ability to assign model to various parents based on morph key value. Morphed relations can be declared exacly same wasy a BELONGS_TO, however you must link your relation to an **interface** rather than specific model.

> Disclaimer: polymorphic relations must be used only when you absolutelly sure about it, avoid using BELONGS_TO_MORPHED as much as you can. Fyi, you are not able to pre-load morphed relations.

Let's try to create new ORM entity Photo which we would like to assign to User or Post models: "create:record photo -f id:primary -f imageURL:string"

```php
class Photo extends Record 
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
        'id'       => 'primary',
        'imageURL' => 'string'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'imageURL' => [
            'notEmpty'
        ]
    ];
}
```

Before declaring our relation we would have to create an internal to link our model to:

```php
interface PhotoHolderInterface
{

}
```

Now we can declare our relation in Photo model, since we want both Post and User get inverted relations we will declare INVERSE option.

```php
protected $schema = [
    'id'       => 'primary',
    'imageURL' => 'string',
    'parent'   => [
        self::BELONGS_TO => PhotoHolderInterface::class,
        self::INVERSE    => [self::HAS_ONE, 'photo']
    ]
];
```

The only thing we have to do to let our User and Post model have photo - implement `PhotoHolderInterface`, inverse relation in this case will be cretaed automatically.

```php
class User extends Record implements PhotoHolderInterface
{
    ...
```

Now we can run schema update and check our "photos" table:

```
Columns of primary.photos:
+-------------+-------------------------+----------------+-----------+--------------------------------------------+
| Column:     | Database Type:          | Abstract Type: | PHP Type: | Default Value:                             |
+-------------+-------------------------+----------------+-----------+--------------------------------------------+
| id          | serial                  | primary        | int       | nextval('primary_photos_id_seq'::regclass) |
| imageURL    | character varying (255) | string         | string    | ---                                        |
| parent_type | character varying (32)  | string         | string    | ---                                        |
| parent_id   | integer                 | integer        | int       | ---                                        |
+-------------+-------------------------+----------------+-----------+--------------------------------------------+

Indexes of primary.photos:
+----------------------------------------------------------+-------+------------------------+
| Name:                                                    | Type: | Columns:               |
+----------------------------------------------------------+-------+------------------------+
| primary_photos_index_parent_type_parent_id_55f98f146672f | INDEX | parent_type, parent_id |
+----------------------------------------------------------+-------+------------------------+
```

As you can see relation created compound index using inner and morph keys "parent_id" and "parent_type". Both keys can be configured using relation options:


Option            | Default                               | Description
---               | ---                                   | ---
OUTER_KEY         | {outer:primaryKey}                    | By default, we are looking for primary key in our outer records, outer key must present in every outer record and be consistent (**id**).
INNER_KEY         | {name:singular}_{definition:outerKey} | Inner key name will be created based on singular relation name and outer key namet (`parent_id`).
MORPH_KEY         | {name:singular}_type                  | Morph key created based on singular relation name and postfix _type (`parent_type`).
CREATE_INDEXES    | true                                  | Relation allowed to create indexes in outer table.
NULLABLE          | true                                  | Nullable by default.

> No foreign keys are created for BELONGS_TO_MORPHED relation.

Now we can operate with our model same way as in case with BELONG_TO relation:

```php
$photo = new Photo();
$photo->imageURL = 'some-url';
$photo->parent = User::findOne();
$photo->save();

//We can change parent to post at any moment
$post = Post::findOne();

$photo->parent = $post;
$photo->save();

dump($post->photo);
```

> Simply declare HAS_MANY inversion to link multiple photos to morphed parent.

## Many To Many

#### Many To Many Morphed

## Other Relations
