# Query Models
One of them most important part of the ORM is the ability to use Record relations in conditional selections and pre-load such relation data using eager loading.

Make sure you already read about [ORM Relations](/old/orm/orm/relations.md).

> Examples are given using SourceTrait for simplicity.

## Join Relation table into selection
To filter our model while selection we can use it's singular name as table alias:

```php
$selection = User::find()->where('user.name', '!=', '');
```

In order to filter based on related data we have to join it's table first:

```php
$selection->with('profile');
```

> Note, `with` only work for data located in the same database.

Our query now will include INNER JOIN for our relation:

```php
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `profile`
    ON `profile`.`user_id` = `user`.`id` 
WHERE `user`.`name` != ''            
```

> Since there is multiple tables involved, ORM will generate all needed table and column aliases to prevent possible collision.

You are able to include any declared relations into your statement except polymorphic relations (you are still can use inversed polymorphic relations) and relations to other sources (i.e. ODM), to find Users who has Profile and at least one published post:

```php
$selection->distinct()->with('profile')->with('publishedPosts');
```

```sql
SELECT DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `profile`
    ON `profile`.`user_id` = `user`.`id`
INNER JOIN `primary_posts` AS `publishedPosts`
    ON `publishedPosts`.`author_id` = `user`.`id` AND `publishedPosts`.`published` = true 
WHERE `user`.`name` != ''
```

> Always use **DISTINCT** flag when including relation with multiple objects. 

#### Sub Relations
To include nested relation use dot notation.

```php
$selection->with('publishedPosts.tags');
```

```sql
SELECT DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_posts` AS `publishedPosts`
    ON `publishedPosts`.`author_id` = `user`.`id` AND `publishedPosts`.`published` = true
INNER JOIN `primary_tagged_map` AS `publishedPosts_tags_pivot`
    ON `publishedPosts_tags_pivot`.`tagged_id` = `publishedPosts`.`id` AND `publishedPosts_tags_pivot`.`tagged_type` = 'post'
INNER JOIN `primary_tags` AS `publishedPosts_tags`
    ON `publishedPosts_tags_pivot`.`tag_id` = `publishedPosts_tags`.`id` 
WHERE `user`.`name` != ''      
```

Note that included table name is generated based on relation chain.

#### Aliases and Relation Conditions 
Based on provided examples you might notice that spiral ORM assign an alias to every joined table, such aliases are generated automatically based on relation and sub relation name. 

You can simply replace "." with underline to understand what type of alias will be assigned to joined table (publishedPosts.tags => "publishedPosts_tags"). In addition to that, every pivot table involved in MANY_TO_MANY relation will get additional postfix "_pivot".

Since all joined tables can be easily located by their alias, we can try to create more complex conditions for our selection. To find every user who has role "admin":

```php
$selection->with('roles')->where('roles.name', 'admin');
```

```sql
SELECT DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_role_user_map` AS `roles_pivot`
    ON `roles_pivot`.`user_id` = `user`.`id` AND `roles_pivot`.`status` = 'active'
INNER JOIN `primary_roles` AS `roles`
    ON `roles_pivot`.`role_id` = `roles`.`id` 
WHERE `user`.`name` != '' AND `roles`.`name` = 'admin'
```

> You can set conditions on any of joined relation. Same way we can set conditions on pivot tables.

To change table alias to be used in a query you can provide additional argument into `with()` method - options, in our case we can declare joined table alias by using option "alias".

```php
$selection->with('roles', ['alias' => 'user_roles'])->where('user_roles.name', 'admin');
```

Many to many relations provide you ability to specify alias for pivot table using "pivotAlias":

```php
$selection->with('roles', [
    'alias'      => 'user_roles',
    'pivotAlias' => 'roles_map' 
])->where('user_roles.name', 'admin');
```

```sql
SELECT DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_role_user_map` AS `roles_map`
    ON `roles_map`.`user_id` = `user`.`id` AND `roles_map`.`status` = 'active'
INNER JOIN `primary_roles` AS `user_roles`
    ON `roles_map`.`role_id` = `user_roles`.`id` 
WHERE `user`.`name` != '' AND `user_roles`.`name` = 'admin'
```

#### Alternative with syntax
If you want to include multiple relations into your selection you can use array form passed into `with` method:

```php
$selection->with([
    'profile' => [],
    'roles'   => [
        'alias'      => 'user_roles',
        'pivotAlias' => 'roles_map'
    ]
])->where('user_roles.name', 'admin');
```

You can specify relation conditions directly inside `with` method, without referencing to relation table alias:

```php
$selection->with([
    'roles' => [
        'where' => [
            '{@}.name' => 'admin'
        ]
    ]
]);
```

Such form can be easier to remember and it has it's own benefits as it's compatible with data pre-loading (see `load` method next). Selection will automatically replace "{@}" with required table or pivot table alias.

> You can use "wherePivot" for pivot conditions as well.
