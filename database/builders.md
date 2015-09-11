# Query Builders
As many other DB layers spiral provide way to construct basic SQL queries using set of generation classes. Such classes support fluent syntax which simplifies their usage in everyday applications.

## Before we start
Before we will start learning about different query builders let's try to create a table in our database first, we are going to use controller action for that:

```php
public function index(Database $database)
{
    $table = $database->table('test');

    $schema = $table->schema();
    $schema->primary('id');
    $schema->datetime('time_created');
    $schema->enum('status', ['active', 'disabled'])->defaultValue('active');
    $schema->string('name', 64);
    $schema->string('email');
    $schema->double('balance');
    $schema->save();
}
```

Once such table created, we can remove schema declaration and leave only table variable (instance `Spiral\Database\Entities\Table` or Table abstraction).

## Insert Builder
To get an instance of InsertBuilder (responsible for insertions), we can execute following code:

```php
$insert = $database->insert('test');
```

As result we will get builder associated with our table. Now we can add some values to our builder to be inserted into related table:

```php
$insert = $database->insert('test');

$insert->values([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]);
```

To run InsertQuery we should only execute method `run()` which will return last inserted id as result:

```php
dump($insert->run());
```

> You can also use fluent syntax: `$database->insert('table')->values(...)->run()`.

### Batch Insert
In many cases you might want to insert multiple records at once, we can achieve such goal with insert builder by specifying set of columns and values separatelly:

```php
$insert->columns(['time_created', 'name', 'email', 'balance']);
for ($i = 0; $i < 20; $i++) {
    //We don't need to specify key names in this case
    $insert->values([
        new \DateTime(),
        StringHelper::random(10),
        StringHelper::random(10) . '@email.com',
        mt_rand(0, 1000) / 10
    ]);
}

$insert->run();
```

### Quick Inserts
You can simplify insertion process by talking directly to Table absraction you want to add data into, in this case our code may look like:

```php
$table = $database->table('test');

dump($table->insert([
    'time_created' => new \DateTime(),
    'name'         => 'Anton',
    'email'        => 'test@email.com',
    'balance'      => 800.90
]));
```

> Table class will automatically run query and return last inserted id. You can also check `batchInsert` method of Table abstraction.

## Select Query Builder
SelectQuery builder can be retrieved two very similar ways, you can either get it either from database or table abstractions, let's try to show both examples (result are identical):

```php
public function index(Database $database)
{
    $table = $database->table('test');

    $select = $database->select()->from('test');
    $selectB = $table->select();
}
```

In our examples we will choose second appropach as it's shorter.

### Configuring and Reading selection
By default SelectQuery selects every column (*) from it's related table. We can always change set of requested columns using `columns` method.

```php
$select->columns(['id', 'status', 'name']);
```

Now, before jumping to where/having and other statements, let's figure out how to read data from our selection. DBAL component provides convinient `QueryResult` class which can help us to iterate thought result. We can receive such iterator either by calling `run()` method or `getIterator()` method. 

```php
$iterator = $select->getIterator();

foreach($iterator as $row) {
    dump($row);
}
```

In addition to that you can simply create foreach using your builder as source:

```php
foreach($select as $row) {
    dump($row);
}
```

> You can add such code to your action to view data created by insertion.

### Where Statements

### Having

### Pagination

### Sorting

### Grouping and Distinct

### Joins

### Caching

### Aggreagations

## Update Query Builder

## Delete Query Builders

