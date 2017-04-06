# Recursive Relations
Spiral ORM and loading mechanism does not limit you in how to build your hierarchies, though it is important to remember basic aspects of how to work with such models.

## Sample Model
We can define model which belongs to itself following way:

```php
class Recursive extends RecordEntity
{
    const SCHEMA = [
        'id'     => 'bigPrimary',
        'name'   => 'string',
        'parent' => [
            self::BELONGS_TO        => self::class,
            self::NULLABLE          => true,
            self::CREATE_CONSTRAINT => false
        ],
    ];
}
```

It is important to mark your relation as nullable.

> You are able to use table FK constrains, however some DBMS (SQLServer) forbid CASCADE behaviour of recursive data.

## Create with nested models
Nesed hierarchy works as any other relation:

```php
$recursive = new Recursive();
$recursive->parent = $recursive_a = new Recursive();
$recursive_a->parent = $recursive_b = new Recursive();
$recursive_b->parent = $recursive_c = new Recursive();

$recursive->save();
```

## Load Nested models
We can use `load` and `with` of our RecordSelector to load (join) any level of relation:

```php
//Only models with 3 level of parents
$recursive = $this->orm->selector(Recursive::class)->with('parent.parent.parent')->findOne();
```

## Errata with self-reference
At this moment, Spiral ORM is not capable to properly save self referenced models, i.e. such constuctions will fail:

```php
$recursive = new Recursive();
$recursive->parent = $recursive;

$recursive->save();
```

Though, it is possible to load such data from ORM by disabling entity map.

```php
$recursive = $this->orm->withMap(null)->selector(Recursive::class)
    ->with('parent.parent.parent')
    ->findOne();
```

> Attention, with disabled map you have a chance to get multiple models represent same row in database!
