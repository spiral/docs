# Standalone Usage
Even if the ODM component integrates deeply with your MongoDB database, it's not required to use this database as your data source. Since all the ODM models can accept input data in your constructor argument, you are able to use them to mock up or validate any type of data in your application.

## Standalone example
Let's try to create a simple example of data structure that we would like to handle:

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

To handle this data, we will need two `DocumentEntity` models - Data and Element. Let's create them:

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

Now we only need to feed our data into the model constructor (do not forget to update the ODM schema):

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

> Standalone ODM usage can be useful when validating complex data, you only have to use create or setFields method to populate your entity.

## Json Documents
One potential option for how the ODM component can be used in your application without a connection to MongoDB databases is by using `JsonDocument`. This class extends the `DocumentEntity` and implements the ORM `ActiveAccessorInterface`. As a result, this object can be used to represent the structured json data in your ORM Records:

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

Now you can use this accessor in your code as a composited field:

```php
$data = new Data();
$data->data->field = 'abc';
$data->save();
```

The generated SQL query for a Postgres database:

```sql
INSERT INTO "abc_datas" ("data")
VALUES ('{"field":"abc"}') RETURNING "id"
```

As is the case with composited documents, you will not be able to save the parent document/record if your json data is invalid:

```php
$data = Data::findOne();
$data->data->field = '';
dump($data->getErrors());
```

> You can use json data in SQL queries while working with PostgresSQL.
