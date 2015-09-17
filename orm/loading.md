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

Resulted SQL will include both of them:

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


## Sub Queries



## Pre-Loading Related Data

































## Relation Conditions (with method)
In many cases we would need to select some database records using values from related tables. We already know how to buid conditions related to our primary record, to do so we only have to use singluar form of our entity:

```php
$selection = User::find();

foreach ($selection as $user) {
    dump($user->name);
}
```

Resulted SQL looks like:

```sql
SELECT
*
FROM `primary_users` AS `user`
WHERE `user`.`name` != ''
```

As you might remember based on previous [guide section] (relations.md) we defined HAS_ONE connection between User and Profile. We can use such connection to filter our results, to do so we must include relation table into our selection. This can be achieved using method `with()`.

```php
$selection = User::find()->where('user.name', '!=', '')->with('profile');

foreach ($selection as $user) {
    dump($user->name);
}
```

Resulted code will find only users who has non empty profiles, profiles table will be aliases using **relation name**:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `profile`
    ON `profile`.`user_id` = `user`.`id` 
WHERE `user`.`name` != ''
```

> In addition to that, ORM forced column aliases.

If you wish to change alias name for your selection, for example to use it where statement - you can provide an array of options into your method:

```php
$selection->with('profile', ['alias' => 'user_profile'])
          ->where('user_profile.biography', '!=', '');
```

Resulted SQL will include new alias:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_profiles` AS `user_profile`
    ON `user_profile`.`user_id` = `user`.`id` 
WHERE `user`.`name` != '' AND `user_profile`.`biography` != ''
```

Since `with()` condition works over INNER JOIN, it will behave as filter by itself, for example we can try to find all users with posts:

```php
$selection = User::find()->distinct()->where('user.name', '!=', '');
$selection->with('posts');
```

Please note, since we are using HAS_MANY relations we should specify distinct on our query, in opposite case we will not be able to use pagination. And again, resulted SQL is pretty simple:

```sql
SELECT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_posts` AS `posts`
    ON `posts`.`author_id` = `user`.`id` 
WHERE `user`.`name` != ''
```

As you can remember we defined another type of HAS_MANY relation - "publishedPosts":

```php
'publishedPosts'  => [
    self::HAS_MANY  => Post::class,
    Post::OUTER_KEY => 'author_id',
    Post::WHERE     => [
        '{@}.published' => true
    ]
],
```

Such relation decalres additional conditon on relation and still can be used for filtering.

```php
$selection = User::find()->distinct()->where('user.name', '!=', '');
$selection->with('publishedPosts');
```

This time SQL will include such condition:

```sql
SELECT  DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_posts` AS `publishedPosts`
    ON `publishedPosts`.`author_id` = `user`.`id` AND `publishedPosts`.`published` = true 
WHERE `user`.`name` != ''
```

As you might guess you are not limited in how many realations to be used for filtering, you can either specify every relation using separate `with()` call or provide their names in array form:

```php
$selection = User::find()->distinct()->where('user.name', '!=', '');
$selection->with(['roles', 'posts']);
```

Both relations will be included into resulted statement:

```sql
SELECT  DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_role_user_map` AS `roles_pivot`
    ON `roles_pivot`.`user_id` = `user`.`id` AND `roles_pivot`.`status` = 'active'
INNER JOIN `primary_roles` AS `roles`
    ON `roles_pivot`.`role_id` = `roles`.`id`
INNER JOIN `primary_posts` AS `posts`
    ON `posts`.`author_id` = `user`.`id` 
WHERE `user`.`name` != ''
```

> You are not able to use polymorphic relations for filtering your results. Hovewer you can use INVERSED relations to do so.

#### Nested relations
In many cases you might want to filter your results using nested relation, this is possible by providing relations using dot separator:

```php
```

```sql
SELECT  DISTINCT
`user`.`id` AS `c1`, `user`.`name` AS `c2`, `user`.`email` AS `c3`, `user`.`status` AS `c4`, `user`.`balance` AS `c5`, `user`.`time_registered` AS `c6`, `user`.`time_created` AS
`c7`, `user`.`time_updated` AS `c8`
FROM `primary_users` AS `user`  
INNER JOIN `primary_posts` AS `posts`
    ON `posts`.`author_id` = `user`.`id`
INNER JOIN `primary_tagged_map` AS `posts_tags_pivot`
    ON `posts_tags_pivot`.`tagged_id` = `posts`.`id` AND `posts_tags_pivot`.`tagged_type` = 'post'
INNER JOIN `primary_tags` AS `posts_tags`
    ON `posts_tags_pivot`.`tag_id` = `posts_tags`.`id` 
WHERE `user`.`name` != ''       
```

In this case joined table 

#### Where Statements
Since you connected your relations you are able to use their aliases (based on relation name)

#### Where Pivot Statements

## Pre-Loading Relations (load method)

## Entity Cache
