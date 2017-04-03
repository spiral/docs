# ORM Relations
The whole power of relational databases and ORMs comes when we are talking about relationships between tables and records. Spiral provide extendable mechanism to create Record relationship and even scaffold required columns, indexes and foreign keys for such connections.

At this moment spiral support following relationships: has one, has many, belongs to parent, many to many, belongs to morphed parent, many to many morphed. Let's try to walk thought every relation, it's definition and abilities.

> Make sure you already read [ORM Basics](basics.md).

## Relation Definition
Relation defintion does not require to create any additional structure or method, you are able to locate such relation directly in **record schema**. In most of cases you can relay on relation itself to create all required columns, indexes and foreign keys, in such case they the only thing you have to do to link 2 tables together - declare relation type and outer model.

```php
'relation' => [self::HAS_ONE => OuterRecord::class]
```

Most of relations have a lot of additional parameters you can customize, for example you can decide if you want to create foreign key or select different inner/outer key (rather then letting relation to use primary key), to specify additional realtion parameters simply include them into it's definition:

```php
'relation' => [
    self::HAS_ONE          => OuterRecord::class,
    self::INNER_KEY        => 'my_key',
    OuterRecord::OUTER_KEY => 'outer_key'
]
```

You can check list of available relation options below.

#### Relation Inversion
One very importan relation option which you might use a lot - INVERSE. Such option used by Spiral ORM to automatically create relation in outer model. For example, if we have HAS_ONE relation, we can declare it's inversion - BELONGS_TO.

```php
'posts' => [
    self::HAS_MANY  => Post::class,
    Post::OUTER_KEY => 'author_id',
    self::INVERSE   => 'author'
]
```

## Has One Relation
One of the most common and useful relation you might use - HAS_ONE. Such relation links two records together and inversed into BELONGS_TO. Classical example - User has one Profile. To demonstrate such relation and it's behaviour let's create new Record - Profile using scaffolding: 'create:record profile -f id:primary -f biography:text'

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

Once done, you can get access to such relation using `__get` method of User model, relation will automatically construct instance of Profile if it does not exists. In addition you can notice that HAS_ONE is stated as EMBEDDED_RELATION by default, this means that such relation will validated and saved with it's parent (if it was previously loaded), as result we can do that:

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

> Attention, you have to deassociate previous Record before assigning new one!

Since Profile stated as embedded model, it will be also be validated with parent User, we can demonstrate that using code:

```php
$user = User::findByPK(1);
$user->profile->biography = '';
$user->save();

dump($user->getErrors());
```

Please note that you can access relation using magic property, such property will cache `Profile` model, meaning it will create database query one of first call. If you wish to talk to relation class closely or even create custom query (which does not have too much sense for HAS_ONE) you can access such relation using `relation()` method of Record model or magic method:

```php
dump($user->relation('profile'));

//Relation mocks Selector instance
dump($user->profile()->count());
dump($user->profile()->findOne());
```

> Accessing relation data using magic property is very useful in combination with [eager loading] (loading.md).
> To remember, calling relation using **property** will only give you access to pre-loaded and cached realtion data, calling relation using **method** will give you access to relation query, pivot tables and etc. Basically you have to READ using property and WRITE or use CUSTOM SELECT QUERIES using method way.

## Has Many Relation
Has Many relation is very similar to has one, however it is not embedded to it's parent by default and it provides us ability to specify set of where conditions to link two tables together. Before we will start, let's create new Record - Post: "create:record post -f id:primary -f published:bool -f title:string -f content:text"

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

The easies way to assign post to user will be using in inversed BELONGS_TO relation (for example author), however, for now, let's use `create` method of relation (attention, WHERE condition will NOT populate post model): 

```php
protected function indexAction()
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

BELONGS_TO relation can be inversed, however, since ORM does not know if relation must be inversed into HAS_ONE or HAS_MANY, you have to clearly state that using provided form:

```php
'author'    => [
    self::BELONGS_TO => User::class,
    User::INVERSE    => [User::HAS_MANY, 'posts']
]
```

Once you have such relation declared in your model, you can use it to assign or de-assign parent to your record:

```php
protected function indexAction()
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
protected function indexAction()
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

## Many To Many
One of the most powerful and complex ORM relations MANY_TO_MANY provides you ability to link many records from different tables using pivot table. Such relation can automatically scaffold needed pivot table with or withour custom user columns. In addition, you are able to set WHERE and WHERE_PIVOT conditions to customize your relations.

To define MANY_TO_MANY relation, first of all let's create new model `Role` using CLI command "spiral create:record role -f id:primary -f name:string":

