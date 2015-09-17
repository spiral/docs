# Entity Cache and Eager Loading
One of them most important part of Spiral ORM is ability to use Record relations in conditional selections and pre-load such relations using eager loading.

Make sure you already read about [ORM Basics] (basics.md) and [ORM Relations] (relations.md).

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



