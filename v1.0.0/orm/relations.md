# ORM Relations
The whole power of relational databases and ORMs comes when we are talking about relationships between tables and records. 

Spiral provide expendable mechanism to create Record relationship and scaffold required columns, indexes and foreign keys for such connections.

Note, examples in this section are given using [`SourceTrait`](/v1.0.0/orm/orm/repositories.md) for simplicity.

## Relation Definition
Relation definition does not require to create any additional structure or method, you are able to locate such relation directly in **record schema**.
 
In most of cases you can relay on relation itself to create all required columns, indexes and foreign keys, in such case they the only thing you have to do to link 2 tables together:

```php
'relation' => [self::HAS_ONE => OuterRecord::class]
```

Most of relations have a lot of additional parameters you can customize, such parameters can be included into relation definition as well:

```php
'relation' => [
    self::HAS_ONE          => OuterRecord::class,
    self::INNER_KEY        => 'my_key',
    OuterRecord::OUTER_KEY => 'outer_key'
]
```

> You can check list of available relation options below.

#### Relation Inversion
Important relation option which you might use a lot - INVERSE.
Such option used by ORM to automatically create relation in outer model. For example, if we have HAS_ONE relation, we can declare it's inversion - BELONGS_TO.

```php
'posts' => [
    self::HAS_MANY  => Post::class,
    
    Post::OUTER_KEY => 'author_id',
    Post::INVERSE   => 'author'
]
```

## Has One Relation
One of the most common and useful relation you might use - HAS_ONE. Such relation links two records together and inversed into BELONGS_TO. 

Example:

```php
class Profile extends Record
{
    const SCHEMA = [
        'id'        => 'primary',
        'biography' => 'text'
    ];
}
```

In your User Schema:

```php
const SCHEMA = [
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

Relation automatically created outer key "user_id" in our profiles table, declared index and foreign key for such column. 

Outer key name was calculated based on User model name. We can alter name of such column or link to existed column by adding addition relation options, let's list them all:

Option            | Default                             | Description
---               | ---                                 | ---
INNER_KEY         | {record:primaryKey}                 | Outer key will be based on parent record role and inner key name (**id**).
OUTER_KEY         | {record:role}_{definition:innerKey} | Outer key will be based on parent record role and inner key name (**user**_**id**).
CONSTRAINT        | true                                | Set constraints (foreign keys) by default.
CONSTRAINT_ACTION | CASCADE                             | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                | Relation allowed to create indexes in outer table.
NULLABLE          | false                               | Has one counted as not nullable by default .

To create inversion:

```php
'profile' => [
    self::HAS_ONE => Profile::class,
    self::INVERSE => 'user'
]
```

Since relation is not nullable ORM will create child entity on demand, use `$user->profile` to access such entity:

```php
$user = User::findByPK(1);
$user->profile->biography = 'This is some biography';
$user->save();
```

You can always assign new Profile instance and remove old data:

```php
$user = User::findByPK(1);
$user->profile = (new Profile())->setBiography('New biography.');
$user->save();
```

## Has Many Relation
HasMany relation is similar to HasOne but represent set of entities:

```php
class Post extends Record 
{
    const SCHEMA = [
        'id'        => 'primary',
        'published' => 'bool',
        'title'     => 'string',
        'content'   => 'text'
    ];
}
```

In User model:

```php
const SCHEMA = [
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

Additional relation options:

Option            | Default                             | Description
---               | ---                                 | ---
INNER_KEY         | {record:primaryKey}                 | Outer key will be based on parent record role and inner key name (**id**).
OUTER_KEY         | {record:role}_{definition:innerKey} | Outer key will be based on parent record role and inner key name (**user**_**id**).
CONSTRAINT        | true                                | Set constraints (foreign keys) by default.
CONSTRAINT_ACTION | CASCADE                             | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                | Relation allowed to create indexes in outer table.
NULLABLE          | true                                | Has many counted as nullable by default .
**WHERE**         | []                                  | HasMany allow us to define default WHERE statement for relation in a simplified array form.
ORDER_BY          | []                                  | Default sort direction.

We can specify select conditions and sort order:

```php
'publishedPosts'  => [
    self::HAS_MANY  => Post::class,
    Post::OUTER_KEY => 'author_id'
    Post::WHERE     => [
        '{@}.published' => true
    ]
    Post::ORDER_BY  => [
        '{@}.id' => 'ASC'
    ]
]
```

> Declaring posts columns using `{@}` prefix, such prefix will be automatically replaced with valid posts table alias based on context. You must always include it into your code.

To add post into list use `add` method of User or inversed relation:

```php
protected function indexAction()
{
    $user = User::findByPK(1);

    $faker = Factory::create();

    $post = new Post();
    $post->author = $user;
    
    $post->title = $faker->text(50);
    $post->content = $faker->text;
    $post->published = $faker->boolean();
    $post->save();
}
```

From user:

```php
protected function indexAction()
{
    $user = User::findByPK(1);

    $faker = Factory::create();

    $post = new Post();
   
    $post->title = $faker->text(50);
    $post->content = $faker->text;
    $post->published = $faker->boolean();

    $user->posts->add($post);
    $user->save();
}
```

To iterate over associated posts:

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
```

## Belongs To Relation
Another powerful relation you might need to use is BELONGS_TO. Such relation is opposite to HAS_ONE and HAS_MANY by it's idea. Since in most of cases we would need both HAS_* and BELONGS_TO relations, it's the best to declare it as inversed.

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

## Many To Many
MANY_TO_MANY provides you ability to link many records from different tables using pivot table. Such relation can automatically scaffold needed pivot table with or without custom user columns. In addition, you are able to set WHERE and WHERE_PIVOT conditions to customize your relations.

To define MANY_TO_MANY relation, first of all let's create new model `Role` using CLI command "spiral create:record role -f id:primary -f name:string":

```php
class Role extends Record
{
    const SCHEMA = [
        'id'        => 'primary',
        'name'      => 'string'
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

Such table will be used to link our User and Role models together. To link models call method `link()` of ManyToMany relation in User Record:

```php
$user = User::findByPK(1);

$user->roles->link(new Role([...]));
$user->roles->link(Role::findByPK(2));

$user->save();
```

To overwrite all associated models:

```php
$user->roles = [
    $role1, $role2
];

$user->save();
```

#### Check Association
To check if user have associated model:

```php
dump($user->roles->has($role1));
dump($user->roles->has(Role::findByPK(1));
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

Once schema is synchronized our pivot table will look like that:

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

Set pivot columns using second argument of `link` method:

```php
$user = User::findByPK(1);

$user->roles->link(Role::findByPK(1), [
    'time_assigned' => new \DateTime(),
    'status'        => 'active'
]);

$user->save();
```

#### Get Pivot Data
To get associated pivot data:

```php
$user->roles->getPivot($role);
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