```php
class Role extends Record
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
        'name'      => 'string'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'name' => [
            'notEmpty'
        ]
    ];
}
```

We can create few roles right in our controller to have some data to work with:

```php
$role = new Role();
$role->name = 'admin';
$role->save();

$role = new Role();
$role->name = 'moderator';
$role->save();
```

Now we are able to define our MANY_TO_MANY relation, as before it can be done by a simple array in User model:

```php
'roles' => [self::MANY_TO_MANY => Role::class]
```

Once schema executed we will get 2 new tables - roles and pivot table "role_user_map", let's check it structure:

```
Columns of primary.role_user_map:
+---------+----------------+----------------+-----------+----------------+
| Column: | Database Type: | Abstract Type: | PHP Type: | Default Value: |
+---------+----------------+----------------+-----------+----------------+
| role_id | integer        | integer        | int       | ---            |
| user_id | integer        | integer        | int       | ---            |
+---------+----------------+----------------+-----------+----------------+

Indexes of primary.role_user_map:
+-----------------------------------------------------------+--------------+------------------+
| Name:                                                     | Type:        | Columns:         |
+-----------------------------------------------------------+--------------+------------------+
| primary_role_user_map_index_user_id_role_id_55f99aaf99ecd | UNIQUE INDEX | user_id, role_id |
+-----------------------------------------------------------+--------------+------------------+

Foreign keys of primary.role_user_map:
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
| Name:                                               | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
| primary_role_user_map_foreign_user_id_55f99aaf99704 | user_id | primary_users  | id              | CASCADE    | CASCADE    |
| primary_role_user_map_foreign_role_id_55f99aaf99710 | role_id | primary_roles  | id              | CASCADE    | CASCADE    |
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
```

Such table will be used to link our User and Role models together. To link models simply call method `link()` of ManyToMany relation in User Record. We are able to provide `Role` model or `Role` id as input into such method:

```php
$user = User::findByPK(1);

$user->roles()->link(1);
$user->roles()->link(Role::findByPK(2));
```

We can also provide array of ids or models as argument:

```php
$user->roles()->link([1, 2]);
$user->roles()->link([Role::findByPK(2), Role::findByPK(1)]);
```

> Attention, you have to call `link()` method using **relation method** not property as it affets pivot table, not pre-loaded/cached relation data.

In addition to `link()` you are able to use similar method `sync()`, the only difference that `sync()` will remove every connection which was not specified in it's arguments:

```php
//Connection with Role::2 will be removed
$user->roles()->sync(1);

//Link only with Role::1 and Role::2
$user->roles()->sync([Role::findByPK(2), Role::findByPK(1)]);
```

> Attention, calling method `link()` or `sync()` **will not** affect already loaded relation data!

To unlink records you can user methods `unlink` or `unlinkAll`.

#### Check Link Status
To check if two models already link together use method `has`, method can accept both model itself, model outer key or array or models/keys.

```php
dump($user->roles()->has(1));
dump($user->roles()->has([1, 2]));
dump($user->roles()->has(Role::findByPK(1));
```

Such method will run query againts pivot_table (with WHERE_PIVOT conditions if specified). You can use similar method via property based access, in this case no additional queries will be created (except loading relation data):

```php
dump($user->roles->has(1));
dump($user->roles->has([1, 2]));
dump($user->roles->has(Role::findByPK(1));
```

> Please note, `has` method **will ignore** conditions specified in WRERE option.

#### Fetching Linked Models
After linking our models together we can fetch them using either propery (pre-loaded/cached) or method way:

```php
$user = User::findByPK(1);

//Cached
foreach ($user->roles as $role) {
    dump($role);
}

//New query every time
foreach ($user->roles()->find(['name' => 'admin']) as $role) {
    dump($role);
}
```

#### ManyToMany Options
MANY_TO_MANY relation defined a lot of options we can use, let's check them all:

