# Standalone Usage
Even if ODM component provides deep integration with MongoDB database, it's not required to use such database as data source. Due all ODM models can accept input data in constructor argument you are able to use them to mock or validate any type of data in your application.

## Standalone example
Let's try to create simple example of data structure we would like to handle:

```php
$data = [
    'name'     => 'value',
    'elements' => [
        ['name' => 'Element A'],
        ['name' => 'Element B'],
        ['name' => 'Element C']
    ]
];
```

To handle such data we will need two `DocumentEntity` models - Data and Element, let's create them:

```php
class Element extends DocumentEntity
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'name' => 'string'
    ];
}
```

```php
class Data extends DocumentEntity
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'name'     => 'string',
        'elements' => [Element::class]
    ];
}
```

Now we only need to feed our data into model constructor (do not forget to update ODM schema):

```php
$data = [
    'name'     => 'value',
    'elements' => [
        ['name' => 'Element A'],
        ['name' => 'Element B'],
        ['name' => 'Element C']
    ]
];

$model = new Data($data);
dump($model->name);

foreach ($model->elements as $element) {
    dump($element->name);
}

//Pack to array
dump($model->serializeData());
```

> Standalone ODM usage can be useful to validate complex data.

## Json Documents
One potential option how ODM component can be used in application without connection to MongoDB databases is using `JsonDocument`. Such class extends `DocumentEntity` and implements ORM `ActiveAccessorInterface`. As result this object can be used to represent structured json data in your ORM Records:

```php
class Data extends Record
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'id'   => 'primary',
        'data' => 'json'
    ];

    /**
     * @var array
     */
    protected $accessors = [
        'data' => JsonData::class
    ];
}
```

Json accessor:

```php
class JsonData extends JsonDocument
{
    /**
     * Entity schema.
     *
     * @var array
     */
    protected $schema = [
        'field' => 'string'
    ];

    /**
     * @var array
     */
    protected $validates = [
        'field' => [
            'notEmpty'
        ]
    ];
}
```

Now you can use such accessor in your code as composited field:

```php
$data = new Data();
$data->data->field = 'abc';
$data->save();
```

Generated SQL query for Postgres database:

```sql
INSERT INTO "abc_datas" ("data")
VALUES ('{"field":"abc"}') RETURNING "id"
```

As in case with composited documents you will not be able to save parent document/record if json data is invalid:

```php
$data = Data::findOne();
$data->data->field = '';
dump($data->getErrors());
```

> You can use json data in SQL queries whily working with PostgresSQL.
