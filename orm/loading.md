# Entity Cache and Eager Loading
One of them most important part of Spiral ORM is ability to use Record relations in conditional selections and pre-load such relations using eager loading.

Make sure you already read about [ORM Basics] (basics.md) and [ORM Relations] (relations.md).

## Join Relation table into selection
Based on previous guide sections we know that ORM can used samy syntax as DBAL Query Builders to select records from database:

```php
$selector = User::find()->where('user.name', '!=', '');
```

However in many cases we would need to filter our results based on values stored in related data, in this case we can use special method `with()` which can accept relation name or names to inlude outer tables into our statement:

```php
$selection->with('profile');
```

Such method will return us only user records which has associated Profile:

```php
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `profile`
    ON `profile`.`user_id` = `user`.`id` 
WHERE `user`.`name` != ''            
```

> Since there is multiple tables involved, ORM will generate all needed table aliases.

You are able to include any declared relations into your statement except polymorphic relations (you are still can use inversed polymorphic relations), for example we can try to find Users who has Profile and at least one published post:

```php
$selection->distinct()->with('profile')->with('publishedPosts');
```

Resulted SQL will include both "profiles" and "posts" tables into our query (including relation WHERE conditions):

```sql
SELECT  DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `profile`
    ON `profile`.`user_id` = `user`.`id`
INNER JOIN `primary_posts` AS `publishedPosts`
    ON `publishedPosts`.`author_id` = `user`.`id` AND `publishedPosts`.`published` = true 
WHERE `user`.`name` != ''
```

Since we using relation which can produce many results, we have to set DISTINCT flag, in opposite case we will not be able to use pagination and limiting methods.

#### Sub Relations
In addition of including Record realtion we are able to include relations which belongs to outer Record, for example let's try to find all Users who has at least one published post with at least one tag, we only have to separate our relations using dot symbol.

```php
$selection->with('publishedPosts.tags');
```

Resulted SQL:

```sql
SELECT  DISTINCT
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

> There is no real limitation on how many relations you can include and how deep you can go.

#### Aliases and Relation Conditions 
Based on provided examples you might notice that spiral ORM assign an alias to every joined table, such aliases are generated automatically based on relation and sub relation name. 

You can simply replace "." with underline to understand what type of alias will be assigned to joined table (publishedPosts.tags => "publishedPosts_tags"). In addition to that, every pivot table involved in MANY_TO_MANY relation will get additional postfix "_pivot".

Since all joined tables can be easily located by their alias, we can try to create more complex conditions for our selection. For example let's try to find every user who has role "admin":

```php
$selection->with('roles')->where('roles.name', 'admin');
```

Resulted SQL:

```sql
SELECT  DISTINCT
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

If you wish to change table alias to be used in query, you can provide additional argument into `with()` method - options, in our case we can declate joined table alias by using option "alias".

```php
$selection->with('roles', ['alias' => 'user_roles'])->where('user_roles.name', 'admin');
```

Many to many relations, in addition, provides you ability to specify alias for pivot table using "pivotAlias":

```php
$selection->with('roles', [
    'alias'      => 'user_roles',
    'pivotAlias' => 'roles_map' 
])->where('user_roles.name', 'admin');
```

SQL statement will include both of them:

```sql
SELECT  DISTINCT
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
    'roles' => [
        'alias'      => 'user_roles',
        'pivotAlias' => 'roles_map'
    ]
])->where('user_roles.name', 'admin');
```

In additon to that, you can specify relation conditions directly inside `with` method, without referencing to relation table alias:

```php
$selection->with([
    'roles' => [
        'where' => [
            '{@}.name' => 'admin'
        ]
    ]
]);
```

Such form can be easier to remember and it has it's own benefits and compatible with data pre-loading (see `load` method next).

> You can use "wherePivot" for pivot conditions as well.

## Sub Queries
In some rare scenarious, you might want to include sub query into your selection, for example if you want to find users who has more that 5 posts. Since ORM Selectors are compatible with DBAL SelectQuery builder you can use same tecnique:

```php
$selection = User::find()->distinct()->where('user.name', '!=', '');
$selection->with([
    'roles' => ['where' => ['{@}.name' => 'admin']]
]);

$selection->where(
    Post::find()->columns('COUNT(*)')->where('post.author_id', new SQLExpression('user.id')),
    '>',
    5
);
```

Such code will generate following query:

```sql
SELECT DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_role_user_map` AS `roles_pivot`
    ON `roles_pivot`.`user_id` = `user`.`id` AND `roles_pivot`.`status` = 'active'