Option | Default | Description
--- | --- | ---
INNER_KEY         | {record:primaryKey}                     | Inner key of parent record will be used to fill "THOUGHT_INNER_KEY" in pivot table. (**role.id**)
OUTER_KEY         | {outer:primaryKey}                      | We are going to use primary key of outer table to fill "THOUGHT_OUTER_KEY" in pivot table. This is technically "inner" key of outer record, we will name it "outer key" for simplicity. (**user.id**)
THOUGHT_INNER_KEY | {record:role}_{definition:innerKey}     | Name of field where parent record inner key will be stored in pivot table, role + innerKey by default. (**role_id**).
THOUGHT_OUTER_KEY | {outer:role}_{definition:outerKey}      | Name field where inner key of outer record (outer key) will be stored in pivot table, role + outerKey by default (**user_id**).
CONSTRAINT        | true                                    | Relations will set constraints in pivot table (foreign keys).
CONSTRAINT_ACTION | CASCADE                                 | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                    | Relation is allowed to create indexes in pivot table
PIVOT_TABLE       | null                                    | Name of pivot table to be declared, default value is not stated as it will be generated based on roles of inner and outer records (**role_user_map**).
CREATE_PIVOT      | true                                    | Relation allowed to create pivot table.
PIVOT_COLUMNS     | []                                      | Additional set of columns to be added into pivot table, you can use same column definition type as you using for your records.
PIVOT_DEFAULTS    | []                                      | Set of default values to be used for pivot table columns.
WHERE_PIVOT       | []                                      | Where statement in a form of simplified array definition to be applied to pivot table data.
WHERE             | []                                      | Where statement to be applied for data in outer data while loading relation data can not be inversed. Attention, WHERE conditions not used in has(), link() and sync() methods.

#### Pivot Table Columns
As you might notice from ManyToMany options, you are able to specify set of columns and their default values to be added into pivot table. Let's try to declare two columns for our purposes:

```php
'roles' => [
    self::MANY_TO_MANY   => Role::class,
    self::PIVOT_COLUMNS  => [
        'time_assigned' => 'datetime',
        'status'        => 'enum(active,disabled)'
    ],
    self::PIVOT_DEFAULTS => [
        'status' => 'active'
    ]
]
```

> I declared such options in User model, however it will be the best to move relations like that into Role model instead and inverse it.

Once schema is syncronized our pivot table will look like that:

```
Columns of primary.role_user_map:
+---------------+-----------------------------+----------------+-----------+---------------------+
| Column:       | Database Type:              | Abstract Type: | PHP Type: | Default Value:      |
+---------------+-----------------------------+----------------+-----------+---------------------+
| role_id       | integer                     | integer        | int       | ---                 |
| user_id       | integer                     | integer        | int       | ---                 |
| time_assigned | timestamp without time zone | timestamp      | string    | 1970-01-01 00:00:00 |
| status        | character (8)               | enum           | string    | active              |
+---------------+-----------------------------+----------------+-----------+---------------------+

Indexes of primary.role_user_map:
+-----------------------------------------------------------+--------------+------------------+
| Name:                                                     | Type:        | Columns:         |
+-----------------------------------------------------------+--------------+------------------+
| primary_role_user_map_index_user_id_role_id_55f99aaf99ecd | UNIQUE INDEX | user_id, role_id |
+-----------------------------------------------------------+--------------+------------------+

Foreign keys of primary.role_user_map:
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
| Name:                                               | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
| primary_role_user_map_foreign_user_id_55f99aaf99704 | user_id | primary_users  | id              | CASCADE    | CASCADE    |
| primary_role_user_map_foreign_role_id_55f99aaf99710 | role_id | primary_roles  | id              | CASCADE    | CASCADE    |
+-----------------------------------------------------+---------+----------------+-----------------+------------+------------+
```

Now, we can use additional arguments and input formats in `link` and `sync` to specify set of pivot table data:

```php
$user = User::findByPK(1);

$user->roles()->sync(Role::findByPK(1), [
    'time_assigned' => new \DateTime(),
    'status'        => 'active'
]);
```

We can also use shorter version when pivot columns are associated with outer model id:

```php
$user->roles()->sync([
    1 => [
        'time_assigned' => new \DateTime(),
        'status'        => 'active'
    ]
]);
```

Since we have our pivot columns populated we can get access to them in our queries (check [eager loading] (loading.md)) or from our code using method `pivotData()`:

```php
$user = User::findByPK(1);

foreach ($user->roles as $role) {
    dump($role->getPivot()['status']);
    dump($role);
}
```

#### Where and Where Pivot Conditions
Since we are able to specify pivot columns, we can also use WHERE_PIVOT conditions:

```php
'roles'           => [
    self::MANY_TO_MANY   => Role::class,
    self::PIVOT_COLUMNS  => [
        'time_assigned' => 'datetime',
        'status'        => 'enum(active,disabled)'
    ],
    self::PIVOT_DEFAULTS => [
        'status' => 'active'
    ],
    self::WHERE_PIVOT    => [
        '{@}.status' => 'active'
    ]
]
```

> Again, notice that we are using {@} as table alias.

You can also specify WHERE conditions same way as for HAS_MANY, however such option will only be used for **selection**, `has` method will ignore it (use `$user->tags->has()` instead of `$user->tags()->has()`).

> You can still use `has` method of relation data (using property) as in this case check will me performed using already selected data.