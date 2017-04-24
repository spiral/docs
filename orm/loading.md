# Eager Loading
The ORM provides you ability to solve N+1 problem using data pre-loading. Pre-loading can be based on relation options, custom sub query or sort direction.

> Read about how to [Query Models](/orm/query.md).

To make pre-loading you will need only one method - `load`. 

> Such method can be combined with joined table conditions without causing collisions, `load` method are fully compatibly by it's syntax with `with` method so you can pre-load many relations at once, specify where conditions and pre-load nested relations using dot separator.

To load user, it's profile (has one) and roles:
 
```php
$selection = $userRepo->find()->where('user.name', '!=', '');
$selection->load('profile')->load('roles';
$selection->limit(10);
```

SQL #1:

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

SQL #2:

```sql
SELECT
`d2`.`id` AS `c1`, `d2`.`name` AS `c2`, `d2`.`available` AS `c3`, `d2_pivot`.`role_id` AS `c4`, `d2_pivot`.`user_id` AS `c5`, `d2_pivot`.`time_assigned` AS `c6`,
`d2_pivot`.`status` AS `c7`
FROM `primary_roles` AS `d2`  
INNER JOIN `primary_role_user_map` AS `d2_pivot`
    ON `d2_pivot`.`role_id` = `d2`.`id` AND `d2_pivot`.`status` = 'active' 
WHERE `d2_pivot`.`user_id` IN (1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

Note, in two SQL queries, profile data was joined to user data using LEFT JOIN (as it was the most optimal way for HAS_ONE) and roles were loaded using additional query. 

> You can force relation to load using JOIN or external query using methods RelationLoader::INLOAD and RelationLoader::POSTLOAD. Note, that table aliases are unique.

You can combine conditional join and eager load without worry about name collisions:

```php
$selection->with('roles', ['where' => ['{@}.name' => 'admin']])
          ->load('roles');
```

SQL queries:

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

You can set conditions for you load data as well:

```php
$selection->where('user.name', '!=', '');
$selection->load('posts', ['where' => ['{@}.published' => true]]);
```

Use "{@}" alias in order to use joined table context:

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

After that, accessing relation data in User model will give us only published posts.

> Attention, it's much more reliable to define more specific relations for such purposes like "publishedPosts".

To order your selection include orderBy option:

```php
$selection->load('posts', ['where' => [
    '{@}.published' => true,
    'orderBy'       => ['{@}.id' => 'DESC']
]]);
```

## Load Alias
You can define table alias for included relation table using `alias` option.

## Caching
To cache your selection (including pre-loaded relations) pass `Psr\SimpleCache\CacheInterface` into `getIterator` method of your selector:

```php
$iterator = $selector->getIterator('cache-key', 8600, $cache);
```

> RecordSelector can fetch cache instance automatically from IoC if such binding exists.
