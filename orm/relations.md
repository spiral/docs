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

## Has Many Relation

## Belongs To Relation

#### Belongs To Morphed

## Many To Many

#### Many To Many Morphed

## Other Relations