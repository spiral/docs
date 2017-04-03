
#### Belongs To Morphed
Spiral ORM provides another relation type very similar to BELONGS_TO - BELONGS_TO_MORPHED. This relation gives ability to assign model to various parents based on morph key value. Morphed relations can be declared exacly same wasy a BELONGS_TO, however you must link your relation to an **interface** rather than specific model.

> Disclaimer: polymorphic relations must be used only when you absolutelly sure about it, avoid using BELONGS_TO_MORPHED as much as you can. 
> Fyi, you are not able to pre-load morphed relations.

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

Before declaring our relation we would have to create an interface to link our model to:

```php
interface PhotoHolderInterface
{

}
```

Now we can describe our relation in Photo model, since we want both Post and User get inverted relations we will declare INVERSE option.

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

WE can operate with our model same way as in case with BELONG_TO relation:

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



#### Many To Many Morphed
As in case with BELONGS_TO you are able to define polymorphic many to many connection. To define such connection we can simply link our relation to an **interface**. Let's try to do an example using model Tag. Again, to create our Tag model - "create:record tag -f id:primary -f name:string":

```php
class Tag extends Record
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
        'id'   => 'primary',
        'name' => 'string'
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

Now, we are going to link tag with `TaggableInterface` (empty) and implement such interface in User and Post models:

```php
protected $schema = [
    'id'     => 'primary',
    'name'   => 'string',
    'tagged' => [
        self::MANY_TO_MANY => TaggableInterface::class,
        self::INVERSE      => 'tags'
    ]
];
```

As before, the best way to create relation from `User` and `Post` to Tags - use INVERSE option. After updating ORM schema let's check pivot table, this time our table follow relation name (not as in previous case "role_user_map") so it named "tagged_map":

```
Columns of primary.tagged_map:
+-------------+------------------------+----------------+-----------+----------------+
| Column:     | Database Type:         | Abstract Type: | PHP Type: | Default Value: |
+-------------+------------------------+----------------+-----------+----------------+
| tag_id      | integer                | integer        | int       | ---            |
| tagged_type | character varying (32) | string         | string    | ---            |
| tagged_id   | integer                | integer        | int       | ---            |
+-------------+------------------------+----------------+-----------+----------------+

Indexes of primary.tagged_map:
+-----------------------------------------------+--------------+--------------------------------+
| Name:                                         | Type:        | Columns:                       |
+-----------------------------------------------+--------------+--------------------------------+
| primary_tagged_map_index_tag_id_55f9c032ee81f | INDEX        | tag_id                         |
| 6ec6a75196a6baf6b0c43f00b500c35b              | UNIQUE INDEX | tag_id, tagged_type, tagged_id |
+-----------------------------------------------+--------------+--------------------------------+

Foreign keys of primary.tagged_map:
+-------------------------------------------------+---------+----------------+-----------------+------------+------------+
| Name:                                           | Column: | Foreign Table: | Foreign Column: | On Delete: | On Update: |
+-------------------------------------------------+---------+----------------+-----------------+------------+------------+
| primary_tagged_map_foreign_tag_id_55f9c032edecc | tag_id  | primary_tags   | id              | CASCADE    | CASCADE    |
+-------------------------------------------------+---------+----------------+-----------------+------------+------------+
```

As in case with BELONGS_TO_MPRHED you can observe morphed key in our pivot table. To better understand how such table is created we can check relation options which are very similar to MANY_TO_MANY:

Option            | Default                                 | Description
---               | ---                                     | ---
MORPHED_ALIASES   | []                                      | Association list between tables and roles, internal.
PIVOT_TABLE       | {name:singular}_map                     | Pivot table name will be generated based on singular relation name and _map postfix (**tagger_map**).
INNER_KEY         | {record:primaryKey}                     | Inner key of parent record will be used to fill "THOUGHT_INNER_KEY" in pivot table. (**tag.id**)
OUTER_KEY         | {outer:primaryKey}                      | We are going to use primary key of outer table to fill "THOUGHT_OUTER_KEY" in pivot table. This is technically "inner" key of outer record, we will name it "outer key" for simplicity. (**id**)
MORPH_KEY         | {name:singular}_type                    | Declares what specific record pivot record linking to (**tagged_type**).
THOUGHT_INNER_KEY | {record:role}_{definition:innerKey}     | Linking pivot table and parent record (**tag_id**).
THOUGHT_OUTER_KEY | {outer:role}_{definition:outerKey}      | Linking pivot table and outer records (**tagged_id**).
CONSTRAINT        | true                                    | Relations will set constraints in pivot table (foreign keys).
CONSTRAINT_ACTION | CASCADE                                 | [https://en.wikipedia.org/wiki/Foreign_key](https://en.wikipedia.org/wiki/Foreign_key)
CREATE_INDEXES    | true                                    | Relation is allowed to create indexes in pivot table
CREATE_PIVOT      | true                                    | Relation allowed to create pivot table.
PIVOT_COLUMNS     | []                                      | Additional set of columns to be added into pivot table, you can use same column definition type as you using for your records.
PIVOT_DEFAULTS    | []                                      | Set of default values to be used for pivot table columns.
WHERE_PIVOT       | []                                      | Where statement in a form of simplified array definition to be applied to pivot table data.

As in case with original MANY_TO_MANY we can link models together:

```php
$tag = new Tag();
$tag->name = 'First';
$tag->save();

$tag->tagged()->link(User::findOne());
$tag->tagged()->link(Post::findOne());
```

> Attention, link() method of ManyToMorphed relation can only accept one `Record` model.

To get access to relation data, we should specify what type of Records we want to receive (simply used pluralize model name, IDE must help you):

```php
$tag = Tag::findOne();


dump($tag->tagged->users);
dump($tag->tagged->posts);
```

> If you want to change names used to get access to data, check MORPHED_ALIASES option.

Technically, ManyToMorphed relation simply aggregates set of ManyToMany relations, meaning you can always get access to inner sub relation using such code:

```php
$tag = Tag::findOne();

$tag->tagged->users()->sync([1, 2, 3, 4]);
```

As before, since we declared INVERSE key, we can get access to our tags using "tags" property of method in models User and Post.

```php
$user = User::findByPK(1);

foreach ($user->tags as $tag) {
    dump($tag);
}
```