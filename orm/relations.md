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

If you wish to change user profile, you can simply assign new model to it:

```php
$user = User::findByPK(1);
$user->profile = (new Profile())->setBiography('New biography.');
$user->save();
```

> Attention, previously assigned Profile will get "user_id" key set as `null`.

## Has Many Relation

## Belongs To Relation

#### Belongs To Morphed

## Many To Many

#### Many To Many Morphed

## Other Relations