INNER JOIN `primary_roles` AS `roles`
    ON `roles_pivot`.`role_id` = `roles`.`id` AND `roles`.`name` = 'admin' 
WHERE `user`.`name` != '' AND (SELECT
COUNT(*)
FROM `primary_posts` AS `post`
WHERE `post`.`author_id` = `user`.`id`) > 5
```

> Avoid using sub queries if you can.

## Pre-Loading Related Data
Spiral ORM provides you ability to solve N+1 problem using data preloading. In addition to that you are able to pre-load onle specific part of realted data using addinal where statemens.

Spiral provides only one method to be used - `load`. Such method can be combined with joined table conditions without causing collisions of confisuon. Even more, load method are fully compatibly by it's syntax with `with` method so you can pre-load many relations at once, specify where conditions and pre-load nested relations using dot separator.

To start, let's try to load every User and pre-load it's profile and roles:

```php
$selection = User::find()->where('user.name', '!=', '');
$selection->load('profile')->load('roles');
$selection->limit(10);
```

> You can freely use pagination and limiting while pre-loading data.

Resuled SQL will look like:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`, `d1`.`id` AS `c9`, `d1`.`biography` AS `c10`, `d1`.`user_id` AS `c11`
FROM `primary_users` AS `user`  
LEFT JOIN `primary_profiles` AS `d1`
    ON `d1`.`user_id` = `user`.`id` 
WHERE `user`.`name` != ''
LIMIT 10
```

```sql
SELECT
`d2`.`id` AS `c1`, `d2`.`name` AS `c2`, `d2`.`available` AS `c3`, `d2_pivot`.`role_id` AS `c4`, `d2_pivot`.`user_id` AS `c5`, `d2_pivot`.`time_assigned` AS `c6`,
`d2_pivot`.`status` AS `c7`
FROM `primary_roles` AS `d2`  
INNER JOIN `primary_role_user_map` AS `d2_pivot`
    ON `d2_pivot`.`role_id` = `d2`.`id` AND `d2_pivot`.`status` = 'active' 
WHERE `d2_pivot`.`user_id` IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

As you can see, spiral produced two SQL queries, profile data was joined to user data using LEFT JOIN and roles were loaded using addional query. 

> This time table aliases are not so predictable and you are forbidden to use them in normal where conditions.

As stated before, you are able to combine pre-loading and query filtering, let's try to select every user who has admin role and load ALL user roles:

```php
$selection->with('roles', ['where' => ['{@}.name' => 'admin']])
           ->load('roles');
```

And again, system will produce two SQL queries indendend of each other:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_role_user_map` AS `roles_pivot`
    ON `roles_pivot`.`user_id` = `user`.`id` AND `roles_pivot`.`status` = 'active'
INNER JOIN `primary_roles` AS `roles`
    ON `roles_pivot`.`role_id` = `roles`.`id` AND `roles`.`name` = 'admin' 
WHERE `user`.`name` != '' 
```

```sql
SELECT
`d1`.`id` AS `c1`, `d1`.`name` AS `c2`, `d1`.`available` AS `c3`, `d1_pivot`.`role_id` AS `c4`, `d1_pivot`.`user_id` AS `c5`, `d1_pivot`.`time_assigned` AS `c6`,
`d1_pivot`.`status` AS `c7`
FROM `primary_roles` AS `d1`  
INNER JOIN `primary_role_user_map` AS `d1_pivot`
    ON `d1_pivot`.`role_id` = `d1`.`id` AND `d1_pivot`.`status` = 'active' 
WHERE `d1_pivot`.`user_id` IN (1)
```

> Pivot table data are pre-loaded as well.

As in case with query filters we can specify conditions and pivot conditions to pre-load only specific portion of data. Let's try to load every user and all his public posts, we are going to ignore already exists relation "publishedPosts":

```php
$selection = User::find()->where('user.name', '!=', '');
$selection->load('posts', ['where' => ['{@}.published' => true]]);
```

As you can see, we have to use "{@}" alias as if we will use `where` our user selection will be constrained. Let's take a look on resulted SQLs:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`
WHERE `user`.`name` != ''
```

```sql
SELECT
`d1`.`id` AS `c1`, `d1`.`published` AS `c2`, `d1`.`title` AS `c3`, `d1`.`content` AS `c4`, `d1`.`author_id` AS `c5`
FROM `primary_posts` AS `d1`
WHERE `d1`.`published` = true AND `d1`.`author_id` IN (1, 2, 3, 4, ...)
```

## Entity Cache
